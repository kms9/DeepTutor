# knowledge Delta Specification

## ADDED Requirements

### Requirement: KB 目录布局与注册表兼容
每个 KB SHALL 位于 `data/knowledge_bases/<name>/`，包含 `raw/`（原始文档，唯一事实来源）、`metadata.json`（名称/时间戳/`rag_provider`/最近索引信息）、以及索引存储目录（默认 provider 使用平铺 `version-N/` 目录，每个 `(profile, model, dimension, base_url)` embedding 组合一个版本；其他 provider 使用各自存储目录）。KB 注册表 SHALL 持久化在 `kb_config.json`（`knowledge_bases` 映射：状态、progress、`rag_provider`、embedding 指纹、`needs_reindex`/`embedding_mismatch` 标记、`index_versions`、默认 KB 等），读写 SHALL 使用文件锁与原子写以兼容基线并发约定。KB 名 SHALL 经与基线一致的命名校验；磁盘上存在但未注册的 KB SHALL 被自动注册。

#### Scenario: 基线 KB 免迁移可用
- **WHEN** deeptutor-go 指向基线创建的 `data/knowledge_bases/` 启动并请求 `GET /api/v1/knowledge/list`
- **THEN** 全部既有 KB 连同统计/状态/provider 信息正确返回，磁盘布局未被改写

### Requirement: KB 创建、上传与删除
系统 SHALL 支持 `POST /create`（表单/上传驱动：登记 KB 为 `initializing` 状态、创建目录结构、把文档落入 `raw/`、经 Sidecar 发起索引，进度经 progress tracker 汇报）、`POST /{kb_name}/upload`（向既有 KB 增量添加文档并触发增量索引）、`DELETE /{kb_name}`（确认后整目录删除并从注册表移除）、`DELETE /{kb_name}/files/{filename}`（删除 raw 文档并同步索引）。上传 SHALL 校验支持的文件类型（`GET /supported-file-types` 公布），拒绝越界路径。长任务 SHALL 返回 task id，日志经 `GET /tasks/{task_id}/stream` SSE 观察。

#### Scenario: 创建 KB 全流程
- **WHEN** 用户以两个 PDF 调用 `POST /create`
- **THEN** KB 以 `initializing` 状态入注册表，文件写入 `raw/`
- **AND** 索引请求转发 Sidecar，阶段性进度可经 progress 端点观察，完成后状态转 ready

### Requirement: 索引/重建经 Sidecar 与版本化存储
索引、增量添加与 `POST /{kb_name}/reindex` SHALL 把 `raw/` 下受支持文件集合交给 Sidecar 的对应 provider pipeline 执行；Go 侧 SHALL NOT 内嵌任何 embedding/解析实现。默认 provider 的重建 SHALL 按当前激活 embedding signature 选择/创建 `version-N/` 目录，历史版本保留不动——切回曾索引过的 embedding 配置时 SHALL 直接复用既有版本而无需重建。`POST /{kb_name}/retry` SHALL 重试失败的初始化。provider 归属 SHALL 跟随 KB 注册的 `rag_provider`（reindex 不得把非默认 provider 的 KB 强拉回默认 pipeline）。

#### Scenario: 切换 embedding 后切回复用旧版本
- **WHEN** KB 曾以 embedding A 索引出 `version-1`，管理员切到 embedding B 重建出 `version-2`，随后切回 A
- **THEN** reconcile 发现 A 的匹配版本已存在，清除 `needs_reindex` 标记
- **AND** 检索直接使用 `version-1`，无需重新索引

### Requirement: embedding 兼容性 reconcile
系统 SHALL 在加载注册表时对每个自管 KB 对照磁盘 `version-N`（含 legacy 布局）与当前 embedding 指纹做 reconcile：找到匹配的 ready 版本 → 清除 `needs_reindex` 与 `embedding_mismatch`；存在 ready 版本但均不匹配、或存储的 `embedding_model`/`embedding_dim` 与当前不符 → 置两标记以驱动 UI 的 Re-index CTA；无 ready 版本且无历史 embedding 记录的全新 KB SHALL NOT 被误标。连接型 KB（Obsidian/linked/LightRAG server/subagent）无 embedding 生命周期，SHALL 跳过 reconcile。`index_versions` 快照 SHALL 随 reconcile 刷新供 UI 展示。

#### Scenario: 更换 embedding 触发重建提示
- **WHEN** 管理员应用了与全部既有版本都不匹配的新 embedding 配置
- **THEN** 每个受影响 KB 的 `needs_reindex=true` 且 `embedding_mismatch=true`
- **AND** 连接型 KB 不受影响

### Requirement: 进度追踪与 WebSocket 推送
初始化/索引进度 SHALL 由 progress tracker 持久化（stage/message/percent/current/total/timestamp/task_id），可经 `GET /{kb_name}/progress` 查询、`POST /{kb_name}/progress/clear` 清理卡死状态。`WS /{kb_name}/progress/ws` SHALL 鉴权后接入 broadcaster 实时推送进度帧；无活动任务（progress 缺失/终态/KB 已 ready）时 SHALL 走快速路径：发送当前状态后立即关闭，避免客户端无限轮询。

#### Scenario: 无活动任务快速关闭
- **WHEN** 客户端对一个已 ready 的 KB 打开 progress WebSocket
- **THEN** 服务端发送当前（终态）进度后主动关闭连接

### Requirement: 连接型 KB（Obsidian / 本地文件夹 / LightRAG server）
系统 SHALL 支持把外部资源注册为连接型 KB（注册表打连接标记，不管理 embedding 生命周期）：`POST /connect-obsidian`（Obsidian vault 路径）、`POST /probe-folder` + `POST /connect-folder`（本地文件夹探测与连接，路径 SHALL 经允许列表校验防任意目录挂载）、`POST /probe-lightrag-server` + `POST /connect-lightrag-server`（远端 LightRAG server 探活与连接）。连接型 KB 的删除 SHALL 只解除注册，SHALL NOT 删除外部原始数据。

#### Scenario: 连接本地文件夹
- **WHEN** 用户 probe 一个含受支持文档的文件夹并确认连接
- **THEN** 注册表新增连接型 KB 条目指向该外部根
- **AND** reconcile 与 reindex 流程对其跳过

### Requirement: linked folders 同步
系统 SHALL 支持向普通 KB 挂接受支持的本地文件夹作为持续来源：`POST /{kb_name}/link-folder`、`GET /{kb_name}/linked-folders`、`DELETE /{kb_name}/linked-folders/{folder_id}`、`POST /{kb_name}/sync-folder/{folder_id}`。同步 SHALL 先检测文件夹相对上次同步状态的增删改，再把变更文件送入增量索引，并更新该 folder 的 sync state（已同步文件清单）。

#### Scenario: 增量同步链接文件夹
- **WHEN** 链接文件夹新增 2 个文件后触发 sync-folder
- **THEN** 仅这 2 个文件进入增量索引
- **AND** folder sync state 更新为最新文件清单

### Requirement: RAG provider 配置面
系统 SHALL 提供 provider 配置路由组：`GET /health`、`GET /rag-providers`（可用 provider 及当前默认）、`PUT /rag-providers/{provider}/mode`、每 provider 的 `GET/PUT /rag-pipelines/{pageindex|llamaindex|graphrag|lightrag}/config`、`GET /rag-pipelines/{provider}/preflight`（就绪预检）、`GET /rag-pipelines/model-options`、`PUT /rag-pipelines/active-model`。配置持久化后 SHALL 即时对后续索引/检索生效；具体 pipeline 参数 schema 以 OpenAPI golden spec 为准。

#### Scenario: preflight 阻止不完整配置
- **WHEN** graphrag 缺少必需模型配置时请求其 preflight
- **THEN** 返回未就绪与缺失项说明，UI 可据此禁用"用该 provider 建库"

### Requirement: KB 配置与默认 KB
系统 SHALL 提供 `GET /configs`（全部 KB 配置）、`GET/PUT /{kb_name}/config`（单 KB 配置读写）、`POST /configs/sync`（注册表与磁盘状态同步）、`GET /default` 与 `PUT /default/{kb_name}`（默认 KB 读写）。默认 KB 别名集合（`default`、`current`、`selected`、`默认` 等）SHALL 在解析 KB 名时归一到当前默认 KB。

#### Scenario: 默认 KB 别名解析
- **WHEN** 请求路径中的 `kb_name` 为 `default` 且默认 KB 为 `my-kb`
- **THEN** 操作作用于 `my-kb`

### Requirement: 文件与目录管理
系统 SHALL 提供 KB 内容管理组：`GET /list`（全部可见 KB 概览：统计/状态/进度/来源/只读标记）、`GET /{kb_name}`（单 KB 详情）、`GET /{kb_name}/files`（raw 文件树）、`POST /{kb_name}/folders`（建目录）、`POST /{kb_name}/files/move`（移动）、`GET /{kb_name}/files/{filename}`（原文件下载）、`GET /{kb_name}/file-preview-text/{filename}`（抽取文本预览，超长截断）。全部文件路径参数 SHALL 拒绝目录穿越。

#### Scenario: 预览超长文档
- **WHEN** 请求一个超出预览字符上限的 PDF 的 file-preview-text
- **THEN** 返回截断后的抽取文本并标记 truncated

### Requirement: 多用户可见性与写保护
KB 路由 SHALL 经归属解析执行访问控制：读操作对可见 KB（自有 + 被分配）开放，写操作 SHALL 校验可写归属（他人 KB 返回禁止）；`GET /list` SHALL 合并自有与被分配 KB 并标记 `assigned`/`read_only`。`rag` 工具的检索请求 SHALL 复用同一解析逻辑（见 tools-builtin spec）。

#### Scenario: 被分配 KB 只读
- **WHEN** 非 admin 用户对一个 admin 分配给他的 KB 调用 upload
- **THEN** 请求被拒绝（只读归属）
- **AND** 对同一 KB 的 `rag` 检索正常工作
