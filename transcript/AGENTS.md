# transcript

会话转录持久化底层：按会话追加式 JSONL，`Writer` 实现 `harness.Recorder`——agentcore 把它挂进 `QueryInput.Recorder`，由 harness 在每条新消息诞生时回放进来。本包只管落盘与重建，不感知 agent 循环。

## 契约

- 布局：主文件 `<dir>/<sessionID>.jsonl`，subagent sidechain 在 `<dir>/<sessionID>/subagents/agent-<agentID>.jsonl`（token 各自单独计）。sidechain 归属靠 ctx 上的 sessionID（context.go），agentcore 开持久化时注入。
- Record 以 UUID/ParentUUID 成链，首条无父；续写用 `ResumeWriter(sessionID, LastUUID(...))`。重建函数 `Messages` 跳过 `Type=="boundary"`（纯观测记录）并把各 assistant usage 累加为总量。
- 压缩边界后的工作集（boundary 标记 + 摘要 + 保留尾部）会作为 message **再录一遍**：冷启动 Resume 恢复的是压缩后视图而非重放全史（agentcore 的 TestResumeRestoresCompactedView 钉死）；保留尾部因此在盘上出现两次。MicroCompact 等不产生边界的编辑不落盘。
- Append 失败不 panic：错误留存于 `Writer.Err()`，磁盘故障退化为内存运行——宿主须主动检查 `Err()`。
- `Load` 对缺失文件返回 (nil, nil)（新会话语义），损坏行报 `transcript: corrupt line`；scanner 上限 64MB，容忍大体积工具输出。

## 验证

`go test ./transcript/`——纯 t.TempDir 文件系统测试，无网络。
