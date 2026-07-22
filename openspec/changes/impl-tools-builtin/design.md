# Design: impl-tools-builtin

## Context

基线 Python 中每个工具是 `BaseTool` 子类（`deeptutor/core/tool_protocol.py`），由 `deeptutor/tools/builtin/__init__.py` 汇总注册；tool result 是 dict 约定（`content`/`sources`/`metadata`/`success`/`pause_for_user`）。Go 版技术约束：agent loop 基于 cloudwego/eino 的 tool-calling 循环，工具必须实现为 eino tool 组件（`tool.InvokableTool`，schema 由 `schema.ToolInfo` 描述）。难点在于 eino 的 `InvokableRun` 只返回一个 string 给 LLM，而本模块要求 `sources`/`metadata`/`pause_for_user` 等旁路信息流向 StreamEvent 与轮次控制，且 `success=false` 不得中断轮次。

## Goals / Non-Goals

**Goals:**

- 与基线逐工具行为对等的 Go 实现（三批共 21 个工具 + alias 表 + coming_soon 占位）。
- 统一 `ToolResult` 协议与 eino 适配层：LLM 看到的 schema/消息与基线一致，旁路信息（sources、pause）完整送达 runtime。
- 服务端注入参数与 LLM 参数硬隔离。

**Non-Goals:**

- 工具挂载时机与 context gate（agent-loop spec）；`/api/v1/tools` REST handler 本体（settings-api change，本 change 提供目录数据源）。
- memory/skills/notebook/cron 的存储侧语义（对应各自 change，本 change 只消费其服务接口）。
- `geogebra_analysis` 执行管线（D-004，不迁移）。

## Decisions

### D1. Go 包结构

```
internal/tools/
├── protocol.go        # ToolResult / Source / Injected / BuiltinTool 接口
├── registry.go        # 目录（canonical 顺序）、集合划分、alias 表、COMING_SOON
├── einoadapter/
│   └── adapter.go     # BuiltinTool → eino tool.InvokableTool 适配
├── llmtools/          # 第一批：brainstorm.go reason.go websearch.go papersearch.go
├── webtools/          # 第一批：webfetch.go github.go
├── mediagen/          # 第一批：imagegen.go videogen.go
├── contexttools/      # 第二批：readsource.go readskill.go loadtools.go
├── memorytools/       # 第二批：readmemory.go writememory.go
├── notebooktools/     # 第二批：listnotebook.go writenote.go
├── controltools/      # 第二批：cron.go askuser.go
└── sandboxtools/      # 第三批：rag.go exec.go codeexecution.go
```

### D2. 工具协议与 eino 适配（关键接口签名）

```go
// internal/tools/protocol.go
package tools

type Source map[string]any // {type:"web"|"paper"|"github"|"artifact"|"rag", ...} 字段随 type

type ToolResult struct {
    Content      string         // role=tool 消息体
    Sources      []Source       // 经 stream.sources 事件下发
    Metadata     map[string]any // 自由负载
    Success      bool           // false 不终止轮次
    PauseForUser map[string]any // 仅 ask_user：结构化问题负载
}

// Injected：pipeline 注入的服务端参数，绝不进 LLM schema
type Injected struct {
    WorkspaceDir        string
    SourceIndex         map[string]string // read_source
    ConversationHistory []schema.Message  // write_note
    CurrentUserMessage  string
    CronOwner           *cron.Owner       // cron
    CronInContext       bool              // _cron_in_context
    ToolLoader          DeferredToolLoader // load_tools
    SandboxMounts       []sidecar.Mount   // exec/code_execution
    EventSink           stream.Sink        // tool_log 进度（videogen）
    UserCtx             auth.UserContext   // 多用户路径解析
}

type BuiltinTool interface {
    Info(ctx context.Context) (*schema.ToolInfo, error)               // eino ToolInfo
    Execute(ctx context.Context, args map[string]any, inj Injected) (*ToolResult, error)
}
```

```go
// internal/tools/einoadapter/adapter.go
// Adapter 实现 eino 的 tool.InvokableTool：
//   InvokableRun(ctx, argumentsInJSON, opts...) (string, error)
type Adapter struct {
    impl    tools.BuiltinTool
    presets map[string]any // alias 预置 kwargs（如 mode=hybrid）
}

func (a *Adapter) Info(ctx context.Context) (*schema.ToolInfo, error)
func (a *Adapter) InvokableRun(ctx context.Context, argsJSON string, _ ...tool.Option) (string, error)
```

- `InvokableRun` 语义：反序列化 args → **剥除全部 `_` 前缀键与注入保留键**（防 LLM 伪造）→ 合并 alias presets → 从 `ctx` 取 per-turn `Injected`（`tools.InjectionFromContext(ctx)`，由 agent-loop 在每次 tool 调用前放入）→ 调用 `Execute`。
- **`success=false` 不返回 Go error**：`InvokableRun` 恒返回 `ToolResult.Content` 作为 string（eino 收到 error 会中断循环，违反本 spec）。只有 panic/编程错误才返回 error。
- 完整 `ToolResult` 经 `Injected.EventSink` 旁路上报：adapter 在返回前把 `{tool_call_id → ToolResult}` 写入 per-turn result registry，agent-loop 据此发 `stream.sources` 事件、检测 `PauseForUser` 触发 `pending_user_input` 暂停（恢复语义在 agent-loop spec）。
- `ToolInfo.ParamsOneOf` 由各工具以 `schema.NewParamsOneOfByParams` 构造；registry 层统一后处理：`type=array` 无 `items` 时补 `{"type":"string"}`。

### D3. 目录、集合与 alias（registry.go）

```go
var ToggleableTools = []string{"brainstorm", "web_search", "paper_search", "reason",
    "geogebra_analysis", "imagegen", "videogen"} // canonical 顺序
var AutoMountTools = []string{"rag", "code_execution", "read_source", "read_memory",
    "write_memory", "read_skill", "list_notebook", "write_note", "web_fetch",
    "github", "exec", "load_tools", "cron", "ask_user"}
var ComingSoonTools = map[string]bool{"geogebra_analysis": true} // D-004

var Aliases = map[string]AliasSpec{
    "rag_hybrid":   {Target: "rag", Presets: map[string]any{"mode": "hybrid"}},
    "rag_naive":    {Target: "rag", Presets: map[string]any{"mode": "naive"}},
    "rag_search":   {Target: "rag", Presets: map[string]any{"mode": "hybrid"}},
    "code_execute": {Target: "code_execution"}, "run_code": {Target: "code_execution"},
}

type Registry struct{ ... }
func (r *Registry) Resolve(name string) (tools.BuiltinTool, map[string]any, bool) // alias 归一
func (r *Registry) CatalogEntries() []CatalogEntry // 供 /api/v1/tools（settings-api 消费）
```

`geogebra_analysis` 只出现在 `CatalogEntries()`（`coming_soon=true, enabled=false`），`Resolve` 与运行时挂载均不可见（D-004，acceptance.md 第 6 节）。

### D4. 与 Python 基线文件映射

| Python 基线 | Go 落位 |
| --- | --- |
| `deeptutor/core/tool_protocol.py`（`BaseTool`/`ToolDefinition`） | `internal/tools/protocol.go` + `einoadapter/adapter.go` |
| `deeptutor/tools/builtin/__init__.py`（目录/别名/COMING_SOON） | `internal/tools/registry.go` |
| `deeptutor/tools/brainstorm.py`、`reason.py` | `internal/tools/llmtools/{brainstorm,reason}.go`（经 eino `model.BaseChatModel` 独立补全） |
| `deeptutor/tools/web_search.py`、`paper_search_tool.py` | `internal/tools/llmtools/{websearch,papersearch}.go` |
| `deeptutor/tools/web_fetch.py`、`github_query.py` | `internal/tools/webtools/{webfetch,github}.go` |
| `deeptutor/tools/media_gen_tool.py` | `internal/tools/mediagen/{imagegen,videogen}.go` |
| `deeptutor/tools/file_tools.py`（read_source）、skills 工具 | `internal/tools/contexttools/` |
| `deeptutor/tools/{list_notebook,write_note}.py` | `internal/tools/notebooktools/` |
| `deeptutor/tools/cron_tool.py`、`ask_user.py` | `internal/tools/controltools/` |
| `deeptutor/tools/{rag_tool,exec_tool}.py` | `internal/tools/sandboxtools/`（经 `internal/sidecar` typed client） |

### D5. 逐工具技术落位要点

- **brainstorm/reason**：持有独立 eino `ChatModel`（reason 用 catalog 的深推理 profile），一次非流式补全；完整结果对象入 `Metadata`，无 `Sources`。
- **web_fetch 安全链**：自定义 `http.Client`，`CheckRedirect` 中对每一跳的 `net.LookupIP` 结果做 `ip.IsPrivate()/IsLoopback()/IsLinkLocalUnicast()` 拒绝；`io.LimitReader` 4MB 硬上限；超 `max_chars` 截断加 truncated 标记。
- **github**：`exec.LookPath("gh")` 缺失 → `Success=false` 友好降级；命令词汇表白名单 `view`/`list`、`api` 恒 GET；输出截 16000 字符。
- **imagegen/videogen**：写入 `Injected.WorkspaceDir` 下 `imagegen_*/videogen_*` 独立子目录；`Sources` 为 artifact 行；videogen 经 `Injected.EventSink` 发 `tool_log` 进度，失败仅记日志。
- **ask_user**：校验 1–4 个 `questions`（遗留顶层 `{question, options}` 归一为单元素、不进 schema）；合法则 `PauseForUser` 携带问题负载、`Content` 为占位文本；校验失败 `Success=false` 不暂停。
- **cron**：`Injected.CronOwner` 缺失 → "当前上下文不可调度"；`CronInContext=true` 时拒绝 schedule；持久化/执行调 `internal/cron` service（cron-events change）。
- **exec/code_execution**：deny-pattern 正则预检（基线列表照搬）→ `internal/sidecar.ExecCode` 提交；`code_execution` 每次调用 `os.MkdirTemp(workspace, "run-*")` 独立子目录，天然并发安全；artifacts 采集排除源文件/`prog`/`stdin.txt`。
- **rag**：`kb_name` 缺失即参数错误（不做默认 KB 回退）；归属解析复用 knowledge 模块访问控制；转发 `internal/sidecar.RAGSearch`。

### D6. 备选方案取舍

- **旁路信息传递**：备选「把 ToolResult 整体 JSON 作为 InvokableRun 返回值、由 loop 再拆」——被否，会改变 LLM 看到的 `role=tool` 消息体，破坏与基线的提示对等。选用 ctx 注入 + result registry 旁路。
- **注入参数传递**：备选「拼进 argsJSON 由工具自己剥离」——被否，存在 LLM 同名覆盖竞态；选用 `Injected` 结构体经 context 传递，schema 永不包含。

## Risks / Trade-offs

- [eino `InvokableRun` 无结构化旁路通道] → 用 context 注入 + per-turn result registry；对 registry 的读写以 tool_call_id 为键、轮内互斥，防并发工具调用竞态。
- [web_fetch DNS rebinding：解析检查与实际连接分离] → 在 `DialContext` 层再校验实际连接 IP，而非仅 Lookup 时校验。
- [第二批工具依赖的服务（memory/skills/notebook/cron）未就绪] → 各工具依赖 Go interface（如 `memorytools.Store`），先以 stub 实现单测，合入需对应 change 已验收（ROADMAP 第 5 节并行约束）。
- [alias 语义漂移：presets 与 LLM 显式参数冲突] → 约定 LLM 参数不覆盖 presets（与基线一致，固定参数优先），单测覆盖。
