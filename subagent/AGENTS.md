# subagent

把 `Agent` CoreTool 暴露给父模型：委托隔离子 agent 复用同一 harness 循环（自带 system prompt、工具子集、permission mode），只把最终文本交回父级。coordinator 的两种委托形态都构建在本包之上。

## 不变量与隔离边界
- 工具从 `ParentTools` 按 `Definition.Tools` 过滤；**空列表 = 继承全部父工具**（原样复用父 registry，不重建）。
- 中间输出不回流：父级只拿到 `KindResult` 的 Terminal 文本，大体量中间产物留在子上下文里。
- 递归深度守卫：深度经 ctx value 传递、`run` 内 +1，`depth >= MaxDepth`（默认 3）返回错误工具结果；Agent 工具自身 `Permissions` 恒 `Allowed()`——真实门禁发生在子 agent 内部的工具调用上（子会话继承 `CanUseTool` 与 `Hooks`）。
- 未知 agent_type、深度超限、子运行失败都归一为 `IsError` 工具结果（"Error: ..."），不向循环抛 Go error。

## 非显而易见
- `Run` 是编程式入口（coordinator `RunParallel` 直接调用），与 Agent 工具共享同一 `run`；深度以调用方 ctx 为准，并行 fan-out 的各 worker 起始深度相同，守卫只约束嵌套。
- `Transcript` 非 nil 时每次运行写独立 sidechain：父 session id 取自 ctx（缺失则新生成），子 agent 用量记在 sidechain，不计入父会话总量。
