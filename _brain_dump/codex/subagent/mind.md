一共有三种sub agents

第一类：MultiAgent
属于是让用户和模型可以感知到的subagent，例如用户主动要求启动一个agent

第二类：Delegate 模式
内部的subagent，例如做review, compact,guardian等

第三类：batch agent
batch启动agent，从csv中读取任务，并行处理


# MultiAgent V2

  一、整体架构                                                                                                                                           
                                                                                                                                                         
  ┌─────────────────────────────────────────────────────────────┐ 
  │                   ThreadManagerState                         │
  │   thread_manager.rs:199                                      │
  │                                                              │
  │   threads: HashMap<ThreadId, Arc<CodexThread>>               │
  │   ┌──────────────┐ ┌──────────────┐ ┌──────────────┐        │
  │   │ Root Thread   │ │ SubAgent 1   │ │ SubAgent 2   │        │
  │   │ (CodexThread) │ │ (CodexThread) │ │ (CodexThread) │       │
  │   │  ┌─────────┐ │ │  ┌─────────┐ │ │  ┌─────────┐ │        │
  │   │  │  Codex  │ │ │  │  Codex  │ │ │  │  Codex  │ │        │
  │   │  │  :378   │ │ │  │  :378   │ │ │  │  :378   │ │        │
  │   │  └────┬────┘ │ │  └────┬────┘ │ │  └────┬────┘ │        │
  │   │       │      │ │       │      │ │       │      │        │
  │   │  Session     │ │  Session     │ │  Session     │        │
  │   │  :797        │ │  :797        │ │  :797        │        │
  │   │       │      │ │       │      │ │       │      │        │
  │   │  submission  │ │  submission  │ │  submission  │        │
  │   │  _loop :4245 │ │  _loop :4245 │ │  _loop :4245 │        │
  │   └──────────────┘ └──────────────┘ └──────────────┘        │
  │                         ▲     ▲                              │
  │                         │     │                              │
  │              ┌──────────┴─────┴──────────┐                  │
  │              │      AgentControl          │                  │
  │              │      control.rs:95         │                  │
  │              │                            │                  │
  │              │  manager: Weak<TMS>        │                  │
  │              │  state: Arc<AgentRegistry> │                  │
  │              │         registry.rs:23     │                  │
  │              └────────────────────────────┘                  │
  │              (同一 root session tree 共享一个实例)              │
  └─────────────────────────────────────────────────────────────┘

> AgentControl clone 传递给子 agent 时，只是 Arc 引用计数 +1——所有 parent/child/sibling agent 操作的是内存中同一个 AgentRegistry对象，这就是"共享"的含义。而每个独立的用户对话会创建自己独立的 AgentRegistry，互不影响。



  每个 agent（包括 root 和所有 subagent）结构完全相同：

  Codex (codex.rs:378)
    ├── tx_sub: Sender<Submission>     ← 接收 Op 的入口
    ├── rx_event: Receiver<Event>      ← 产出 Event 的出口
    ├── agent_status: watch::Receiver  ← 状态订阅
    └── session: Arc<Session>          ← 核心状态
          ├── conversation_id: ThreadId
          ├── services.agent_control: AgentControl  ← 所有同树 agent 共享
          └── submission_loop() 在 tokio::spawn 中独立运行

  二、数据传输方式

  整个系统有 三层数据通道：

  层 1：Op/Event 通道（每个 agent 内部）

    外部调用者                    Agent 内部
        │                           │
        │   Op (Submission)          │
        ├──────────────────────────► │ tx_sub → rx_sub
        │   async_channel::bounded   │     │
        │                            │     ▼
        │                            │ submission_loop() :4245
        │                            │     │
        │   Event                    │     │
        │ ◄────────────────────────  │ tx_event → rx_event
        │   async_channel::unbounded │

  Op 类型 (protocol.rs:212)：
  - UserInput { items, ... } — 用户输入
  - InterAgentCommunication { communication } — agent 间消息
  - Interrupt — 中断
  - Shutdown — 关闭

  层 2：AgentControl 跨 agent 通信

    Agent A (parent)                    Agent B (child)
        │                                    │
        │  agent_control.send_input(B, op)   │
        │ ──────────────────────────────────► │
        │  → TMS.send_op(B, op)              │
        │    → thread.submit(op)             │
        │      → tx_sub.send(Submission)     │
        │                                    │
        │  agent_control                     │
        │  .send_inter_agent_communication   │
        │ ──────────────────────────────────► │
        │  → TMS.send_op(B, Op::InterAgent)  │
        │                                    │
        │  agent_control.subscribe_status(B) │
        │ ◄──── watch::Receiver<AgentStatus> │
        │  (实时推送状态变化)                   │

> 同一棵 agent tree 内部，用绝对路径可以和任意 agent 通信，没有 parent/child 限制。只是相对路径只能往下找子 agent，不支持 .. 往上走。而跨 root tree 则完全隔离。

  层 3：InterAgentCommunication 结构化消息

  // protocol.rs:521
  pub struct InterAgentCommunication {
      pub author: AgentPath,       // 发送者路径，如 "/"
      pub recipient: AgentPath,    // 接收者路径，如 "/my-task"
      pub other_recipients: Vec<AgentPath>,
      pub content: String,         // 消息文本
      pub trigger_turn: bool,      // 是否立即触发 turn
  }

  接收端处理 (codex.rs:4662-4681)：
  收到 Op::InterAgentCommunication
    │
    ├── 尝试 inject_response_items() 到当前运行中的 turn
    │     └── 成功 → 追加到当前对话上下文
    │
    └── 失败（无活跃 turn）
          ├── queue_response_items_for_next_turn() 排队
          └── if trigger_turn == true
                └── 启动新 turn → spawn_task(RegularTask)

  三、完整调用链

  3.1 spawn_agent — 创建子 agent

  Model 调用 "spawn_agent" tool
  │
  ▼
  [1] spec.rs:1049  ── builder.register_handler("spawn_agent", SpawnAgentHandlerV2)
  │   ToolCallRuntime::handle_tool_call()  parallel.rs:56
  │     → router.dispatch_tool_call()      parallel.rs:112
  │       → Handler::handle()
  │
  ▼
  [2] multi_agents_v2/spawn.rs:25  ── Handler::handle(invocation)
  │   │
  │   ├── :33  parse_arguments → SpawnAgentArgs { task_name, message, agent_type, model, ... }
  │   ├── :41  parse_collab_input(message, items) → Op   (multi_agents_common.rs:162)
  │   ├── :45  next_thread_spawn_depth()                  (registry.rs:71)
  │   ├── :47  exceeds_thread_spawn_depth_limit()         (registry.rs:75)
  │   ├── :66  build_agent_spawn_config()                 (multi_agents_common.rs:203)
  │   ├── :67  apply_requested_spawn_agent_model_overrides() (multi_agents_common.rs:274)
  │   ├── :75  apply_role_to_config(&mut config, role)    (role.rs:38)
  │   ├── :78  apply_spawn_agent_runtime_overrides()      (multi_agents_common.rs:241)
  │   ├── :79  apply_spawn_agent_overrides()              (multi_agents_common.rs:267)
  │   ├── :81  thread_spawn_source()                      (multi_agents_common.rs:136)
  │   │         → SessionSource::SubAgent(SubAgentSource::ThreadSpawn { ... })
  │   │
  │   └── :88  构建 InterAgentCommunication 作为初始 Op（V2 特有）
  │             → Op::InterAgentCommunication { author: /, recipient: /task_name, trigger_turn: true }
  │
  ▼
  [3] :88  session.services.agent_control.spawn_agent_with_metadata(config, op, source, options)
  │                                                        (control.rs:131)
  │
  ▼
  [4] control.rs:142  ── spawn_agent_internal()
  │   │
  │   ├── :149  self.upgrade()? → Arc<ThreadManagerState>
  │   ├── :150  state.reserve_spawn_slot(max_threads)     (registry.rs:80)
  │   │           └── AtomicUsize CAS 限流，失败返回 AgentLimitReached
  │   ├── :151  inherited_shell_snapshot_for_source()      (control.rs:899)
  │   ├── :155  inherited_exec_policy_for_source()         (control.rs:915)
  │   ├── :165  prepare_thread_spawn()                     (control.rs:854)
  │   │           ├── register_root_thread(parent_id)      (registry.rs:121)
  │   │           ├── reserve_agent_path(AgentPath)         (registry.rs:242)
  │   │           ├── reserve_agent_nickname()              (registry.rs:202)
  │   │           │    └── 从 agent_names.txt 随机选取
  │   │           └── 返回 (SessionSource, AgentMetadata)
  │   │
  │   └── :243  state.spawn_new_thread_with_source(config, agent_control.clone(), source, ...)
  │                                                        (thread_manager.rs:717)
  │
  ▼
  [5] thread_manager.rs:727  ── spawn_thread_with_source()
  │   │
  │   └── :853  Codex::spawn(CodexSpawnArgs { config, session_source, agent_control, ... })
  │                                                        (codex.rs:430)
  │
  ▼
  [6] codex.rs:454  ── Codex::spawn_internal()
  │   │
  │   ├── :475  创建 async_channel (tx_sub ↔ rx_sub)      ← Op 通道
  │   ├── :476  创建 async_channel (tx_event ↔ rx_event)   ← Event 通道
  │   ├── :491  depth >= max_depth → 禁用 Collab feature
  │   ├──       构建 Session { conversation_id, services { agent_control, ... }, ... }
  │   │
  │   └── :667  tokio::spawn(submission_loop(session, config, rx_sub))
  │              └── :4245  submission_loop()  ← 子 agent 的独立 agent loop 启动!
  │
  ▼ 返回到 control.rs:142
  │
  [7] control.rs:259  ── reservation.commit(agent_metadata)  → 注册到 AgentRegistry
  │   control.rs:265  ── state.notify_thread_created(thread_id) → broadcast
  │   control.rs:274  ── self.send_input(thread_id, initial_operation)
  │                      → TMS.send_op(thread_id, op)         (thread_manager.rs:672)
  │                        → thread.submit(op)                 → tx_sub.send(Submission)
  │                      ← 初始 prompt 送入子 agent 的 submission_loop
  │
  │   control.rs:281  ── maybe_start_completion_watcher()      (control.rs:778)
  │                      └── tokio::spawn 后台 watcher:
  │                          监听 subscribe_status(child_id)
  │                          当 child 到达 final status →
  │                          send_inter_agent_communication(parent_id, notification)
  │                                                            (control.rs:838-841)
  │
  └── 返回 LiveAgent { thread_id, metadata, status }
        → spawn.rs 返回 SpawnAgentResult { task_name, nickname }
          → 作为 tool output 返回给 model

  3.2 Agent Loop — 子 agent 的运行循环

  [8] codex.rs:4245  ── submission_loop(session, config, rx_sub)
  │   │
  │   └── :4247  while let Ok(sub) = rx_sub.recv().await  ← 循环等待 Op
  │         │
  │         ├── :4329  Op::UserInput / Op::UserTurn
  │         │     └── handlers::user_input_or_turn()       (codex.rs:4572)
  │         │           ├── :4643  sess.spawn_task(turn_context, items, RegularTask)
  │         │           │                                   (tasks/mod.rs:164)
  │         │           │
  │         │           └── :209  tokio::spawn(task.run())  (tasks/mod.rs:209)
  │         │                 │
  │         │                 ▼
  │         │           [9] tasks/regular.rs:38  ── RegularTask::run()
  │         │                 │
  │         │                 ├── :49  emit TurnStarted event
  │         │                 └── :68  loop {
  │         │                       run_turn(sess, ctx, input, ...)
  │         │                            │      (codex.rs:5548)
  │         │                            │
  │         │                            ├── 构建 system prompt, 注入 context
  │         │                            ├── 调用 OpenAI Responses API (streaming)
  │         │                            ├── 处理 response stream:
  │         │                            │
  │         │                            │   :7254  handle_output_item_done()
  │         │                            │          (stream_events_utils.rs:197)
  │         │                            │     │
  │         │                            │     ├── :205  ToolRouter::build_tool_call()
  │         │                            │     │   如果 model 输出了 tool call:
  │         │                            │     │
  │         │                            │     └── :220  tool_runtime.handle_tool_call(call)
  │         │                            │               (parallel.rs:56)
  │         │                            │          └── :112  router.dispatch(session, turn, call)
  │         │                            │                    → 匹配到对应 handler
  │         │                            │                    (如 spawn_agent / send_message / ...)
  │         │                            │
  │         │                            └── tool 输出作为 response_input 送回 model
  │         │                                → model 继续推理或输出最终文本
  │         │                       }
  │         │
  │         ├── :4333  Op::InterAgentCommunication { communication }
  │         │     └── handlers::inter_agent_communication()  (codex.rs:4662)
  │         │           ├── :4668  inject_response_items()    注入到当前 turn
  │         │           ├── :4672  或 queue_for_next_turn()   排队
  │         │           └── :4674  if trigger_turn → spawn_task(RegularTask)
  │         │
  │         ├── :4252  Op::Interrupt → 中断当前 turn
  │         └── :4495  Op::Shutdown → break 退出循环

  3.3 assign_task — 向子 agent 发送任务（触发 turn）

  Model 调用 "assign_task" tool
  │
  ▼
  [10] multi_agents_v2/assign_task.rs:20
  │    └── handle_message_tool(invocation, MessageDeliveryMode::TriggerTurn)
  │
  ▼
  [11] multi_agents_v2/message_tool.rs:95  ── handle_message_tool()
  │   │
  │   ├── :107  parse MessageToolArgs { target, items, interrupt }
  │   ├── :108  resolve_agent_target(session, turn, target)    (agent_resolver.rs:8)
  │   │           ├── :14  尝试解析为 ThreadId (UUID)
  │   │           └── :18  agent_control.resolve_agent_reference(current_id, source, target)
  │   │                                                         (control.rs:636)
  │   │                     └── :648  state.agent_id_for_path(agent_path)
  │   │                                                         (registry.rs:136)
  │   │                               在 agent_tree HashMap 中查找 path → ThreadId
  │   │
  │   ├── :138  构建 InterAgentCommunication {
  │   │           author: parent 的 AgentPath (如 "/"),
  │   │           recipient: child 的 AgentPath (如 "/my-task"),
  │   │           content: 消息文本,
  │   │           trigger_turn: true   ← assign_task 的关键区别
  │   │         }
  │   │
  │   └── :147  agent_control.send_inter_agent_communication(thread_id, communication)
  │                                                           (control.rs:507)
  │               └── :518  TMS.send_op(agent_id, Op::InterAgentCommunication { communication })
  │                           (thread_manager.rs:672)
  │                   └── thread.submit(op) → tx_sub.send(Submission)
  │
  ▼
  子 agent 的 submission_loop 收到 Op::InterAgentCommunication
    → codex.rs:4333 → handlers::inter_agent_communication()  (codex.rs:4662)
      → trigger_turn=true → spawn_task(RegularTask) → 开始新的推理 turn

  3.4 send_message — 向子 agent 发送消息（不触发 turn）

  与 assign_task 完全相同的路径，唯一区别：

  [12] multi_agents_v2/send_message.rs:20
       └── handle_message_tool(invocation, MessageDeliveryMode::QueueOnly)

  message_tool.rs:28  ── QueueOnly → trigger_turn: false
    → 消息排队到 idle_pending_input
    → 不启动新 turn，等下次有 Op 时一起处理

  3.5 wait — 等待子 agent 完成

  Model 调用 "wait" tool
  │
  ▼
  [13] multi_agents_v2/wait.rs:26  ── Handler::handle()
  │   │
  │   ├── :36  resolve_agent_targets() → Vec<ThreadId>
  │   │          (每个 target 通过 agent_resolver.rs 解析 path → ThreadId)
  │   │
  │   ├── :52  timeout_ms.clamp(MIN=10s, MAX=3600s)
  │   │
  │   ├── :77  对每个 target: agent_control.subscribe_status(id)
  │   │                                                (control.rs:658)
  │   │          → thread.subscribe_status()
  │   │            → watch::Receiver<AgentStatus>
  │   │
  │   ├── :111 如果已有 final status → 立即返回
  │   │
  │   └── :113 否则：
  │         ├── 为每个 target 创建 wait_for_final_status() future  (:210)
  │         │     └── loop { status_rx.changed().await; if is_final(status) → return }
  │         ├── FuturesUnordered 并发等待
  │         └── timeout_at(deadline) 超时控制
  │
  └── 返回 WaitAgentResult { message, timed_out }

  3.6 子 agent 完成 → 自动通知 parent

  [14] control.rs:778  ── maybe_start_completion_watcher()
  │
  │   tokio::spawn 后台任务:
  │   ├── :793  subscribe_status(child_thread_id)
  │   ├── :796  loop: while !is_final(status) { status_rx.changed().await }
  │   │
  │   └── 当 child 到达 final status:
  │         ├── :815  format_subagent_notification_message()  (session_prefix.rs:8)
  │         │           → JSON: { "agent_path": "/my-task", "status": "Completed(...)" }
  │         │
  │         └── :832  构建 InterAgentCommunication {
  │                     author: child 的 AgentPath,
  │                     recipient: parent 的 AgentPath,
  │                     trigger_turn: false   ← 不强制触发 parent 的 turn
  │                   }
  │               :839  agent_control.send_inter_agent_communication(parent_id, communication)
  │                       → parent 的 submission_loop 收到通知
  │                       → 注入到 parent 的对话上下文

  3.7 close_agent — 关闭子 agent

  Model 调用 "close_agent" tool
  │
  ▼
  [15] multi_agents_v2/close_agent.rs:17  ── Handler::handle()
  │   │
  │   ├── :27  resolve_agent_target() → ThreadId
  │   ├── :46  agent_control.get_status(agent_id)
  │   └── :59  agent_control.close_agent(agent_id)        (control.rs:571)
  │              │
  │              ├── :576  state_db.set_thread_spawn_edge_status(Closed)  持久化
  │              └── :581  shutdown_agent_tree(agent_id)    (control.rs:585)
  │                          ├── live_thread_spawn_descendants() 找到所有后代
  │                          ├── shutdown_live_agent(agent_id)   (control.rs:551)
  │                          │     ├── ensure_rollout_materialized()
  │                          │     ├── flush_rollout()
  │                          │     ├── TMS.send_op(id, Op::Shutdown)
  │                          │     ├── TMS.remove_thread(id)
  │                          │     └── registry.release_spawned_thread(id)
  │                          └── 对每个后代重复 shutdown_live_agent()

  3.8 list_agents — 列出所有活跃 agent

  Model 调用 "list_agents" tool
  │
  ▼
  [16] multi_agents_v2/list_agents.rs:4  ── Handler::handle()
  │
  └── agent_control.list_agents(session_source, path_prefix)
                                                    (control.rs:699)
        ├── registry.live_agents()                   (registry.rs:155)
        │     └── 过滤 agent_tree 中有 agent_id 且非 root 的
        ├── 对每个 agent: state.get_thread(id) → thread.agent_status()
        └── 返回 Vec<ListedAgent { agent_name, agent_status, last_task_message }>

  四、关键数据结构索引

  ┌─────────────────────────────┬───────────────────────┬──────────────────────────────────────────┐
  │           结构体            │        文件:行        │                   用途                   │
  ├─────────────────────────────┼───────────────────────┼──────────────────────────────────────────┤
  │ Codex                       │ codex.rs:378          │ 每个 agent 的高层接口，持有 channel pair │
  ├─────────────────────────────┼───────────────────────┼──────────────────────────────────────────┤
  │ Session                     │ codex.rs:797          │ agent 运行状态，持有 services            │
  ├─────────────────────────────┼───────────────────────┼──────────────────────────────────────────┤
  │ TurnContext                 │ codex.rs:833          │ 单次 turn 的上下文                       │
  ├─────────────────────────────┼───────────────────────┼──────────────────────────────────────────┤
  │ AgentControl                │ control.rs:95         │ 跨 agent 控制平面                        │
  ├─────────────────────────────┼───────────────────────┼──────────────────────────────────────────┤
  │ AgentRegistry               │ registry.rs:23        │ agent 注册表，限流                       │
  ├─────────────────────────────┼───────────────────────┼──────────────────────────────────────────┤
  │ AgentMetadata               │ registry.rs:36        │ 单个 agent 的元数据                      │
  ├─────────────────────────────┼───────────────────────┼──────────────────────────────────────────┤
  │ SpawnReservation            │ registry.rs:294       │ spawn 预留槽位（RAII 释放）              │
  ├─────────────────────────────┼───────────────────────┼──────────────────────────────────────────┤
  │ ThreadManagerState          │ thread_manager.rs:199 │ 全局线程管理                             │
  ├─────────────────────────────┼───────────────────────┼──────────────────────────────────────────┤
  │ InterAgentCommunication     │ protocol.rs:521       │ V2 结构化消息                            │
  ├─────────────────────────────┼───────────────────────┼──────────────────────────────────────────┤
  │ AgentStatus                 │ protocol.rs:1604      │ agent 状态枚举                           │
  ├─────────────────────────────┼───────────────────────┼──────────────────────────────────────────┤
  │ SessionSource::SubAgent     │ protocol.rs:2372      │ 标识 subagent 来源                       │
  ├─────────────────────────────┼───────────────────────┼──────────────────────────────────────────┤
  │ SubAgentSource::ThreadSpawn │ protocol.rs:2383      │ spawn 来源详情                           │
  ├─────────────────────────────┼───────────────────────┼──────────────────────────────────────────┤
  │ AgentPath                   │ protocol crate        │ 层次路径如 /root/my-task                 │
  ├─────────────────────────────┼───────────────────────┼──────────────────────────────────────────┤
  │ Op                          │ protocol.rs:212       │ 所有操作类型枚举   

 
  ## 结构
  ThreadManager（全局唯一）
    │
    │  持有
    ▼
ThreadManagerState（Arc 共享）
    │
    ├── threads: HashMap<ThreadId, Arc<CodexThread>>  ← 所有 Agent 线程
    ├── auth_manager: Arc<AuthManager>                ← 全局共享
    ├── models_manager: Arc<ModelsManager>            ← 全局共享
    ├── environment_manager                            ← 全局共享
    ├── skills_manager                                 ← 全局共享
    └── plugins_manager                                ← 全局共享

每个 CodexThread
    │
    ├── Codex（:378）
    │     ├── tx_sub   ← 提交通道
    │     ├── rx_event ← 事件通道
    │     └── Session（:797）
    │           │
    │           ├── SessionServices
    │           │     ├── agent_control ──→ AgentControl
    │           │     │                        │
    │           │     │                        ├── manager: Weak<ThreadManagerState>
    │           │     │                        │   ↑ 弱引用回 ThreadManagerState
    │           │     │                        │
    │           │     │                        └── state: Arc<AgentRegistry> （应该还有agent metadata）
    │           │     │                             ↑ 同一棵树共享
    │           │     │
    │           │     ├── exec_policy       ← 可从父 Agent 继承
    │           │     ├── model_client      ← 各自独立
    │           │     └── mcp_manager       ← 各自独立
    │           │
    │           ├── active_turn             ← 各自独立
    │           ├── conversation history    ← 各自独立（fork 时复制）
    │           └── session_source          ← 各自独立（标识身份）
    │
    └── submission_loop（后台运行）

  ## 进程模型
  他很想进程模型
  OS 进程模型	Codex Agent 模型
操作系统	ThreadManagerState
进程	CodexThread（Agent）
进程 ID (PID)	ThreadId (UUID)
进程表	threads: HashMap<ThreadId, CodexThread>
fork()	fork_thread_with_source()
exec()	send_input(initial_operation)
进程间通信 (IPC)	InterAgentCommunication
进程树 (parent/child)	AgentPath 树（/root/write-tests/lint）
kill()	close_agent
waitpid()	wait_agent
环境变量继承	inherited_shell_snapshot
资源限制 (ulimit)	agent_max_threads、agent_max_depth
连 fork 的行为都很像：


OS:    fork() → 复制父进程的内存 → 子进程继续执行
Codex: fork_thread_with_source() → 复制父 Agent 的对话历史 → 子 Agent 继续工作
一句话：Agent 模型就是一个用户态的"轻量级进程系统"——有进程表、进程树、fork、IPC、wait、kill，只是跑在 tokio 异步运行时上而不是 OS 内核上。

## 监听结束
实际上委派了一个独立的后台任务（自己clone了一个agent control）去监听子agent的状态，等执行完毕后通知父agent

  ## Qustions
  1. 有预设的sub agents么
  2. subagent的模型是在spawn agent tool中提供的available models，LLM自己选择的
  3. get_rollout_history 看看里面存储的到底是什么，理论上应该是 conversataion history
  4. 为什么是父agent去监听，而不是让sub agent主动发送它结束了呢？

像 OS 里，子进程退出后是内核通知父进程（SIGCHLD），而不是子进程自己发信号——因为子进程可能是异常退出的，没机会发。
一句话：发的是子 Agent 的路径和状态（JSON 格式），由外部 watcher 发而不是子 Agent 自己发，是因为更可靠——不管子 Agent 怎么结束的都能通知到。

  5. sub agent的消息只会在父agent spawn后的监听的时候可以写入last ai msg。如果是兄弟agent发送等待任务，它实际上是接受不到消息的
  