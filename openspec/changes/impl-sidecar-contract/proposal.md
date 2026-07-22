# Proposal: impl-sidecar-contract

## Why

RAG、文档解析、Manim 渲染与 Python 代码沙箱是 Python 生态强绑定域，按架构决策保留在 `deeptutor-sidecar`（Python, FastAPI），Go 主进程以强类型客户端调用；这条边界契约必须在 Wave 0 冻结（schema 校验、错误码、进度回报、降级语义），下游 tools/knowledge/visualize 模块才能并行推进。本模块是 ROADMAP 中 Wave 0（基础契约）/ M0 里程碑的成员（M0 交付「Sidecar 骨架：typed API 可调用、schema 校验拒绝未知字段」），行为规格见 `docs/golang-req/openspec/specs/sidecar-contract/spec.md`。

## What Changes

- 定义 typed API 总则：每端点显式请求/响应 schema、双边校验、未知字段拒绝、禁止 `dict[str, Any]` 直传（开放结构字段显式声明为 opaque JSON object）；契约冻结于 `contracts/sidecar/`。
- 实现健康检查端点（整体 ready、版本、RAG/解析/Manim/沙箱各子能力可用性）与 Go 侧短超时探测。
- 实现 `RAGSearch` 检索契约（provider 按 KB 绑定解析、响应字段归一 `answer`/`content` 双键、`error_type`/`needs_reindex`、状态事件经进度通道）。
- 实现 `RAGIndex` 索引契约（initialize/add_documents、KB 版本模型：llamaindex 按 embedding 签名、其余 provider 合成签名、长任务受理 + `task_id`）。
- 实现索引进度回报与 Go 侧 `/{kb_name}/progress/ws` 转发（`{"type":"progress","data":{...}}` 外层、就绪快速路径、`task_id` 过滤、终态结束推送）。
- 实现 `ParseDocument` full 档（引擎枚举、engine 无关 IR、content-addressed 缓存、引擎就绪门禁、空产物报错）与 light 档附件抽取（bytes-in/text-out、多编码回退、限额按 `system.json` 实时解析）。
- 实现 `RenderManim` 渲染契约（video/image 两模式、quality 映射、工作目录布局、`/api/outputs/` URL 产物、stdout/stderr 逐行进度、非零退出码报错）。
- 实现 `ExecCode` 沙箱执行契约（`ExecRequest`/`ExecResult` 对应、隔离级别报告与 `off` 拒绝、每用户配额、head+tail 输出截断）。
- 实现 Go 客户端超时与降级（长任务「受理 + 进度/轮询」、Sidecar 不可用时 chat 主链路不受影响、依赖功能快速返回明确错误、不无限重试）。
- 实现错误归一（`validation_error`/`parser_error`/`render_error`/`rag_error`/`sandbox_error`/`unavailable` 稳定错误码 + 用户可读 message）。
- Python 侧剥离：新建 `deeptutor-sidecar` FastAPI 薄壳，以库形式复用基线 `deeptutor/services/rag`、`deeptutor/services/parsing`、`deeptutor/services/sandbox`、`deeptutor/agents/math_animator/renderer.py`、`deeptutor/utils/document_extractor.py`（基线仓库只读、不修改）。

## Capabilities

### New Capabilities

- `sidecar-contract`: Go 主进程与 Python Sidecar 之间的 typed API 契约——RAG 检索/索引、文档解析两档、Manim 渲染、沙箱执行、健康检查、进度回报、错误归一与降级语义。

### Modified Capabilities

（无）

## Impact

- 新建代码：`deeptutor-go/internal/sidecar/`（typed client + 生成代码）、`deeptutor-go/contracts/sidecar/`（冻结的 OpenAPI schema + golden fixtures）、新建 `deeptutor-sidecar` Python 包（FastAPI 薄壳）。
- 依赖的其他 change（按 ROADMAP 依赖图与模块 spec 声明）：`impl-foundation-config`（PathService/settings，附件限额与解析引擎配置）、`impl-foundation-stream`（进度事件转为 StreamEvent 的出口信封）。
- 被依赖：`impl-tools-builtin`（side → tools：rag/exec/code_execution 工具）、`impl-knowledge`（side → kb：索引/检索与 progress WS）、`impl-capability-visualize`（side → vis：Manim 渲染）。
- Python 侧剥离涉及：从基线 v1.5.2 复用 `services/rag/`（service.py、factory.py 及各 pipeline）、`services/parsing/`（service.py、types.py）、`services/sandbox/`（service.py、spec.py）、`agents/math_animator/renderer.py`、`utils/document_extractor.py`——以依赖引用方式装入 sidecar 进程，基线代码不做任何修改。
- 不修改 `web/` 前端；progress WS 的对外路由（`/api/v1/knowledge/{kb_name}/progress/ws`）消息面与基线一致。
