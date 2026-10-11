# harness

Agent 主循环（stream → 调度执行工具 → 结果写回 → 重复）的唯一实现：`Query` 以 `iter.Seq2[Event, error]` 流式产出，最终事件恒为 `KindResult`（宿主提前 break 除外）。本包只做编排——消息模型与流事件归 `llm`，工具契约与输出截断归 `tool`，allow/deny 判定归 `permission`；compaction/transcript/hook 的实现各在其包，这里只依赖 `Compactor`/`Recorder`/`HookRunner` 窄接口。

## 必须维持的不变量

- **双 ctx 取消语义**：模型调用永远绑父 `ctx`（预算从不切流）；`toolCtx` 只派生给工具执行，携带 `MaxDuration` deadline。父 ctx 取消 = 真中断（流中/turn 边界 `ReasonAbortedStreaming`、工具期 `ReasonAbortedTools`，绝不进入收尾）；`toolCtx` 到期 = 打断在飞工具并就地进入 settlement，终因 `ReasonTimeout`。二者区分靠 `ctx.Err()` 与 `toolCtx.Err()`（timeout_settle_test.go 有一对回归测试锚定）。
- **每个 tool_use 必须配对 tool_result**：deny、schema 校验失败、unknown tool、hook 拦截、取消/超时中断，均由 `execOne` 与 `drainSynthetic`/`results()` 合成 error 结果补齐，任何路径都不允许漏配对。
- **turnCount 只计 tool-turn**（首个模型调用即 turn 1）；max_tokens 恢复是同 turn 重试，不推进计数、不触发 MaxTurns。`MaxTurns`/`MaxDuration` 只在 turn 边界检查。settlement 收尾跑完仍报原预算 reason，永远不是 `ReasonCompleted`。
- **历史非破坏**：`l.messages` 全量保留，模型只见 `MessagesForAPI` 投影或 `ContextView.View` 的投影——View 必须是纯函数、结果绝不写回（消息字节恒定是 content-hash 身份的前提）。`Recorder` 记录 raw 全保真历史（MicroCompact 临时清空不经过它）；`SystemReminder` 每次调用前临时插入、不入历史。
- **调度 = FIFO + 独占**：并行资格由 `IsConcurrencySafe(input)` 逐调用判定；safe 可并行，非 safe 独占并阻塞队列中其后所有工具；`drain()` 在首个未完成的非 safe 工具处停，保证结果按到达序 yield。工具在 tool_use 块流完时即开始执行，不等整条消息。

## 模式与坑

- 测试缝是 `QueryDeps.CallModel`/`CallModelSync`/`NewID`（默认取 `Provider.Stream`/`Complete`）：`scriptedModel` 按次回放预设 `llm.StreamEvent` 序列，整包 `go test ./harness/` 纯离线。循环改动须在流式与 `NonStreaming` 两条路径同时验证——`completeAndExecute` 与流式返回同一元组，loop 不感知差异。
- 每条 continuation 都有界：escalate 64k 至多一次、resume-recovery 至多 3 次、prompt 溢出仅 reactive compact 重试一次（再溢出即 `ReasonPromptTooLong`）、`maxLoopIterations` 兜底。新增继续路径必须自带终止条件。
- 所有工具输出在 `capOutput` 单点统一截断（`CaptureOnce` 幂等，不二次截断）；绕过该 choke point 的新执行路径可能撑爆上下文窗口。
