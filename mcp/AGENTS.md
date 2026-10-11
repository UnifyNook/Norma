# mcp — MCP 集成：stdio JSON-RPC 客户端与进程内 server

外部 MCP server 经换行分隔的 JSON-RPC 2.0 stdio 接入，Go 函数经 `InProcessServer` 以同一约定暴露；两者都产出 `tool.CoreTool`，命名统一走 `ToolName`：`mcp__<server>__<tool>`（三段式、双下划线，名字里的下划线不做转义）。本包只做协议适配与传输，不做 schema 校验与调度。

## 必须维持的契约
- 生命周期：`NewStdioClient` = spawn 子进程 + `initialize` 握手（protocolVersion `2024-11-05`、clientInfo norma，随后 `notifications/initialized` 通知）；握手失败必须 `Close()`——`stdioCloser` 关 stdin、Kill、Wait 三连，任何改动不得泄漏子进程。
- 工具 Run 一律把错误折叠为 `tool.Errorf("Error: "+…)` 且 Go error 返回 nil：MCP 故障作为 tool_result 文本随循环流动，绝不中断 harness；远程响应的 `isError` 原样透传为 `Result.IsError`。
- 权限默认 `permission.AskUser`（进程内与远程一致）；放行由宿主经 AllowedTools 等机制做，不要改这里的默认值。
- 响应按自增 int ID 关联（从 1 起）；解析失败或 ID==0 的帧被静默忽略——server 复用 ID 或对 ID 0 回包会破坏关联。单行（单帧）上限 8MB（scanner buffer）。

## 模式与坑
- `conn` 刻意只依赖 io.Reader/io.Writer（传输无关）：测试经私有 `newClientWithConn` + io.Pipe + `fakeServer` 桩全内存跑通握手/tools/list/tools/call，不 spawn 真子进程；改传输逻辑先看 mcp_test.go 这套桩。
- `tools/call` 结果只拼接 `type:"text"` 块；`inputSchema` 为 nil 时补 `{"type":"object"}`；`env` 参数叠加在父进程环境之上（nil 则原样继承）。

## 验证
- `go test ./mcp/`：命名约定（TestInProcessServerNamingAndCall）与管道上的完整客户端往返（TestStdioClientOverPipes），无外部依赖。
