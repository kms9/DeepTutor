# tools-builtin Specification

## Purpose

本模块定义 deeptutor-go 全部内置工具（Level 1 Tool）的行为契约：每个工具的 LLM function-calling schema、输入参数语义、执行结果如何进入 `role=tool` 消息与 `stream.sources`，以及用户可切换 / 自动挂载两类工具集合的划分。实现按三批推进：第一批 LLM/HTTP 类（M1，随 chat 能力交付）、第二批本地状态类与第三批 Sidecar 类（M2，随 memory/skills/notebook/cron/sidecar 各后端模块交付）。工具的挂载时机与 context gate 逻辑属于 agent-loop spec，本 spec 只约束工具本身的输入输出行为。

- 参考实现（基线）：`deeptutor/tools/builtin/__init__.py`、`deeptutor/core/tool_protocol.py`、`deeptutor/tools/{brainstorm,reason,web_search,paper_search_tool,web_fetch,github_query,media_gen_tool,exec_tool,rag_tool,list_notebook,write_note,cron_tool,ask_user}.py`
- 依赖 spec / 里程碑：依赖 llm-provider、agent-loop、sidecar-contract；第二批工具分别依赖 memory、skills-persona、notebook、cron-events；里程碑：第一批 M1，第二/三批 M2

## Requirements

### Requirement: 工具协议与执行结果归一
系统 SHALL 为每个内置工具提供 OpenAI function-calling 兼容的 schema（`name` / `description` / `parameters`，`type=array` 参数必须携带 `items`，缺省补 `{"type":"string"}` 以兼容 Gemini/Anthropic 严格校验）。工具执行 SHALL 返回统一的 tool result 结构：`content`（作为 `role=tool` 消息体回给 LLM）、`sources`（引用条目，经 `stream.sources` 事件下发前端）、`metadata`（自由负载，含结构化 UI 提示）、`success`（`false` 表示显式失败，LLM 仍可读取 `content` 中的错误说明，轮次不因此终止）、`pause_for_user`（仅 `ask_user` 使用）。以 `_` 前缀或专用 kwarg 注入的服务端参数（`_sandbox_*`、`_workspace_dir`、`_cron_owner`、`_tool_loader`、`source_index`、`conversation_history` 等）SHALL 由 pipeline 注入，SHALL NOT 出现在暴露给 LLM 的 schema 中，也 SHALL NOT 接受 LLM 提供的同名覆盖。

#### Scenario: 失败结果仍进入对话
- **WHEN** 某工具执行失败（如 `web_fetch` 目标不可达）并返回 `success=false` 与错误说明 `content`
- **THEN** 该 `content` 仍作为 `role=tool` 消息进入 agent loop 的下一轮上下文
- **AND** 轮次继续执行，LLM 可基于错误说明改变策略

#### Scenario: 服务端注入参数不可被 LLM 伪造
- **WHEN** LLM 在 `exec` 的调用参数里自行携带 `_sandbox_workdir`
- **THEN** pipeline 注入的值覆盖 LLM 提供的值，工具行为不受影响

### Requirement: 工具目录与用户可切换集合
系统 SHALL 注册与基线一致的内置工具目录，并划分两个集合：用户可切换集合（`/settings/tools` 页开关，canonical 顺序）为 `brainstorm`、`web_search`、`paper_search`、`reason`、`geogebra_analysis`、`imagegen`、`videogen`；自动挂载集合（context gate 决定挂载，用户视角锁定开启，partner 可按 companion 配置允许/拒绝）为 `rag`、`code_execution`、`read_source`、`read_memory`、`write_memory`、`read_skill`、`list_notebook`、`write_note`、`web_fetch`、`github`、`exec`、`load_tools`、`cron`、`ask_user`。系统 SHALL 支持 tool alias 表（`rag_hybrid`/`rag_naive`/`rag_search` → `rag` 附加固定参数，`code_execute`/`run_code` → `code_execution`），alias 调用等价于目标工具加上预置 kwargs。

#### Scenario: 用户关闭可切换工具
- **WHEN** 用户在 `/settings/tools` 中关闭 `web_search` 后发起 chat 轮次
- **THEN** 该轮次的工具列表不包含 `web_search`
- **AND** 自动挂载集合中的工具不受该设置影响

#### Scenario: alias 调用归一
- **WHEN** LLM 调用 `rag_hybrid` 并传入 `query` 与 `kb_name`
- **THEN** 系统按 `rag` 工具执行且附加 `mode=hybrid`

### Requirement: geogebra_analysis 不迁移（COMING_SOON）
系统 SHALL 将 `geogebra_analysis` 列入 `COMING_SOON` 集合：不注册到运行时工具注册表（chat agent 无法调用），但 `/api/v1/tools` 列表仍返回该工具条目并带 `coming_soon=true`，供设置页渲染占位卡片。（有意差异：基线 v1.5.2 中该工具已注册可用且属于用户可切换集合；Go 版首发不迁移其 VisionSolver 视觉管线，仅保留目录占位。）

#### Scenario: coming_soon 工具不可调用
- **WHEN** 前端请求 `GET /api/v1/tools`
- **THEN** 响应包含 `geogebra_analysis` 且 `coming_soon=true`、`enabled=false`
- **AND** 任何轮次的可用工具 schema 列表均不包含 `geogebra_analysis`

### Requirement: 第一批 — LLM 与检索类（brainstorm / reason / web_search / paper_search）
`brainstorm` SHALL 接受 `topic`（必填）与 `context`（可选），用独立 LLM 调用做广度优先的方案探索，返回带各方案理由的文本作为 `content`；`reason` SHALL 接受 `query`（必填）与 `context`（可选），用独立的深度推理 LLM 调用解决复杂子问题；两者完整结果对象放入 `metadata`，不产生 `sources`。`web_search` SHALL 接受 `query`，经配置的搜索 provider 返回摘要化答案与 citations：`content` 为答案文本，`sources` 为 `{type:"web", url, title}` 列表。`paper_search` SHALL 接受 `query`、`max_results`（默认 3）、`years_limit`（默认 3）、`sort_by`（`relevance`/`date`，默认 `relevance`），检索 arXiv 预印本：`content` 为逐篇渲染的 markdown（标题/年份/作者/arXiv id/URL/摘要截断 400 字符），`sources` 为 `{type:"paper", provider:"arxiv", url, title, arxiv_id}` 列表；arXiv 限流或网络错误时 SHALL NOT 抛出，而是返回友好的"暂不可用"`content` 且 `metadata.error=true`，空结果返回"未找到"提示。

#### Scenario: brainstorm 正常调用
- **WHEN** LLM 以 `topic="期末复习计划"` 调用 `brainstorm`
- **THEN** 工具发起一次独立 LLM 补全并返回方案列表文本作为 `content`
- **AND** `metadata` 携带完整结果负载

#### Scenario: arXiv 限流降级
- **WHEN** `paper_search` 调用 arXiv API 遭遇限流异常
- **THEN** 工具返回 `content` 为"temporarily unavailable"类提示、`sources=[]`、`metadata.error=true`
- **AND** 轮次不中断

### Requirement: 第一批 — 抓取与查询（web_fetch / github）
`web_fetch` SHALL 接受 `url`（必填）与 `max_chars`（可选，默认 50000），抓取该 URL 并提取可读文本返回 markdown。安全约束 SHALL 与基线一致：仅允许 `http`/`https` scheme；对解析出的主机做私网/回环/链路本地地址拒绝，且在重定向后对最终 URL 再检查一次；原始响应体硬上限 4MB；提取文本超过 `max_chars` 时截断并带明确 truncated 标记；成功时 `sources` 为 `{type:"web", url(最终 URL), title}`，`metadata` 含 `char_count`、`truncated`。`github` SHALL 接受 `query_type`（enum：`pr`/`issue`/`run`/`repo`/`api`）与 `target`，经 `gh` CLI 执行只读查询（命令词汇表仅含 `view`/`list`，`api` 恒为 GET），输出截断至 16000 字符；`gh` 未安装时 SHALL 返回 `success=false` 与"工具在本服务器不可用"的友好说明而非抛错；成功时 `sources` 为 `{type:"github", query_type, target}`。

#### Scenario: 重定向到内网被拒
- **WHEN** LLM 调用 `web_fetch` 且目标 URL 302 跳转到 `http://127.0.0.1:8001/...`
- **THEN** 工具返回 `success=false` 与拒绝原因，不发起对内网地址的请求

#### Scenario: gh 缺失时的降级
- **WHEN** 服务器 PATH 中不存在 `gh` 且 LLM 调用 `github`
- **THEN** 工具返回 `success=false`，`content` 说明 CLI 未安装、可请管理员安装

### Requirement: 第一批 — imagegen 与 videogen（媒体生成）
`imagegen` SHALL 接受 `prompt`（必填）、`size`（可选 `WxH`）、`n`（1–4，默认 1，越界 clamp）；`videogen` SHALL 接受 `prompt`（必填）、`aspect_ratio`、`duration`（均可选）。两者 SHALL 经当前 catalog 激活的生成模型产出媒体字节，写入本次调用专属的运行目录（pipeline 注入 `_workspace_dir` 下的 `imagegen_*/videogen_*` 子目录），以 public artifact 形式返回：`content` 为 artifact 渲染文本，`sources` 为 `{type:"artifact", filename, url, path, mime_type, size_bytes}` 行。模型未配置时返回 `success=false` 与配置提示。`videogen` SHALL 通过 event sink 以 `tool_log` 事件尽力上报渲染进度，进度上报失败不影响渲染。挂载前提（用户开关 + per-user grant + 已配置模型）由 pipeline 判定。

#### Scenario: 生成图片并以卡片呈现
- **WHEN** LLM 以 `prompt="a watercolor fox", n=2` 调用 `imagegen`
- **THEN** 两个图片文件写入本次调用的独立子目录并作为 artifacts 返回
- **AND** 前端通过 `sources` 中的 `url` 渲染下载卡片

### Requirement: 第二批 — 上下文与技能读取（read_source / read_skill / load_tools）
`read_source` SHALL 接受 `source_id`（必填），从 pipeline 注入的本轮 `source_index`（`{source_id: 全文}`）中取出对应全文作为 `content`；id 前缀语义与基线一致：`nb-`（notebook 记录）、`bk-`（book 引用）、`hs-`（历史会话）、`qb-`（题库条目）、`at-`（文档附件）；本轮无 source_index 或 id 未命中时返回 `success=false`，未命中时错误信息列出本轮全部合法 id。`read_skill` SHALL 接受 `name`（必填）与 `file`（可选，默认 `SKILL.md`），经 skill service 读取技能包内文件（路径穿越拒绝、超长截断语义见 skills-persona spec）；非 admin 用户还 SHALL 在 admin 分配技能范围内回退查找；未找到时返回 `success=false` 与"按 Skills 清单原名调用"的提示。`load_tools` SHALL 接受非空 `names` 数组，经 pipeline 注入的 per-turn deferred tool loader 装载 Extended Tools 的完整 schema，返回 `loaded` / `already_loaded` / `unknown` 三组名单文本；loader 缺失（非 chat 上下文）时返回 `success=false`；已装载工具在本会话剩余轮次保持可调用。

#### Scenario: 未知 source_id
- **WHEN** LLM 以不存在的 `source_id="at-xyz"` 调用 `read_source`
- **THEN** 返回 `success=false` 且 `content` 列出本轮全部可用 id

#### Scenario: 装载部分未知工具
- **WHEN** LLM 调用 `load_tools(names=["mcp_search","no_such_tool"])`
- **THEN** `content` 报告 `mcp_search` 已装载、`no_such_tool` 未知并提示使用 Extended Tools 清单中的准确名称
- **AND** `success=true`（存在成功装载项）

### Requirement: 第二批 — read_memory 与 write_memory
`read_memory` SHALL 无参数，返回当前用户 L3 四份文档（`recent`/`profile`/`scope`/`preferences`）非空正文按 `---` 分隔的拼接；全部为空时返回固定的"暂无记忆"提示文本。`write_memory` SHALL 接受 `op`（enum `add`/`edit`，必填）、`text`（必填，≤240 字符约定）、`target_id`（`edit` 必填，形如 `m_xxx`）、`reason`（可选）；执行时 SHALL 先向 L1 chat surface 追加一条 `preference_stated` trace 事件，再以该 trace id 作为来源引用写入 `L3/preferences.md`（详见 memory spec）；写入被拒时返回 `success=false` 与拒绝原因。两工具的路径解析 SHALL 随当前用户上下文，多用户安全。

#### Scenario: 显式偏好落盘
- **WHEN** 用户明确说"以后都用中文回答"且 LLM 调用 `write_memory(op="add", text="总是用中文回答")`
- **THEN** L1 trace 先记录 `preference_stated` 事件
- **AND** `preferences.md` 新增条目且其来源引用指向该 trace id
- **AND** 工具返回包含新 entry id 的确认文本

### Requirement: 第二批 — list_notebook 与 write_note
`list_notebook` SHALL 支持两种模式：无参调用返回用户全部笔记本索引（id/名称/描述/记录数/更新时间，上限 50 条并注明截断）；带 `notebook_id` 调用返回该笔记本内记录列表（record id/标题/类型/摘要/创建日期，按创建时间倒序，上限 80 条）。未知 `notebook_id` 时返回 `success=false` 并列出合法 id。`write_note` SHALL 支持 `mode="append"`（新建记录：`title` 必填；`content` 缺省时由工具用注入的 `conversation_history` + `current_user_message` 渲染最近 `turns_to_include`（整数字符串或 `"all"`，默认 3）轮真实对话稿作为正文，提供 `content` 时保存 agent 撰写的 markdown；`note` 作为置顶评注）与 `mode="edit"`（按 `record_id` 修改既有记录的 title/正文/summary，`record_id` 必填）。长度上限 SHALL 与基线一致（title 200、note 4000、content 200000 字符，超限截断）。所有错误以 `ok=false` 结果而非异常返回。

#### Scenario: 默认保存真实对话稿
- **WHEN** LLM 调用 `write_note(mode="append", notebook_id=<合法 id>, title="二次函数答疑")` 且未提供 `content`
- **THEN** 新记录正文为最近 3 轮 user/assistant 真实对话的渲染稿，而非 LLM 生成的摘要
- **AND** 返回文本包含新 record id

### Requirement: 第二批 — 轮次控制类（cron / ask_user）
`cron` SHALL 接受 `action`（enum `schedule`/`list`/`cancel`，兼容 `add`→`schedule`、`remove`→`cancel`），归属上下文来自 pipeline 注入的 `_cron_owner`（缺失时返回"当前上下文不可调度"）。`schedule` SHALL 要求 `message` 且 `at` / `every_seconds` / `cron_expr` 三者恰好其一（`at` 为 ISO 8601，无时区按服务器本地时区；`every_seconds` 最小 30；`cron_expr` 为 5 段 cron 表达式，可配 `tz` IANA 时区），成功后返回 job id、调度描述与首次运行时间；在 cron 触发的轮次内（`_cron_in_context`）SHALL 拒绝再次 schedule。`list` 仅列出当前 owner 的任务；`cancel` 要求 `job_id` 且仅能取消当前 owner 的任务（持久化与执行语义见 cron-events spec）。`ask_user` SHALL 接受 `questions` 数组（1–4 个问题，每个含 `prompt` 必填、`header`、`options`（`label` 必填 + `description`）、`multi_select`、`allow_free_text`、`placeholder`）与可选 `intro`；遗留顶层 `{question, options}` 入参 SHALL 仍被接受并归一为单元素 `questions`，但不在 schema 中暴露。`ask_user` 的执行结果 SHALL 设置 `pause_for_user` 为结构化问题负载：chat loop 据此暂停本轮、发出 `pending_user_input` 事件、等待用户在同一轮内回复，回复到达后以用户答案替换该 tool result 的 `content` 并恢复同一循环；占位 `content` 仅在运行时中途崩溃时才会被 LLM 看到；参数校验失败时返回 `success=false`，不暂停轮次。

#### Scenario: 跨会话任务不可见
- **WHEN** 会话 A 中调用 `cron(action="list")` 而定时任务全部由会话 B（不同 owner key）创建
- **THEN** 返回"本会话无定时任务"

#### Scenario: 提问并恢复
- **WHEN** LLM 调用 `ask_user` 询问"报告输出语言"并提供 2 个选项
- **THEN** 轮次暂停并向前端发出携带问题负载的 `pending_user_input` 事件
- **AND** 用户选择后，loop 以"用户回答：…"作为该 tool call 的结果内容恢复执行

### Requirement: 第三批 — rag 检索（Sidecar 转发）
`rag` SHALL 接受 `query`（必填非空）与 `kb_name`（必填非空，必须是本轮附加的知识库之一），经多用户访问控制解析 KB 归属后，将检索请求转发给 Python Sidecar 的 RAG 接口并等待综合答案；`content` 为答案文本，`sources` 含 `{type:"rag", query, kb_name}`，`metadata` 为 Sidecar 返回的完整结果。工具 SHALL NOT 做任何默认 KB 回退或别名解析（kb_name 缺失即报错）；同轮多 KB 检索由 LLM 逐 KB 发起调用、系统并行执行。检索管线本体行为见 sidecar-contract spec。

#### Scenario: 缺少 kb_name 直接报错
- **WHEN** LLM 调用 `rag(query="什么是 LUT")` 未提供 `kb_name`
- **THEN** 工具以参数错误结果返回，不发起 Sidecar 请求

### Requirement: 第三批 — exec 与 code_execution（沙箱执行）
`exec` SHALL 接受 `command`（必填）与 `timeout`（默认 30s，clamp 至 1–300）；执行前先按基线 deny-pattern 列表（`rm -rf`、`mkfs`、`shutdown`、fork bomb 等）做防御性拦截（命中即返回被拦截提示，沙箱隔离才是真正安全边界），然后以本轮 workspace 为工作目录、连同注入的 mounts 提交给 Sidecar 沙箱执行；`content` 为渲染后的 stdout/stderr（按输出上限截断）加 artifact 清单，`success = 运行成功且 exit_code==0`，`metadata` 含 `exit_code`、`timed_out`、`sandbox_error`、`artifacts`。`code_execution` SHALL 接受 `language`（`python`/`c`/`cpp`，含 `py`/`python3`/`c++`/`cxx`/`cc` 别名归一）、`code`（必填）、`stdin`（可选）、`timeout`（同 clamp）；每次调用 SHALL 在 workspace 下创建独立 run 子目录写入源文件（及 `stdin.txt`），按语言模板编译运行（`python3 {src}`；`cc -O2`；`c++ -std=c++17 -O2`），采集 artifacts 时 SHALL 排除源文件、编译产物 `prog` 与 `stdin.txt`。两工具在沙箱后端未配置时不挂载。

#### Scenario: 并发代码执行互不干扰
- **WHEN** 同一轮内 LLM 并发发起两次 `code_execution`
- **THEN** 两次调用分别在独立 run 子目录写源码与产物，互不覆盖

#### Scenario: 危险命令被预检拦截
- **WHEN** LLM 调用 `exec(command="rm -rf /")`
- **THEN** 工具不提交沙箱，直接返回 `success=false` 的"命令被拦截"提示
