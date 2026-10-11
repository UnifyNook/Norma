# AGENTS.md

本文件是仓库级统一入口。子目录中更近的 `AGENTS.md` 会补充或覆盖这里的规则；用户在对话中的明确要求优先级最高。整套体系的编写与维护约定见 `docs/AGENTS.md完全使用指南.md`（本地参考，`docs/` 不入库）。

## 项目概览

Norma 是 Go 语言通用 **Agent Harness SDK**（module `github.com/UnifyNook/Norma`）：把 LLM 变成自主 agent 的引擎——流式工具调用循环、调度与安全边界，provider 与场景无关（Anthropic Messages / OpenAI Chat Completions / OpenAI Responses 三种 wire format 归一化为同一消息模型）。上层应用（如 Artex）以 SDK 依赖形式消费本仓库。改动任何代码前先锚定三条设计立场：

- **宿主拥有 prompt**：SDK 不内置任何预设 system prompt，场景化 prompt 只存在于 `examples/`，不进核心包。
- **Fail-closed**：新工具默认不可并行、需审批；只读工具显式声明后才获得并行执行资格。
- **依赖极简**：直接依赖仅 pty / vt10x / x/net / yaml.v3，新增依赖须克制并说明理由。

## 常用命令

- 构建与静态检查：`go build ./...` && `go vet ./...`（CI 目前就这两步）
- 测试：`go test ./...`——注意 CI 暂未启用全量测试：`noa` 有 2 个属设计参数 TODO 的失败测试，`noaadapter` 的 chaos 测试带随机 seed 不稳定（缘由见 `.github/workflows/ci.yml` 注释，修好前提交需自行跑受影响包的测试）
- CLI 演示：`go build -o agentcore ./cmd/agentcore`，运行方式见 README「CLI demo」

## 全局要点

- 纯 Go 库仓库：无前端、无数据库；Go 版本以 go.mod 为准（当前 1.26，使用了 `iter`）。
- **本地联调**：上层应用通过 `go.work`（gitignore，不提交）把本仓库替换为本地路径；本仓库内不要提交 go.work。
- `docs/` 是本地设计文档（AGENTS.md 指南等），在 .gitignore 中，不入库；引用其中结论时以代码为准。
- **提交规范**：Conventional Commits（`feat:` / `fix:` / `ci:` …），描述用中文，scope 用包名（如 `fix(llm):`）。

## ⚠ 分层规则

**修改某个目录下的代码前，必须先阅读该目录下的 `AGENTS.md`（如有）。** 根文件只含全局规则与导航，子目录的 `AGENTS.md` 承载该包特有的契约、不变量与坑。

### 子规则索引

| 目录 | 职责 |
| --- | --- |
| [llm/AGENTS.md](llm/AGENTS.md) | 供应商中立消息模型 + `Provider` 接口；Anthropic Messages / OpenAI Chat Completions / Responses 三适配器归一化为同一流式事件 |
| [tool/AGENTS.md](tool/AGENTS.md) | `CoreTool` 契约、fail-closed `Build` 工厂、`Registry` 与核心工具集（Read/Write/Edit/Bash…、后台任务工具、deferred tools） |
| [permission/AGENTS.md](permission/AGENTS.md) | 安全边界：allow / deny / ask、六种 PermissionMode、deny 优先规则、四段 `Evaluate` 管线 |
| [harness/AGENTS.md](harness/AGENTS.md) | 主循环：`Query` 流式迭代器、只读并行/写串行调度、tool_use↔tool_result 恒配对、终止原因 |
| [compaction/AGENTS.md](compaction/AGENTS.md) | 上下文窗口管理：Snip / MicroCompact / Context Collapse / AutoCompact、熔断器、prompt-too-long 恢复 |
| [noa/AGENTS.md](noa/AGENTS.md) | 模型驱动上下文压缩引擎的纯核心：无 I/O、无模型调用、不依赖 Norma 其他包 |
| [noaadapter/AGENTS.md](noaadapter/AGENTS.md) | noa 引擎接入层：llm.Message 投影/回投、状态与归档持久化、Compress 工具 |
| [hook/AGENTS.md](hook/AGENTS.md) | 八个生命周期钩子：可阻断工具调用、改写输入、注入上下文、拦截 Stop |
| [subagent/AGENTS.md](subagent/AGENTS.md) | `Agent` 工具：委托隔离子 agent 复用同一循环，递归深度守卫 |
| [mcp/AGENTS.md](mcp/AGENTS.md) | MCP 集成：stdio JSON-RPC 客户端与进程内 server，工具暴露为 `mcp__<server>__<tool>` |
| [plan/AGENTS.md](plan/AGENTS.md) | 计划模式：只读探索 → `ExitPlanMode` 呈报审批 → 权限模式中途翻转 |
| [memory/AGENTS.md](memory/AGENTS.md) | 持久长期记忆：frontmatter markdown + `MEMORY.md` 索引、Recall/Record 工具 |
| [skill/AGENTS.md](skill/AGENTS.md) | `SKILL.md` 指令包：Skill 工具按需注入（抗 compaction）、可解锁声明的 MCP 工具 |
| [coordinator/AGENTS.md](coordinator/AGENTS.md) | 协调者模式：顶层 agent 分解工作并委托隔离 worker（LLM 驱动 / `RunParallel` 编程式） |
| [agentcore/AGENTS.md](agentcore/AGENTS.md) | 公共入口：有状态 `Session`（`Prompt` 流式 + `Resume`）聚合全部能力 |
| [transcript/AGENTS.md](transcript/AGENTS.md) | 追加式 JSONL 会话转录持久化（`Session.Resume` 的底层） |
| [cmd/agentcore/AGENTS.md](cmd/agentcore/AGENTS.md) | 可运行 REPL 演示：provider + 宿主 prompt + 核心工具集 + 交互式审批 |
| [examples/AGENTS.md](examples/AGENTS.md) | 各场景完整示例（custom-llm / coding-agent / security-audit / planner-memory / coordinator） |

## 特殊工具提示

本仓库在 `.mcp.json`、`.cursor/mcp.json`、`.codex/config.toml` 与 `.opencode/opencode.json` 中配置了 Serena 与 CodeGraph 两个 MCP，与原生工具按任务形状分工，不要无条件只用某一个：

- 跨文件架构、功能入口、调用链、依赖关系、改动影响面，且 CodeGraph 已初始化（`.codegraph/` 存在）时，优先用 CodeGraph 获取全局上下文。
- 已知具体符号后的定义查看、引用查找、精确修改、重命名，用 Serena/LSP。
- 精确文本搜索（错误信息、日志字符串、配置键）、整文件阅读、行级编辑、命令执行，用原生工具。
- 索引结果（CodeGraph 图数据、调用计数）与实际源码不一致时，以当前源码和可执行验证结果为准；逐点精确的引用清单以 Serena（LSP references）为准。
- Serena 返回的行号是 0-based，向用户报告时 +1。

## 行为反模式警示：拒绝纠错残留

严厉禁止以下“纠偏后残留历史痕迹与自我辩解”的行为模式：

> 用户让做一盘“番茄炒蛋”，Agent 擅自加了“东坡肉”；被指出后虽然去掉了，但提交/PR 时写着「番茄炒蛋（无东坡肉）」，并在注释中大篇幅解释为什么本道菜不需要加东坡肉。

**交付要求**：任何代码、注释、文档、提交信息及回复，都必须直接呈现对齐后的**干净终态设计**（Clean final-state design），严禁包含任何前序错误、自我辩解或“在此删掉某逻辑”的纠错痕迹。

## AI协助开发声明

凡使用了 AI 辅助生成或修改的代码，必须在 **commit 信息结尾**（以及对应 **PR 描述**中）按以下格式注明所使用的工具环境和模型：

```
assisted-by：{agent_name}：{model}
```

- `{agent_name}`：使用的 AI 编码工具/环境，如 `opencode`、`codex`、`cursor`
- `{model}`：实际使用的模型（含供应商，格式可参考 `供应商/模型`），如 `Zhipu/GLM-5.3[Max]`

示例：

```bash
git commit -m "feat(channel): support batch model pulling

assisted-by：opencode：Zhipu/GLM-5.3[Max]"
```

多个模型/工具参与时逐行列出。纯人工改动无需此声明，但需在 PR 描述中说明。
