# Proposal: impl-llm-provider

## Why

LLM 供应商层是 agent loop 与全部 capability 的上游依赖：provider 注册表/匹配探测、OpenAI-compatible 与 Anthropic 两类流式内核的统一归一、usage/cost 统计、以及支撑离线契约测试的 mock/replay provider，必须先冻结才能让 M1 chat 主链路可验收。本模块是 ROADMAP 中 Wave 0（基础契约）的成员，行为规格见 `docs/golang-req/openspec/specs/llm-provider/spec.md`；按 ROADMAP §4，M0 交付「契约 + mock provider」、M1 交付 OpenAI-compat/Anthropic 实现（spec 内 usage/cost 与匹配全集的完成定义对应 M2 里程碑标注），OAuth 类 provider（`openai_codex`、`github_copilot`）为后置 requirement。

## What Changes

- 内置与基线一致的 provider 注册表（direct/gateway/standard/local/oauth 五类、注册顺序即匹配优先级），每条目携带 `env_key`、`default_api_base`、`strip_model_prefix`、thinking_style、`model_overrides` 等元数据。
- 实现 `canonical_provider_name` 归一化与别名表、三条匹配路径（`find_by_name`/`find_by_model`/`find_gateway`，含 key 前缀与 base 关键词探测）、`strip_provider_prefix`。
- 实现统一响应形状 `LLMResponse`（content/tool_calls/finish_reason/usage/reasoning_content），tool_call arguments 经 json repair 容错解析。
- 实现 OpenAI-compatible SSE 流式内核：内容/reasoning 增量、按 index 归组的分片 tool_call 累积、finish_reason 与 usage 捕获、`max_tokens`/`max_completion_tokens` 切换、thinking 参数注入、`response_format` 降级重试与记忆。
- 实现 Anthropic Messages 流式内核 + OpenAI 形状适配层（内容增量实时透出、tool_call 流后补发、收尾帧、prompt caching 标记），对消费方透明。
- 实现消息清洗与请求兼容（键白名单、空 content 替换、多模态空分片剔除、tool_call id 归一）。
- 实现瞬态错误重试（1s/2s/4s 退避、`finish_reason="error"` 收敛）与图片降级重试。
- 实现 `UsageTracker`（精确/估算双路径、流式只记账一次、`summary()` 五字段、`metadata.cost_summary` 注入约定）。
- 新增 mock/replay provider（自定义 eino ChatModel，从 `contracts/` fixtures 回放 chunk 序列，仅测试可用、不进生产选择路径）。
- OAuth provider 保留注册表条目，发起调用时返回明确的「未实现/后置」错误。

## Capabilities

### New Capabilities

- `llm-provider`: deeptutor-go 的 LLM 供应商层——provider 注册表与匹配探测、OpenAI-compat/Anthropic 流式内核统一归一、消息清洗与重试、UsageTracker、mock/replay provider。

### Modified Capabilities

（无）

## Impact

- 新建代码：`deeptutor-go/internal/llm/`（registry、两类内核、清洗/重试、usage、mock/replay）。
- 技术选型：`cloudwego/eino` + `eino-ext`——provider 适配实现为 eino `ChatModel`（openai/claude 组件），流式经 `StreamReader` 输出并由上层映射到 `StreamEvent`；mock/replay provider 同样是自定义 `ChatModel`。
- 依赖的其他 change（按 ROADMAP 依赖图与模块 spec 声明）：`impl-foundation-config`（settings/模型配置读取）。
- 被依赖：`impl-agent-loop`（llm → loop），进而支撑全部 capability 与 M1「chat 主链路」。
- 契约测试：mock/replay provider 使 agent loop、turn runtime、WS 层的契约测试可离线确定性运行（acceptance.md §2「LLM 确定性」）。
- 不修改 Python 基线与 `web/` 前端。
