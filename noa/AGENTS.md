# AGENTS.md

模型驱动的上下文压缩引擎纯核心：摘要由模型写成并以字符串参数传入，noa 只编排——决定何时压、压哪段、记账、应用压缩决定、裁剪视图。无 I/O（唯一持久化口是 Archiver 接口，由宿主实现，且要求原子写）、不调模型、不读时钟（CreatedAt/Now 由调用方传入）、只依赖标准库；接入 Norma 主干的宿主侧在 noaadapter 包。移植自 acp-kernel（MIT）。

## 必须维持的契约

- **CoreMessage 平坦表示**：宿主把自身消息投影为扁平 CoreMessage（一条宿主消息 → 0..N 条，每 content block 一条），处理后重组，noa 永远看不到宿主类型。Text 是已剥离 ref tag 的裸正文——tag 不得进 identity hash，否则 id 随重渲染漂移。
- **ref 永不重编号**：mNNNNN（1..99999，5 位补零使字典序=数值序）一经分配终身有效；硬保护消息记为 BlockedRef，占 ByRaw 防重访但不占号、模型不可寻址。
- **TokenSnapshot 冻结**：每个 ref 首次渲染时的 token 数永不更新——ref tag 变一个字节，前缀缓存即失效。
- **节点顺序不可协商**（processturn.go）：assign-refs → sync-blocks → prune → hide-compress-calls → emergency-truncate → nudge-inject。前 4 个（sizingNodes）不读 TokenCount，可在测量前跑；后 2 个读视图大小，必须在测量后跑。宿主两段式：ProjectedTokenCount（只跑 sizingNodes，测 pruned 视图）→ ProcessTurn；改测 raw 历史会让压力阶梯永久钉死在高压区。
- **ProcessTurn 是纯函数**，输入不被 mutate：节点内部一律先 CloneState 再自由改。压缩成功后清零 Nudge 基线（ApplyCompression 已做），否则下轮误判增长、重放模型刚执行过的 nudge。
- **摘要消息识别四条件全中**：id 前缀 + RoleUser + CTText + SummaryHeader；只查 id 前缀会让宿主撞名消息被 rebuild 静默删除。
- **Tier 一次只压一层**：DecideTier 只吸收范围内最低 tier 的 block，更高 tier 存活（SurvivingBlockIDs）——这条规则同时是 tier 3 可达的前提。block 的 Tier 创建后固定；BlockID 跨 tier 单一扁平序列（数字即全局创建顺序）。
- **TierConfig.MaxTier 不是扩展点**：仅允许 2 或 3（tier 规则 prompt 手写至 3），是测试杠杆——触达终层守卫，MaxTier=3 需约 100 次合成压缩，=2 约十次。
- **三种保护语义互不相同**：AlwaysProtectedTools/Config.ProtectedTools 是硬排除（永不压缩、不分配 ref）；PreserveRecent* 是随对话前移的软保护区；NeverPreserveRecentTools（Norma 工具名）只是不占软保护区名额，仍完全可压缩。
- **ApplyCompression 六阶段有序**：解析 → 批内重叠仲裁（先到先得）→ 原子大小门（整批拒绝，block 边界范围豁免）→ 内存建块 → 全部归档落盘 → 才改 state；任一归档失败整批回滚（含已写 archive）。无归档的 block 不允许存在。

## 已知现状与坑

- **CI 暂未启用 `go test ./...`**（见 .github/workflows/ci.yml 注释）：本包 2 个参数断言类测试当前失败，属设计参数 TODO 而非逻辑 bug——TestAdaptiveGrowthTracksTheWindow（默认 GrowthFloor == GrowthCap == 50000，clamp band 零宽，GrowthRatio 恒不生效，自适应退化为常数）与 TestSuppressionReleaseFitsInsideTheWindow（40k/64k 窗口下 suppression 释放需 56%/35% 窗口，越过 25% 上限）。修的是 DefaultConfig 的 Nudge 默认值；修好后取消 ci.yml 中对应注释。
- **prompts_rules.go 四条压缩规则从 acp-kernel VERBATIM 移植、不得翻译**（翻译即 override）；其中 4 处是有意的事实性修正——上游有 decompress 工具而 noa 没有。
- **ParseCompressArgs 契约**：永不 panic、永不发明范围（少恢复可接受，多恢复不行）。CompressToolSchema 的 content 故意不带 `type: "array"`，兼容把嵌套参数字符串化的 provider。
- **CompressToolName 是唯一真相源**：该字符串散布在至少 6 处，一切引用经此常量。
- ApplyResult 的 Errors/Warnings 是 model-facing 校正文本：必须告诉模型「该怎么做」，而非只报「哪错了」。ValidateConfig 只返回 warnings 不返回 errors，调用方记录后继续。

## 验证命令

- `go test ./noa/`（fuzz 种子随普通测试运行；当前含上述 2 个已知失败）
- 单测：`go test ./noa/ -run TestDecideTier`
- 真正 fuzz（一次只能 fuzz 一个目标）：`go test ./noa/ -run '^$' -fuzz FuzzParseCompressArgs -fuzztime 30s`（另有 FuzzParseBoundary/FuzzTruncateBody/FuzzAssignRefs/FuzzStubSummary）
