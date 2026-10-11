# cmd/agentcore

REPL 演示入口：展示 provider + 宿主 system prompt（内置编码 prompt，可经 `-system FILE` 替换）+ 核心工具 + 交互式审批的最小组装。定位是示例不是库代码——保持单文件、只做装配与终端 IO；新能力加到 agentcore 与各子包，不要在这里堆功能。

## 契约与坑

- flag 即契约：`-model` 必填（缺失 exit 2，可用 env `AGENT_MODEL`），`-provider` 默认 env `AGENT_PROVIDER` 或 anthropic，`-p` 单轮即退。改名/默认值会破坏 README 的调用示例。
- 审批交互：`CanUseTool` 恒为终端 `[y/N]` 询问，仅 `y`/`yes` 放行，其余一律 `Denied`；`-yes` 只是把 PermissionMode 切到 `ModeBypass`（回调仍挂着，权限管线不再走到 ask），并非移除回调。
- `AskUserQuestion` 交互：按编号选项作答，MultiSelect 逗号分隔；非数字输入原样作为自由文本回传。
- 内建命令 `/exit`、`/quit`、`/reset`（调 `Session.Reset()`）；Ctrl-D 退出；SIGINT/SIGTERM 经 `signal.NotifyContext` 传播。本演示未接 Transcript/Compaction/Memory，不代表 SDK 默认行为。

## 验证

`go build ./cmd/agentcore`（本目录无测试文件）。
