# coordinator

协调者模式：顶层 agent 分解工作并委托隔离 worker 执行，自己不直接实现。完全构建在 subagent 之上——`Config.subagentConfig` 把本包配置包装成仅含单个 `"worker"` 定义的 `subagent.Config`。

## 两种委托形态（勿混淆）
- LLM 驱动：宿主把 `AgentTool()`（工具名即 "Agent"）加进协调者会话工具，`SystemSegment()` 追加到 system prompt；segment 会列出 worker 的工具名，并指示模型只做分解与综合。不要再给协调者会话加其他会改状态的工具——segment 假定它不亲自动手。
- 编程式：`RunParallel` 把 tasks 经信号量并发扇出给 worker，结果按任务序号写入切片，**输出顺序与入参一致**；单个 worker 失败记在 `Result.Err`，不影响其余任务。

## 默认值与非显而易见
- worker 默认值：system prompt 用 `DefaultWorkerPrompt`，permission mode 用 `ModeAcceptEdits`，`MaxParallel` 默认 4；信号量在派发循环内获取，天然形成背压（超限任务排队而非全部起 goroutine）。
- `MaxDepth`/`CanUseTool`/`Hooks`/`MaxTurns` 原样透传给 subagent；`RunParallel` 用未加深度的 ctx，所有并行 worker 起始深度相同，subagent 的深度守卫只约束 worker 内部再嵌套。
