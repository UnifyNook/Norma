# AGENTS.md — tool 包

为 agent 提供双手的包：CoreTool 契约、fail-closed 的 `Build` 工厂、保序的 `Registry`、最小 JSON-Schema 校验器与内置工具集。边界：本包只产出 `permission.Decision` 建议（`CheckPermissions`），判定管线在 permission 包；schema 前置校验、并行调度、输出兜底截断都由 harness 消费本包接口完成（`CaptureOnce` 由 harness 的 post-tool net 调用，靠标记幂等）。

## 契约与 fail-closed 默认

- `Spec` 任一行为字段为 nil，`Build` 填 fail-closed 默认：not read-only、not concurrency-safe、`CheckPermissions` 返回 AskUser。只读工具必须显式 `ReadOnly: always` + `Concurrent: always` + `Permissions: allowReadOnly` 才获得并行资格——harness 只并发执行 concurrency-safe 的工具。
- 工具运行期故障返回 `Errorf` Result（IsError）+ nil Go error，把错误交给模型而非中断循环；仅输入 JSON 本身解不开才返回 Go error。
- `RawInput` 让 Run 绕过 harness 的 schema 校验，供自行修复畸形参数的工具；`AcceptsRawInput()` 刻意不放进 CoreTool 接口（公开接口，加方法会破坏全部宿主实现），harness 用类型断言发现。
- `ValidateInput` 只覆盖最小子集（object/required/基本类型/array items）：未知 property 容忍放行，enum、minimum/maximum 等不校验。
- 新增核心工具：写 Spec（`Prompt` 会被 `Registry.Schemas` 并进 description）→ `Build` → 只读则显式 opt-in → 加入 `DefaultTools()` 或提供 opt-in 构造函数（网络/有状态工具不进默认集）。跨调用状态用外部 store/Manager 绑定（如 `TodoStore.Tool()` 与 `Manager`），不用包级全局量。`Registry.Add` 同名替换且保持原注册位。

## 后台任务与 deferred tools

- 后台任务（Bash run_in_background、Monitor、交互 shell 族）唯一状态源是 `ToolContext.Tasks`（per-Session `*Manager`）：输出流式写任务文件，模型只拿 task id + 路径，完成通知只带路径不带内容；Tasks 为 nil 时工具降级为 "not enabled" 的 Error Result。超时的前台命令自动转后台（sleep 在禁转名单），单任务输出超 50MiB 被看门狗终止。
- deferred tools 的 schema 不进模型工具列表，仅名字经 `RenderDeferredToolsBlock` 进缓存的系统提示前缀；`SearchExtraTools` 发现 schema，`ExecuteExtraTool` 调用。「延迟」（不列 schema）与「解锁」（UnlockSet 门控调用）相互独立：skill-gated 工具名字可见但锁住。`ExecuteExtraTool` 自身 Permissions 是 Allowed——安全全靠 unlock 检查，改动时勿破坏。

## 坑

- Grep 双后端：ripgrep 路径进程内 `sync.Once` 解析一次（PATH 有 rg 用之，否则尝试 npm 安装 @vscode/ripgrep），失败落纯 Go。测试/禁用用 `NORMA_DISABLE_RIPGREP=1`，有 rg 的环境加 `NORMA_RIPGREP_NO_INSTALL=1` 防自动安装。
- Bash 与 Monitor 共享 catastrophicBash 安全底线（rm -rf / 等，硬 deny 不论模式）——这是底线不是 sandbox，宿主仍须以 CanUseTool 门控。
- `Capture` 有 OutputDir 时全文落盘 + 头部 + 指针，否则 head+tail 截断；Read 例外：从不 spill（源文件已在盘上），改为提示 offset 续读，预算另加 1000。
- 验证：`go test ./tool/...`。文件/搜索工具在 t.TempDir() 真实执行；PTY 测试经 ptySupported() 在无 PTY 平台 skip；`AGENT_CORE_DISABLE_BACKGROUND_TASKS` / `AGENT_CORE_DISABLE_INTERACTIVE_SHELL` 可全局关闭对应能力。
