# hook

八个生命周期钩子的 Go 回调注册表（`Registry`），直接实现 `harness.HookRunner`，接入 Session 无需胶水。回调在调用方 goroutine 上按注册顺序同步执行。

## 回调契约
- PreToolUse：第一个 `Decision:"block"` 立即短路，返回其 `Message` 并丢弃此前任何 `UpdatedInput`；未被阻断时改写输入取**最后一个**非空 `UpdatedInput`（后面的 hook 覆盖前面的）。
- Stop 的两种语义不可混用：`PreventContinuation` 硬终止循环（stop_hook_prevented），任一 hook 设置即胜出并短路其余结果；`Decision:"block"` 强制循环继续，把 `AdditionalContext`（为空则退回 `Message`）作为 blockingErrors 注入对话。
- PostToolUse 在 `isErr` 时会以同一 Event 再触发一次 PostToolUseFailure。
- UserPromptSubmit 返回各 hook 的 `AdditionalContext` 以 `\n` 拼接注入；SessionStart/SessionEnd/SubagentStart 是 fire-and-forget。

## 非显而易见
- `harness.HookRunner` 接口只含 PreToolUse/PostToolUse/Stop 三个方法；其余五个经 `Fire*` 方法触发——UserPromptSubmit 等由 agentcore 调，SubagentStart 由 subagent 经匿名接口断言调用（自定义 HookRunner 不实现该方法只是静默收不到事件，不会报错）。
- `run` 会强制覆写 `ev.Type`，`On` 注册的 Func 只收到对应类型的事件。
