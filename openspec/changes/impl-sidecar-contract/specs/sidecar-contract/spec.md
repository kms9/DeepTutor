# Delta Spec: sidecar-contract

> 事实源：`docs/golang-req/openspec/specs/sidecar-contract/spec.md`（Requirement/Scenario 原样搬运，未增删语义）。

## ADDED Requirements

### Requirement: typed API 总则与 schema 校验
Sidecar 的每个端点 SHALL 定义显式的请求/响应 schema（字段名、类型、枚举、必填性），双方 SHALL 在边界做校验：请求含 schema 之外的未知字段时 SHALL 拒绝（明确的校验错误，而非静默忽略）；缺必填字段、枚举越界同样拒绝。Go 客户端与 Sidecar 服务端 SHALL NOT 以无类型 map（对应 Python 的 `dict[str, Any]`）直传业务载荷；确需开放结构的字段（如 RAG 结果的 `metadata`）SHALL 在 schema 中显式声明为 opaque JSON object 并注明消费方。契约的字段级对等由 contracts/ 下 golden fixtures 保证。

#### Scenario: 未知字段被拒绝
- **WHEN** Go 客户端向 `RAGSearch` 发送带多余字段 `foo` 的请求
- **THEN** Sidecar 返回 schema 校验错误并指明违规字段，不执行检索

### Requirement: 健康检查
Sidecar SHALL 暴露健康检查端点，返回至少：整体 ready 状态、版本号、各子能力可用性（RAG 各 provider 依赖是否就绪、解析引擎就绪状态、Manim 是否可用、沙箱 backend 的隔离级别）。Go 侧 SHALL 在启动时与调用失败后按需探测，探测 SHALL 使用短超时；健康检查失败 SHALL 只标记 Sidecar 能力不可用，SHALL NOT 阻塞 Go 主进程启动。

#### Scenario: 部分能力降级可见
- **WHEN** Sidecar 环境未安装 Manim 但 RAG 正常
- **THEN** 健康检查显示 `manim` 不可用、`rag` 可用，Go 侧据此仅禁用动画渲染

### Requirement: RAGSearch 检索契约
`RAGSearch` 请求 SHALL 至少含：`query`（必填）、`kb_name`（必填）与可选的检索参数。provider SHALL 由 Sidecar 按 KB 绑定解析（KB 创建时绑定，检索固定走该 pipeline），枚举为 `llamaindex`（默认）、`pageindex`、`graphrag`、`lightrag`、`lightrag-server`；未知/失效 provider 名 SHALL 回落 `llamaindex`。响应 SHALL 归一为：`query`、`answer` 与 `content`（互为别名，两者都出现）、`provider`（以 Sidecar 解析出的实际 provider 为准，覆盖 pipeline 自述值）、可选 `sources`、以及失败时的 `error_type` 与 `needs_reindex`（布尔）。检索过程中的状态事件（开始检索、检索完成字符数、错误）SHALL 经进度回报通道透出，供 Go 侧转为 StreamEvent。

#### Scenario: 检索结果字段归一
- **WHEN** 对绑定 `llamaindex` 的 KB 发起检索且 pipeline 只返回 `content`
- **THEN** 响应中 `answer` 与 `content` 同值、`provider="llamaindex"`、`query` 回填

#### Scenario: 索引失效提示重建
- **WHEN** KB 的索引版本与当前 embedding 签名不匹配
- **THEN** 响应带 `needs_reindex=true` 与 `error_type`，Go 侧向用户呈现明确的重建提示而非空答案

### Requirement: RAGIndex 索引契约与 KB 版本
`RAGIndex` SHALL 覆盖 `initialize`（新建索引）与 `add_documents`（增量入库）两个操作，请求含 `kb_name`、`file_paths` 及 provider 相关参数；不支持增量的 pipeline SHALL 回落全量 initialize。索引产物 SHALL 按基线的 KB 版本模型管理：`llamaindex` 的版本按当前 embedding 签名区分（embedding 配置变更后旧版本判定为过期）；`pageindex`/`graphrag`/`lightrag` 写各自的合成 provider 签名，SHALL NOT 因 embedding 配置变更被标记过期。索引为长任务：请求 SHALL 立即返回任务受理（含 `task_id`），进度经进度回报通道推送。

#### Scenario: embedding 变更只影响 llamaindex 版本
- **WHEN** 用户更换 embedding 模型后查看各 KB 状态
- **THEN** `llamaindex` KB 提示需要重建，`graphrag`/`lightrag` KB 不受影响

### Requirement: 索引进度回报供 progress WS 转发
Sidecar SHALL 对每个 KB 持续产出进度状态，形状与基线 ProgressTracker 对齐：`stage`（含终态 `completed`/`error`）、`message`、`percent`、`current`、`total`、`timestamp`（ISO 格式）、`task_id`。Go 侧 SHALL 据此驱动 `/{kb_name}/progress/ws`（基线路由 `/api/v1/knowledge/{kb_name}/progress/ws`）：消息外层为 `{"type":"progress","data":{...}}`；连接时无活跃任务（终态、或时间戳老于活跃窗口）且 KB 已就绪时 SHALL 立即发送一帧 `stage=completed, percent=100` 后关闭；携带 `task_id` 查询参数时 SHALL 只透传该任务的进度；进度进入终态后 SHALL 结束推送。

#### Scenario: 就绪 KB 的快速路径
- **WHEN** 前端对一个已就绪且无活跃索引任务的 KB 打开 progress WS
- **THEN** 收到一帧 `{"type":"progress","data":{"stage":"completed","percent":100,...}}` 后连接关闭，不进入轮询

#### Scenario: task_id 过滤
- **WHEN** WS 连接带 `task_id=A` 而 Sidecar 正在推送任务 B 的进度
- **THEN** 任务 B 的进度帧不透传给该连接

### Requirement: ParseDocument 之 KB 入库解析档
`ParseDocument`（full 档）SHALL 复刻基线 ParseService 语义：请求含 `source_path` 与可选 `engine`（缺省取 `document_parsing.json` 的 active engine），引擎枚举 `text_only`、`mineru`、`docling`、`markitdown`、`pymupdf4llm`。解析结果 SHALL 是 engine 无关的 IR（对应 `ParsedDocument`）：`markdown`（必有）、`blocks`（MinerU content_list 形状，可空）、`asset_dir`、`source_hash`、`parser_signature`、`engine`、`workdir`。解析 SHALL 走 content-addressed 缓存（key 为 `(source_hash, parser_signature)`，位于 `data/parse_cache/`）：命中直接返回缓存 IR；未命中时先过引擎就绪门（如本地模型未下载且未允许自动下载时 SHALL 以明确错误拒绝），解析产物为空时 SHALL 报错并清理失败工作目录。引擎不支持的文件后缀 SHALL 以明确错误拒绝。

#### Scenario: 缓存命中不重复解析
- **WHEN** 同一文件字节以同一引擎签名再次请求解析
- **THEN** 直接返回缓存 IR，引擎不被再次调用

#### Scenario: 模型未就绪被门禁
- **WHEN** active engine 为 `mineru` local 模式、模型未下载且 `allow_local_model_download=false`
- **THEN** 返回引擎未就绪的明确错误（含用户可读文案），不发生静默下载

### Requirement: ParseDocument 之附件轻量抽取档
附件抽取档（light 档，对应基线 `document_extractor`）SHALL 是 bytes-in/text-out：请求含文件名与文件字节（或 base64），不落盘解析；支持二进制 Office 族（`.pdf`/`.docx`/`.xlsx`/`.pptx`）与文本族（与 KB 文件路由的 TEXT_EXTENSIONS 同集合），文本族按多编码回退链解码。抽取 SHALL 施加限额并按当前 `system.json`（含 env override）在每次调用时解析：单文件字节上限（默认 20 MiB）、单消息总字节上限（默认 25 MiB）、单文档抽取字符上限（默认 200000）、单消息抽取字符总上限（默认 150000）。超限与不支持格式 SHALL 返回带文件名的用户可读错误。

#### Scenario: 限额随设置即时生效
- **WHEN** 用户把 `chat_attachment_max_chars_per_doc` 调小后发送下一条带附件消息
- **THEN** 新限额即时生效，无需重启

### Requirement: RenderManim 渲染契约
`RenderManim` 请求 SHALL 含：`turn_id`、`code`（Manim Python 源码）、`output_mode`（枚举 `video`/`image`）、`quality`（枚举 `low`/`medium`/`high`，映射 Manim `-ql`/`-qm`/`-qh`，未知值回落 `-qm`）。行为 SHALL 与基线一致：工作目录为 `workspace/chat/math_animator/<turn_id>/` 下的 `source`/`artifacts`/`media`/`meta`；video 模式从代码中提取首个 `Scene` 子类为场景名，缺失时报错；image 模式要求代码仅由 `### YON_IMAGE_n_START ###`/`### YON_IMAGE_n_END ###` 锚块组成（无锚块或有残余代码均报错），逐块渲染最后一帧 PNG。产物 SHALL 落入 `artifacts/` 并以 `/api/outputs/<相对 user_data_dir 路径>` 形式返回 URL（`{type, filename, url, content_type, label}`）。渲染过程的 stdout/stderr 行与阶段消息 SHALL 经进度回报通道逐行透出（区分 raw 与 summary 层）；Manim 退出码非零时 SHALL 返回带裁剪后错误输出的渲染错误。

#### Scenario: video 渲染成功
- **WHEN** 提交含一个 Scene 子类的代码、`output_mode=video, quality=high`
- **THEN** 返回一个 `type=video, content_type=video/mp4` 的 artifact，URL 以 `/api/outputs/` 开头且文件真实存在

#### Scenario: image 模式锚块校验
- **WHEN** image 模式代码在锚块之外还有残余语句
- **THEN** 渲染被拒绝，错误说明 image 模式只允许 YON_IMAGE 锚块

### Requirement: ExecCode 沙箱执行契约
`ExecCode`（承载 `exec` 与 `code_execution` 工具）请求 SHALL 对应基线 `ExecRequest`：`command`、`workdir`、`mounts`（`{host_path, sandbox_path, read_only}` 列表）、`env`、`limits`（`timeout_s` 默认 30、`memory_mb` 默认 512、`max_output_chars` 默认 10000、`cpu_seconds` 默认 30）。响应 SHALL 对应 `ExecResult`：`stdout`、`stderr`、`exit_code`、`timed_out`、`error`（仅沙箱自身故障时置位；命令失败以 `exit_code` 表达，SHALL NOT 视为调用错误）。Sidecar SHALL 报告隔离级别（`system`/`application`/`off`），`off` 时拒绝执行；SHALL 施加每用户配额（并发数与每分钟次数），超额以 `error` 返回。模型可见输出 SHALL 按 `max_output_chars` 做 head+tail 截断并附截断说明与退出码。

#### Scenario: 命令失败不等于调用失败
- **WHEN** 沙箱内命令退出码为 2
- **THEN** 响应 `exit_code=2`、`error=""`，Go 侧把输出正常回给模型

#### Scenario: 无可用沙箱时明确拒绝
- **WHEN** 隔离级别为 `off` 时请求执行
- **THEN** 响应 `error` 说明无可用沙箱 backend，不在宿主机裸跑命令

### Requirement: Go 客户端超时与降级
Go 侧 Sidecar 客户端 SHALL 按端点特性配置超时：健康检查与 `RAGSearch` 用短超时；`RAGIndex`、`ParseDocument`（full 档）、`RenderManim` 为长任务，SHALL 采用「受理 + 进度/轮询」模式而非单请求长阻塞。降级策略 SHALL 为：Sidecar 不可用（连接失败、健康检查失败、超时）时，chat 主链路（不依赖 Sidecar 的对话、工具）SHALL 完全不受影响；依赖 Sidecar 的功能（KB 检索/索引、文档解析、Manim、沙箱执行）SHALL 快速返回明确的「Sidecar 不可用」错误，错误 SHALL 经 StreamEvent `error` 或工具结果透出为用户可读文案。客户端 SHALL NOT 无限重试；恢复探测按健康检查节奏进行。

#### Scenario: Sidecar 宕机不影响纯聊天
- **WHEN** Sidecar 进程未启动且用户发起不带 KB 的普通聊天
- **THEN** 聊天正常流式返回，无 Sidecar 相关报错

#### Scenario: KB 检索快速失败
- **WHEN** Sidecar 宕机且 chat 内触发 `rag` 工具
- **THEN** 工具在客户端超时内返回明确的 Sidecar 不可用错误，turn 继续（模型可据此作答），不挂起

### Requirement: 错误归一
Sidecar SHALL 把内部异常归一为带稳定错误码的结构化错误：至少区分 `validation_error`（schema 校验失败）、`parser_error`（对应基线 ParserError，含引擎未就绪、格式不支持、空产物）、`render_error`（对应 ManimRenderError）、`rag_error`（含 `error_type`/`needs_reindex` 附加字段）、`sandbox_error`（沙箱自身故障/配额超限）、`unavailable`（能力未安装/未就绪）。每个错误 SHALL 携带用户可读 message；Go 侧 SHALL 按错误码映射到对应的 UI 呈现与重试策略，SHALL NOT 依赖解析 message 文本分支。

#### Scenario: 错误码驱动前端呈现
- **WHEN** 解析请求因引擎未就绪失败
- **THEN** 响应错误码为 `parser_error` 且 message 提示到 Settings → Document Parsing 处理，Go 侧不解析文本即可归类
