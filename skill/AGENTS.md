# skill — SKILL.md 指令包与 Skill 工具

技能 = `LoadDir` 加载的 `SKILL.md`（或裸 .md）：YAML frontmatter（name/description/license/compatibility，`mcps:` 声明需解锁的 MCP server）+ Markdown 正文。`Registry.Tool()` 产出 Skill 工具：模型按名调用，正文作为独立 user 消息注入对话。格式对齐 agentskills.io 标准。与 memory 的边界：skill 是按需加载的操作流程，不是持久事实。

## 必须维持的契约
- 注入通道即抗 compaction 保证：指令经 `llm.SkillInvocationMessage` 包成 `<invoked-skill name="…">` 信封的独立 user 消息放进 `Result.Extra`，tool_result 只留一句 ack——compaction 靠信封 sentinel 在摘要边界后逐字重注入各技能最新一次调用（每技能 5K / 总量 25K token 预算）。改措辞或改通道都会静默毁掉该机制。
- `Registry.OnInvoke` 是宿主钩子：本包自己不解锁任何 MCP，只把 `Skill.MCPs` 交给宿主，由宿主在钩子里 reveal deferred tools 并 Add 进 `UnlockSet`（见 agentcore）；钩子返回文本追加到 tool_result。
- frontmatter 用真 yaml.v3 解析；坏 YAML 优雅降级（字段空、正文照载，绝不丢技能）；`stringOrList` 兼容标量（含逗号分隔）/流式/flow 三种 `mcps` 写法，`mcp`/`when_to_use` 等旧键名均接受。`WhenToUse` 是 legacy，仅 Description 为空时回退展示。
- `LoadDir` 同时接受「子目录/SKILL.md」与「目录下裸 .md」两种布局；`Dir` 字段用于替换正文中的 `${SKILL_DIR}`（裸 .md 的 Dir 是父目录）。单文件读失败静默跳过，仅 dir 本身不可读才报错。

## 模式与坑
- 发现列表（tool 的 Prompt）有预算：总 8000 字符、单 description 截 500，超预算后降级为只列名字——技能永不从列表消失（模型必须始终可调用），测试对此有断言。
- `Add` 同名替换但保留首次注册位置；`List` 按注册序而错误提示的 Available 列表按字母序。Skill 工具本身 ReadOnly+Allowed，`args` 以「Additional context from the caller」小节拼在正文后。

## 验证
- `go test ./skill/`：真 YAML（引号冒号、block 标量、block/flow 列表）、坏 YAML 降级、listing 预算不丢名、LoadDir 双布局、Extra 消息信封自识别、OnInvoke 触发与返回值追加。
