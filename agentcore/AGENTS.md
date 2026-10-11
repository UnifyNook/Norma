# agentcore

SDK 公共入口（聚合层）：把 `Options` 装配成有状态 `Session`，`Prompt` 以 range-over-func 流式驱动 `harness.Query`，`Run` 是自动 `Close` 的一次式封装。本包只做装配与转发——循环在 harness、压缩算法在 compaction、工具实现在 tool，改这些去对应包。SDK 不注入任何 preset system prompt，`Options.SystemPrompt` 完全归宿主。

## 不变量与契约

- `Session` 非并发安全，禁止对同一 Session 并发 `Prompt`。messages/usage 在迭代器吐出 `KindResult`（`ev.Terminal`）时写回；在该事件前提前 break 退出迭代会留下过期状态。
- 上下文管理器二选一：设了 `Compactor`（宿主接管）则 `Compaction` 被忽略并经 `OnWarn` 告警（compactor_test.go 固化）。`compaction.Compactor` 绝不能实现 `harness.ContextView`——noa_off_test.go 钉死此约束，否则每个 Session 会静默改走投影路径绕过 boundary 切片。
- SDK 自身不写 stdout/stderr：配置告警只投递给 `OnWarn`（NewSession 内同步调用，nil 安全）。
- sessionID 恒生成（不开 Transcript 也需要——它命名后台任务临时目录 `<tmp>/norma/<sessionID>`）；`Close()` 杀后台任务并清目录、可重复调用，长驻宿主应 defer，`Run` 自动调用。
- `Resume` 必须先设 `Options.Transcript` 否则报错；`cleanResumed` 丢掉尾部未落 tool_result 的 assistant tool_use（悬挂 tool_use 会被 API 拒绝）；续写经 `ResumeWriter` 接上 `LastUUID` 父链，sessionID 随之替换。
- `DeferredTools` 非空时自动注册 `SearchExtraTools`/`ExecuteExtraTool` 并向模型隐藏原 schema；`UnlockSet` 为 nil 时默认全解锁，宿主对其 `Add()` 实现技能式按需披露。

## 非显而易见

- WebSearch 后端配置错误时丢弃该工具而非使整个 Session 失败（NewSession 内注释写明该取舍）。
- 仅当开启持久化时 `Prompt` 才把 sessionID 挂上 ctx（`transcript.WithSessionID`）——subagent sidechain 落到父会话目录依赖它。
- `Prompt` 前置顺序：memory 自动注入与 UserPromptSubmit hook 的额外上下文都追加到用户输入尾部，随后排空后台任务完成通知（作为独立 user message 注入并落盘），最后才写本轮用户消息。
- noa_off_test.go 是「默认惰性」守卫组：noa 相关能力必须 additive 且默认无副作用；给默认工具集加东西时确认未绕过 schema 校验（`Spec.RawInput`）。

## 验证

`go test ./agentcore/`——全部经 httptest SSE 假服务或注入 `Deps.CallModel` 脚本化轮次驱动，无需真实 API key 或网络。
