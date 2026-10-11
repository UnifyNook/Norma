# noaadapter — noa 压缩引擎的 Norma 接入层

把 `llm.Message` 投影到 noa 平坦表示并回投（project.go）、持久化 state 与 tier archive、经 `Enable` 把 Compactor + Compress 工具 + 常驻 prompt 段原子接入 agentcore（noa.go）。一切压缩决策在 noa 包；本包只做投影、持久化、接线与防御。

## 必须维持的契约

- 消息身份是内容哈希（`DeriveMessageID`，哈希前先 `StripRefTag`）。`View` 输出只服务当次请求、绝不写回宿主历史——写回带 tag 或截断过的正文会改变全部 id，令模型持有的 ref 一次性失效（enable_test.go 有专测）。
- `Project`/`Reassemble` 必须成对修改：`Sidecar`（thinking 签名、origins、tool result IsError）承载平坦表示装不下的 provider 数据；thinking 的 Signature 必须逐字节还原，否则 Anthropic 回放拒收；两侧对 `compact_boundary` 的截断必须一致，否则 origins 整体偏移；tool_use 参数回写仅在新文本仍是合法 JSON 时执行。
- `Enable` 是唯一入口且三件套同生共死（缺 Compactor＝压缩进虚空、缺工具＝教不存在的工具、缺 prompt＝摘要丢关键信息），不调它即整体关闭；Compactor 与 Compress 工具必须共享同一个 `Session`，同一 archive 目录不得建第二个 Session。
- 持久化：state.json（带 version，不符即拒读）与 archive 均为 temp+rename 原子写；state 损坏只告警并从空状态起步。丢失可经 `RebuildFromHistory` 重放 transcript 里的 Compress 调用参数恢复（摘要 byte-identical；`reuseArchiver` 只采纳已有 archive、不重写不删除）。`ArchiveBaseDir` 必填，无静默兜底（`ErrNoArchiveDir`）。
- 会话继承（inherit.go）：仅当自身 state 无 blocks 时沿 `ParentSessionIDs` 向上找（最近优先，至多 8 层），继承 ledger（blocks/refs/NextBlockID/TokenSnapshot）但重置 Nudge 与 Stats；继承块保留原 ArchivePath。
- Compress 工具：`RawInput=true` 绕过 harness 的 schema 校验门，否则 `ParseCompressArgs` 的容错解析在生产中不可达（schema 仍对外声明，toolgate_test.go 验证）；ReadOnly/Concurrent 均 false、permission 恒 Allow（只写自身 archive 目录）；永不返回 Go error，一切失败都以 tool result 呈现给模型。
- 压力梯子只认 `resolveTokenCount` 的一个数：prune 后真正发送的视图（`ProjectedTokenCount`），不是 raw 历史；provider 上报值仅作 raise-only floor，且早于上次压缩的过期锚点直接忽略（sentview_test.go / defenses_test.go 钉死此口径）。
- `Materialize` 是「关 noa 继续跑」的纯函数：应用 prune 但不打 ref tag、不动 state 与 transcript，幂等且可逆，无条件调用安全。

## 模式与坑

- 溢出恢复 `Reactive`：用 `ParseOverflowWindow` 从错误文本学真实窗口，按其 95% 重建视图；必须实测发送视图确实缩小（对比 `lastSentTokens`，不是恒更大的 raw projection）才返回 true，否则承认失败——硬返回 true 会制造 413→retry 死循环。
- 失败阶梯：压缩失败与「收到 nudge 但没调工具」同等计数；达 `MaxCompressAttempts` 后抑制 nudge，上下文再增长满 `NudgeGrowthFloor` 或出现真实 user turn 才解除。同一批 range（顺序无关签名）失败两次后，第三次由 DeadRangeTracker 直接拒绝并附存活 block 清单；压缩成功即 Reset。
- ref tag 只以自闭合形式追加在正文尾部；`StripRefTag` 只剥尾部形式且恰好消费一个换行——改贪婪匹配会破坏 round-trip、静默篡改正文。tag 内 tokens 首次渲染即冻结进 TokenSnapshot（保 prefix cache 稳定）。
- 「documented limitation」类测试是 tripwire：故意断言某限制仍在，限制修复后测试会失败且失败信息自述「应改写为正向断言」（tiering_test.go、chaos_test.go 各一例）——先读失败信息再动手改断言。

## 验证

- 仓库 CI 暂未启用 `go test ./...`（.github/workflows/ci.yml 注释：noa 包 2 个参数断言 TODO + 本包 chaos 测试带随机性）；本包验证用 `go test ./noaadapter/ -count=1`（全量约 1 分钟），并发回归加 `-race` 跑 `-run Concurrent`。
- chaos 测试的 seed 硬编码在 chaos_test.go（如 {1,7,42,1337,90210}），RNG 为 `rand.NewPCG(seed, seed^0x9e3779b9)`，同 seed 全程确定性可复现；失败输出含 turn 与行为 kind，按子测试精确复现：`go test ./noaadapter/ -run 'TestChaosAgentPreservesInvariants/seed42' -v`。
- 测试走公开路径：simAgent/chaosAgent 经 `View` 渲染、正则读 nudge 文本、以普通 JSON 调 `runCompress`，不伸入 noa 内部；`checkInvariants` 校验全局不变量（block id 不重复、active block 覆盖不重叠、无消息凭空消失、archive 在盘且自名、请求 provider 可接受）。
