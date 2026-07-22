# Tasks: impl-sidecar-contract

## 1. 契约定义与冻结

- [ ] 1.1 定义全部端点的请求/响应 schema（`sidecar/schemas.py`，pydantic `extra="forbid"`，全字段显式类型，opaque 字段显式声明并注明消费方）
- [ ] 1.2 定义六类稳定错误码的统一错误信封（`{"error":{"code","message","details"}}`）与 FastAPI exception handlers
- [ ] 1.3 导出并冻结 `contracts/sidecar/openapi.json`；为每端点编写请求/响应 golden fixtures（`contracts/sidecar/fixtures/`）

## 2. Python Sidecar 薄壳

- [ ] 2.1 搭建 `deeptutor-sidecar` 包骨架（pyproject 引用基线 v1.5.2 为只读依赖、FastAPI app、与 Go 共享 `data/` 根的启动参数）
- [ ] 2.2 实现 `GET /health`：整体 ready、版本号、RAG 各 provider/解析引擎/Manim/沙箱隔离级别的子能力可用性聚合
- [ ] 2.3 实现 `POST /rag/search`：包装基线 RAG service，KB 绑定 provider 解析（未知回落 `llamaindex`）、`answer`/`content` 双键归一、`provider` 覆盖、`error_type`/`needs_reindex`
- [ ] 2.4 实现 `POST /rag/index`（initialize/add_documents、受理返回 `task_id`、不支持增量回落全量）与 KB 版本模型（llamaindex 按 embedding 签名、其余合成 provider 签名）
- [ ] 2.5 实现进度桥接 `progress.py`：ProgressTracker 形状帧（`stage/message/percent/current/total/timestamp/task_id`）经 SSE 推送，含最近快照先发
- [ ] 2.6 实现 `POST /parse/document`（full 档）：engine 无关 IR、`(source_hash, parser_signature)` content-addressed 缓存、引擎就绪门禁、空产物报错清理、不支持后缀拒绝
- [ ] 2.7 实现 `POST /parse/extract`（light 档）：bytes-in/text-out、Office 族 + 文本族多编码回退、四项限额每次调用时按 `system.json`（含 env override）解析、超限带文件名报错
- [ ] 2.8 实现 `POST /render/manim`：工作目录布局（source/artifacts/media/meta）、video 场景名提取、image 模式 YON_IMAGE 锚块校验、quality 映射（未知回落 `-qm`）、`/api/outputs/` URL 产物、stdout/stderr 逐行进度（raw/summary 层）、非零退出码报渲染错误
- [ ] 2.9 实现 `POST /exec`：`ExecRequest`/`ExecResult` 对应、隔离级别报告与 `off` 拒绝、每用户配额（并发 + 每分钟次数）、`max_output_chars` head+tail 截断附说明与退出码

## 3. Go typed client

- [ ] 3.1 用 oapi-codegen 从冻结 `openapi.json` 生成 `internal/sidecar/gen/`（types + client），CI 校验重新生成零 diff
- [ ] 3.2 实现 `internal/sidecar/client.go` 薄封装：按端点超时（health/search 短超时、index/parse/render 受理制、exec 按 `timeout_s`+余量）、`APIError` 错误码映射、`ErrSidecarUnavailable` 快速失败不重试
- [ ] 3.3 实现健康探测与能力缓存：启动时 + 调用失败后按需 + 周期性短探测，失败只标记不可用、不阻塞 Go 主进程启动
- [ ] 3.4 实现 `StreamProgress` SSE 消费（含快照帧、终态结束、`task_id` 过滤参数）

## 4. progress WS 转发驱动

- [ ] 4.1 实现 `/api/v1/knowledge/{kb_name}/progress/ws` 转发 handler（gin 内 gorilla/websocket upgrade、`{"type":"progress","data":{...}}` 外层封装）
- [ ] 4.2 实现就绪 KB 快速路径（一帧 `stage=completed, percent=100` 后关闭）、`task_id` 过滤、终态结束推送

## 5. 测试与验收

- [ ] 5.1 把 `sidecar-contract` spec 的全部 Scenario（11 个 Requirement / 17 个 Scenario）逐条落为自动化测试（Sidecar 端 pytest contract test + Go 端集成测试，golden fixtures 双向校验）
- [ ] 5.2 契约验收：`RAGSearch/RAGIndex/ParseDocument/RenderManim/ExecCode` typed API 全部可调用、未知字段/缺必填/枚举越界被拒（对应 acceptance.md §4 M0 矩阵「Sidecar 骨架」）
- [ ] 5.3 降级验收：kill sidecar 后纯聊天不受影响、`rag` 工具在超时内返回明确不可用错误不挂起（对应 acceptance.md §3.4「Sidecar 降级」）
- [ ] 5.4 数据兼容抽查：`data/knowledge_bases/` 基线快照经 Sidecar 可检索、`data/parse_cache/` 缓存命中不重复解析（对应 acceptance.md §3.2 B 类）
