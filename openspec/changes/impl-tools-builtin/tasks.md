# Tasks: impl-tools-builtin

## 1. 协议与注册骨架

- [ ] 1.1 实现 `internal/tools/protocol.go`：`ToolResult` / `Source` / `Injected` / `BuiltinTool` 接口与 `InjectionFromContext` 帮助函数
- [ ] 1.2 实现 `internal/tools/einoadapter/adapter.go`：`BuiltinTool` → eino `tool.InvokableTool` 适配（args 反序列化、`_` 前缀键剥除、alias presets 合并、`success=false` 不返回 error、ToolResult 旁路 registry）
- [ ] 1.3 实现 `internal/tools/registry.go`：工具目录（canonical 顺序）、Toggleable/AutoMount 集合、alias 表、`COMING_SOON`（`geogebra_analysis`，D-004）、schema 后处理（`type=array` 补 `items`）
- [ ] 1.4 单测：注入参数不可被 LLM 同名覆盖；alias `rag_hybrid` 归一为 `rag`+`mode=hybrid`；array 参数 schema 补全

## 2. 第一批 — LLM/HTTP 类工具（M1）

- [ ] 2.1 `llmtools/brainstorm.go` 与 `llmtools/reason.go`：独立 eino ChatModel 补全，完整结果入 `metadata`，无 `sources`
- [ ] 2.2 `llmtools/websearch.go`：搜索 provider 适配，`sources` 为 `{type:"web", url, title}`
- [ ] 2.3 `llmtools/papersearch.go`：arXiv 查询、markdown 渲染（摘要截 400 字符）、限流/网络错误降级（`metadata.error=true` 不抛出）、空结果提示
- [ ] 2.4 `webtools/webfetch.go`：scheme 白名单、私网/回环/链路本地拒绝（含重定向后与 DialContext 层复检）、4MB 上限、`max_chars` 截断标记
- [ ] 2.5 `webtools/github.go`：`gh` CLI 只读词汇表（`view`/`list`、`api` GET）、16000 字符截断、`gh` 缺失友好降级
- [ ] 2.6 `mediagen/imagegen.go` 与 `mediagen/videogen.go`：n clamp、独立 `imagegen_*/videogen_*` 子目录、artifact sources、模型未配置降级、videogen `tool_log` 进度尽力上报

## 3. 第二批 — 本地状态类工具（M2）

- [ ] 3.1 `contexttools/readsource.go`：`source_index` 注入读取、id 前缀语义、未命中列出全部合法 id
- [ ] 3.2 `contexttools/readskill.go` 与 `contexttools/loadtools.go`：对接 skills-persona 服务接口（穿越/截断语义在服务侧）、loader 缺失降级、`loaded/already_loaded/unknown` 三组报告
- [ ] 3.3 `memorytools/readmemory.go` 与 `memorytools/writememory.go`：对接 memory 存储接口（L3 拼接、trace-first 写 preferences）、空记忆固定提示、写拒绝返回 `success=false`
- [ ] 3.4 `notebooktools/listnotebook.go` 与 `notebooktools/writenote.go`：索引/钻取两模式与上限、`conversation_history` 渲染对话稿、长度截断（title 200 / note 4000 / content 200000）、`ok=false` 错误约定
- [ ] 3.5 `controltools/cron.go`：action 归一（`add`→`schedule` 等）、owner 注入校验、`_cron_in_context` 拒绝再调度、三种 schedule 参数互斥校验
- [ ] 3.6 `controltools/askuser.go`：`questions` 校验（1–4）、遗留入参归一、`pause_for_user` 负载、校验失败不暂停

## 4. 第三批 — Sidecar 转发类工具（M2）

- [ ] 4.1 `sandboxtools/rag.go`：`query`/`kb_name` 必填校验（无默认 KB 回退）、KB 归属解析、Sidecar RAG 转发、`metadata` 透传完整结果
- [ ] 4.2 `sandboxtools/exec.go`：deny-pattern 预检、timeout clamp（1–300）、Sidecar 沙箱提交、stdout/stderr 渲染与 artifact 清单、`success=exit_code==0`
- [ ] 4.3 `sandboxtools/codeexecution.go`：语言别名归一、独立 run 子目录、语言编译模板（`python3` / `cc -O2` / `c++ -std=c++17 -O2`）、artifacts 排除规则
- [ ] 4.4 沙箱后端未配置时两工具不挂载的 gate 数据（供 agent-loop 消费）

## 5. 测试与验收（spec Scenario 落测试 + 协议对等）

- [ ] 5.1 将本 spec 全部 18 个 Scenario 落为 Go 单测/集成测试（mock LLM + mock Sidecar），逐条对应：失败结果进对话、注入不可伪造、开关/alias、coming_soon、arXiv 降级、内网重定向拒绝、gh 缺失、imagegen 卡片、未知 source_id、部分装载、write_memory 落盘链、write_note 对话稿、cron 跨会话隔离、ask_user 暂停恢复、rag 缺 kb_name、并发 code_execution、危险命令拦截
- [ ] 5.2 `/api/v1/tools` 目录条目（含 `geogebra_analysis` 的 `coming_soon=true, enabled=false`）经 OpenAPI golden spec contract test 对齐（acceptance.md M2 矩阵「Settings/System/Personas/Tools API」项；D-004 设置页占位登记手工验证）
- [ ] 5.3 第一批工具按 acceptance.md M1 矩阵「第一批 tools」项跑单测/集成（`brainstorm/reason/web_search/paper_search/web_fetch/github` + Sidecar 的 `rag/exec/code_execution`）
- [ ] 5.4 WS fixtures 回放：chat turn 中 tool_call/tool_result/sources 事件序列与基线 fixtures 对齐（acceptance.md 3.1 WebSocket 对等）
