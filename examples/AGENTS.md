# examples — 场景级完整宿主示例

五个自包含 main 程序，示范如何把 agentcore 装配成不同形态的 agent：custom-llm（自定义 provider/prompt/工具）、coding-agent（全功能编码 agent：PreToolUse hook、TodoWrite、compaction、交互审批、`-p` 单发）、security-audit（ModePlan 只读 + tracer 子代理）、planner-memory（Plan 审批 + Memory AutoInject）、coordinator（`RunParallel` 扇出 + LLM 协调者 + skill）。

## 约定（新增/修改示例必须维持）
- **宿主自持 prompt**：SDK 核心不内置任何 system prompt，场景 prompt 只活在这里——每个示例自带完整 systemPrompt（coding-agent 在 prompt.go），禁止把场景 prompt 挪进核心包。示例须自包含：不引 examples 间共享代码（`envOr` 等各写一份）、不带测试。
- Provider 一律经环境变量（AGENT_MODEL / AGENT_PROVIDER）或 flag 注入，默认值走 `envOr`；交互示例是 stdin REPL（Ctrl-D 退出），事件循环只打印 KindText/KindToolUse 等必要行。
- 各示例演示互不重叠的能力面，改某包行为后先找覆盖它的示例同步适配：permission→coding-agent（confirm / `-yes` 翻转 ModeBypass+nil CanUseTool）、memory→planner-memory（MemoryOptions，记忆存 `<wd>/.agent-memory`）、subagent→security-audit、coordinator→coordinator（`AppendSystemPrompt` 挂 `coord.SystemSegment()`，Tools 只放 `coord.AgentTool()`）。

## 运行
- 与主模块同 go.mod，编译随 `go build ./...` / `go vet ./...`（无独立 go.mod，也无 go.work）；运行 `go run ./examples/<dir>`，需真实 API key 与模型，不参与 `go test ./...`。
