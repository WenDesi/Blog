
# 调用顺序

 src/entrypoints/cli.tsx L33
  - 快速路就检查，例如-v

 src/main.tsx main L584
   - 处理远程连接
   - 处理deeplink
   - 记录一些初始信息，例如是通过什么终端启动的

 src/main.tsx run L884
   - 初始化命令行参数解释器
   - 增加内部preAction钩子，用来注册日志输出，pluginDir等配置
   - 注册主命令的所有CLI选项
   - 链接MCP
   - 处理远程模式配置
   - 加载内置skills和tools
   - 确定会话名、确定模型、拿到命令列表和 agent 定义
   - 启动session start hook
   - 如果是headless模式，直接执行

src/replLauncher.tsx L12
   - 加载App组件，REPL组件

src/interactiveHelpers.tsx L98
   - 加载user context，取skill等
   - 渲染Ink，启动循环，等待退出

 src/screens/REPL.tsx L572
   - 我理解就是定义整个UI的ReAct Component，其中重点看onsubmit
   - OnSubmit主要做了
       = 滚动、恢复 proactive
       = 各种早期返回（即时命令、远程、空闲、推测）
       = 存历史、清输入框
       = 调 handlePromptSubmit（真正干活的）
       = 恢复暂存输入

 src\utils\handlePromptSubmit.ts L120
   - 做Prompt的展开和过滤
   - 处理队列中的消息等等

 src\utils\handlePromptSubmit.ts L396
   - 拼接消息

 src/screens/REPL.tsx L2856
 src/screens/REPL.tsx L2661
 
 src/query.ts L219
 src/query.ts L241

# Questions
1. KAIROS 这个feature flag到底是干啥的，他和assistant mode有关系
2. 说是有一个team agent的概念，叫做agent swarms
3. 有5种模式，本地cli， ssh remote, sdk, remote session/bridget, assistant. 需要理解它们之间的差别。（是不是还有一种direct connect和CCR）
 （assistant有一个会定期收到<tick>提示，让agent自主找活干）
4. Coordinator 模式 是干啥的
5. 还有一个advisor模式
6. 还有proactive模式