# permission

Agent 的安全边界：allow/deny/ask 三态、六种 `PermissionMode`、deny 优先的规则匹配与四段 `Evaluate` 管线。本包刻意不 import `tool`——`Request.IsReadOnly` 与 `Request.ToolDecision` 由 harness（`harness/tools.go`）从工具侧取好后传入。

## 不变量（勿破坏）
- 四段顺序即 deny 优先语义：deny 规则与工具自判 `Deny` 绝对优先（bypass 也压不住，见 `TestDenyPriorityOverBypass`）→ mode 短路（bypass 全放行；plan 只放行 read-only，不走 allow 规则）→ allow 规则 / acceptEdits 的 `Write|Edit|NotebookEdit` / read-only / 工具自判 `Allow` → ask。调整顺序必须同步 `permission_test.go` 的矩阵。
- Fail-closed 收尾：`Ask` 为 nil、回调返回空 `Behavior`、或 mode 为 dontAsk/auto 不可提示时，一律 `Deny`。新增分支必须以 deny 兜底，不得默认放行。

## 非显而易见
- `matchRule` 只匹配 `(` 之前的工具名：`"Bash(git:*)"` 会放行任意 Bash 调用，括号内参数不参与匹配；`"*"` 匹配一切。
- 第 3 段的工具自判 `Allow` 不区分 mode，dontAsk 下同样放行；dontAsk 与 auto 的实际差异只有第 4 段的拒绝文案（与 `ModeAuto` 常量注释的字面含义有出入，改语义时以测试为准）。
- `Decision.UpdatedInput`（改写输入，如剥除危险 flag）与 `Suggestions`（"always allow" 持久化建议）只由本包生产，落地是宿主（harness / 审批 UI）的职责。
