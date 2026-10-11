# plan

计划模式：只读探索 → `ExitPlanMode` 呈报审批 → 权限模式中途翻转到 target，agent 据此实施。核心是 `Controller`（互斥锁保护的实时 mode）加两个工具；agentcore 把 `Controller.Mode` 接到 `QueryInput.PermissionModeFunc`，翻转无需重启循环即对后续每次权限判定生效。

## 时序契约
- `Enter` 记住进入前的 mode（`prePlan`）并翻到 `ModePlan`；已在 plan 中时重复 Enter 是幂等 no-op。
- 批准后 `approve()` 把 mode 置为 **target**（`NewController` 的 target 为空则默认 `ModeAcceptEdits`），不是恢复 `prePlan`——prePlan 目前只写不读，勿假设"批准即回原模式"。
- 拒绝：返回 error 工具结果（附 feedback），mode 停留 plan，提示模型修订后重报；没有重试计数，模型可无限次走这个修订循环。
- 不在 plan 中调用 `ExitPlanMode`：返回普通文本 no-op，不改 mode、不报错；`Approver` 为 nil 视为自动批准。

## 非显而易见
- 两个工具都标 `ReadOnly=true` 且 `Permissions` 恒 `Allowed()`——不这样它们自己就会被 plan 模式的"只许 read-only"拦下，根本调不到；`ExitPlanMode` 另标 `Concurrent=false`（要弹审批并改 mode，绝不能被并行早跑）。
