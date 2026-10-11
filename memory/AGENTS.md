# memory — 文件制持久长期记忆

每条记忆是带 frontmatter（name/description/type）的 markdown 文件，存于 `Store.Dir`；`MEMORY.md` 是索引。agent 侧经 `RecallMemory`（相关性检索）与 `RecordMemory`（落盘）两个工具读写；宿主侧经 `SystemSegment`（索引进 system prompt）与 `RenderForInjection`（按 prompt 注入）做被动披露。与 skill 的边界：memory 存「事实」，skill 存「操作流程」。

## 必须维持的契约
- 磁盘格式兼容性：frontmatter 解析是手写 `strings.Cut`（非 YAML 库），只认 name/description/type 三个裸 `key: value`，值不做引号剥离/多行处理；无 frontmatter 的 .md 也能读（全文当 Content，Name 回退文件名）。文件名 = `slug(name)+".md"`（小写、非字母数字折叠为 `-`），同名 Save 即覆盖（upsert），`Read` 同时接受原名或 slug。
- 每次 `Save` 全量重建 `MEMORY.md`；但 `Index()` 优先读盘上已有文件——手改索引对 `SystemSegment` 可见，直到下次 Save 被覆盖。
- `Scan` 静默跳过解析失败的文件；目录不存在时返回 nil,nil 而非报错（目录在首次 Save 才创建）。
- 两个工具都 `permission.Allowed()`——RecordMemory 虽是写操作但不走审批；错误一律 `tool.Errorf("Error: …")` 内联返回，不返回 Go error，缺 name/content 的校验在 Run 内做。

## 模式与坑
- `Relevant` 是子串记分（name×3 / description×2 / body×1；token 小写、len≥2、去 stopword），平分按 ModTime 新者优先；max<=0 表示不限，RecallTool 固定取 5。注入时超过 48h 的记忆自动附「非实时状态，需对照当前源码验证」的 stale 提示。

## 验证
- `go test ./memory/`：Save→Scan→Read 往返、索引 slug 链接与落盘 frontmatter 形状、相关性排序、两工具端到端（临时目录，无环境依赖）。
