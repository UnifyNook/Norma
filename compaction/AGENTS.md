# compaction — 上下文窗口压缩（MicroCompact / AutoCompact / Reactive）

`Pre` 在每次模型请求前按序执行：过 microThreshold（effectiveWindow 的 60%）先 MicroCompact（把 stale COMPACTABLE 工具结果正文换成可重读的占位文本，保留最近 `microKeepRecent`=5 条），达 autoThreshold 或 predictive 预测下轮溢出再 AutoCompact（在保留尾前插入 boundary + 摘要）。`Reactive` 在 prompt-too-long 后只重跑一次 AutoCompact，不做 micro/snip 级联。Snip 与 Context-Collapse 有意不实现（前者无自动触发源，后者 claude-code 默认关闭）。常量与 summary prompt 与 claude-code 1:1 对齐，改 `baseCompactPrompt` 不得顺手规范化文本（须保留源尾随空格）。

## 必须维持的契约

- 非破坏性：AutoCompact 恒输出「原数组 + boundary + summary(+重注入 skills)」（测试断言 `len(out)==len(msgs)+2`），从不动历史；模型只见 `llm.MessagesForAPI`（最后一条 boundary 之后），`count` 也只统计该 API 视图。
- 熔断器：summarizer 连续失败 3 次（`maxAutoCompactFails`）即 `tripped`，代码无解除路径，Pre 与 Reactive 的 AutoCompact 随之失效；任一次成功把失败计数清零。MicroCompact 不受熔断影响。
- `Reactive` 的 bool 语义是「API 视图确实变小」，不是「插入了 boundary」：摘要大于原工作集时 boundary 已插入仍返回 false，避免 413→retry→413 死循环。
- skill 重注入保证指令存活：只重注入最新 invocation 落在新 boundary 之前的 skill，逐字复制（每条 5000、合计 25000 token 预算，截断附说明），超预算 newest-first 保留后恢复时间序；最新 invocation 已在保留尾部的 skill 跳过。
- token 计数优先 `lastInputTokens`（上次响应报告值），仅 0 时回退 `EstimateTokens*4/3`；工作集被改写后必须传 0 强制重估。buffer 分层按 effectiveWindow（非原始窗口）取档。

## 模式与坑

- 禁用 `Pre` 须显式负 `ContextWindow`（`New` 把 0 默认成 200000）；`summarize == nil` 只禁 AutoCompact（MicroCompact 照常），`Reactive` 则恒 false。
- `ProviderSummarizer` 对 summary 请求自身溢出按 API round 分组丢最旧 20%（≥1 且保留 ≥1 组）重试，结果以 assistant 开头时补 `ptlRetryMarker`（API 要求 user-first）；overflow 立即上抛进入该 PTL 循环，不消耗 streaming 瞬态重试（上限 2 次）。
- `pairForReplay` 对悬空 tool_use/tool_result 是丢弃而非合成占位（摘要只需周边文本）。
- AutoCompact 的保留尾用 `Config.KeepRecent`（默认 8 条消息），与 MicroCompact 的 `microKeepRecent`（5 条工具结果）是两个独立概念。

## 边界与验证

- 本包仅 import `llm`，与 noa 系包互不依赖（import 已核实）。反向：`noaadapter` import 本包只为复用 `IsOverflow`，并以同一 `harness.Compactor` 接口作为模型驱动压缩的替代实现（其 `Pre` 是 no-op——noa 依赖消息内容哈希身份，历史不可改写）。`compaction.Compactor` 绝不能实现 `harness.ContextView`（agentcore/noa_off_test.go 钉死，否则 Session 静默改走投影路径绕过 boundary 切片）；改 `IsOverflow` 字符串匹配会同时影响 noaadapter。
- `go test ./compaction/` 纯单元测试无外部依赖；阈值数学用 `raw()` 直构 `Compactor` 绕过 `New` 默认值做表驱动断言。
