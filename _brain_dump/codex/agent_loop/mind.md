
codex-rs/core/src/codex.rs
codex-rs/core/src/tasks/mod.rs
codex-rs/core/src/tasks/regular.rs run
codex-rs/core/src/codex.rs run_turn

1. 记得需要看一下有active turn的时候怎么处理
2. 子agent执行的结果可能在agent loop 完毕后返回，此时会把子agent的消息缓存到queued response items，但是也会立即执行。
需要了解清楚子agent和父agent的交互方式
3. react loop还不一样，有regular, compact还有ghotst_snapshot, review, undo, user shell
4. regular.rs中的server reasoning included为false，改成true会咋地
// 计算总 token 用量时
state.get_total_token_usage(state.server_reasoning_included())
server_reasoning_included	效果
true	token 统计包含 reasoning（思考）token
false	token 统计不包含 reasoning token
为什么初始设为 false？
因为任务刚开始时还不知道服务器返回的响应里有没有 reasoning token。等模型实际返回响应后，会根据实际情况更新（7333行）：



// 收到模型响应时，根据服务器实际返回来设置
ResponseEvent::ServerReasoningIncluded(included) => {
    sess.set_server_reasoning_included(included).await;
}
一句话：先设为 false（不含推理 token），等服务器告诉我们"响应里有推理 token"时再改为 true，确保 token 用量统计准确。

5. 需要检查一下skills的dependency到底是从哪里读取的。
maybe_prompt_and_install_mcp_dependencies(
        sess.as_ref(),
        turn_context.as_ref(),
        &cancellation_token,
        &mentioned_skills,
    )
    .await;

6. 查看plugins到底是干啥的