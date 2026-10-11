# llm — 供应商中立的消息模型与 Provider 适配层

统一模型（`Message`/`ContentBlock`/`StreamEvent`）+ `Provider` 接口，内置 Anthropic Messages、OpenAI Chat Completions、OpenAI Responses 三种 wire format 的适配器。内部模型镜像 Anthropic 的 content-block 形状（超集），OpenAI 侧在其边界翻译；harness/tool 只面向统一模型，本包不做工具执行与 agent 循环。

## 必须维持的契约
- 三个适配器都归一化为同一组 `SE*` 流事件；新 wire 事件必须映射进该词表，由 `Accumulator` 重组为 `Message`，不得透传厂商专有事件。
- Usage 口径：`InputTokens` 是含 cache 的总输入（OpenAI prompt_tokens 语义）；Anthropic 原生 `input_tokens` 只含非缓存部分，`norm()` 折入 cache read+creation。Anthropic 流中 message_start 只带输入侧、message_delta 只带输出侧——防止累加时双计，改解析勿破坏。
- 各适配器 buildBody 的整形链即契约：Anthropic 先 `filterThinkingBlocks`（thinking 关闭时剥 thinking 块并丢弃清空的消息）再 `mergeAdjacentSameRole`（API 要求 role 交替）；OpenAI `sanitizeOpenAIMessages` 双向丢弃 tool_call↔role:tool 半对——它是兜底而非主修复，上游应保证配对；Responses 无状态（store:false）、回放时丢 thinking 块、system 走顶层 instructions。
- `CompletionRequest.Thinking` 优先于 `Config.ThinkingType`（""=继承）；compaction 摘要调用固定传 "disabled"。OpenAI thinking 模式下回放 tool_calls 的 assistant 轮必须带 reasoning_content，取自 `b.Thinking` 而非 `b.Text`（否则 DeepSeek 风格网关 400）。
- OpenAI 输出上限两键互斥、恰好发一个（`Config.MaxTokensField`，默认旧键 `max_tokens`）；Anthropic 与 Responses 自行命名该字段，忽略此开关。
- 重试只覆盖建流阶段（网络错误/408/429/5xx），流中断原样抛错绝不重试；空响应重试（仅 FormatOpenAI）独立计数，只在 end_turn/"" 停止时触发——max_tokens 截断归 harness 上抛处理，重试会与它打架。
- OpenAI 网关 reasoning 字段三名并存：`pickReasoning` 按 reasoning_content→reasoning→reasoning_text 取首个非空（兼去重）；`refusal` 须读成文本，否则拒绝轮解码为空轮。

## 模式与坑
- 扩展点：新厂商 = 新 `Format` 常量 + `NewProvider` 分支 + 实现 buildBody/Stream/Complete；`CompletionRequest` 新增字段必须在每个适配器的 buildBody 里显式决定去向（含 omitempty 取舍）。
- RateLimit 按 Provider 实例共享（limiter 指针随 cfg 拷贝），只计逻辑请求、重试不消耗；自定义 Provider 用 `RateLimited` 包装。
- boundary/skillmsg/todomsg 是消息层构件：`MessagesForAPI` = 最后一个 boundary 之后的工作集 + 剥标记 + `PairToolBlocks` 丢孤儿 tool 块；skill/todo reminder 靠固定 sentinel 文本自识别，改措辞会破坏 compaction 与节流检测。

## 验证
- `go test ./llm/...` 无网络与环境依赖：全部经 httptest 桩 SSE（helpers：`sse`、`event`/`sse2`、`captureBody`/`captureReq`）。坑：空 SSE 桩会触发 OpenAI 空响应重试——只断言请求体的测试须给非空 stub 内容。
