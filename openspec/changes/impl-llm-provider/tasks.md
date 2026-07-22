# Tasks: impl-llm-provider

## 1. 注册表与匹配

- [ ] 1.1 实现 `internal/llm/registry/spec.go`：`ProviderSpec` 元数据结构与基线全集内置表（direct/gateway/standard/local/oauth，注册顺序即优先级，含 `env_key`/`default_api_base`/`strip_model_prefix`/thinking_style/`model_overrides`/`reasoning_model_patterns`）
- [ ] 1.2 实现 `CanonicalProviderName`（snake_case 归一 + 别名表 + 空输入返回空）
- [ ] 1.3 实现三条匹配路径：`FindByName`、`FindByModel`（前缀等值 → keyword 子串、`-`/`_` 归一）、`FindGateway`（key 前缀与 base 关键词探测）与 `StripProviderPrefix`

## 2. 归一层与请求兼容

- [ ] 2.1 实现 `LLMResponse`/`ToolCall`/`Usage` 类型与 `Accumulate`（chunk 聚合：内容/reasoning 增量、按 index 归组 tool_call、finish_reason 与最后一帧 usage 捕获）
- [ ] 2.2 实现 json repair：malformed arguments 容错解析、失败回落空 object
- [ ] 2.3 实现消息清洗 `sanitize.go`：键白名单、空 content 替换（assistant+tool_calls → null / `"(empty)"`）、多模态空 text 分片剔除、tool_call id 归一且跨消息一致
- [ ] 2.4 实现重试包装器：瞬态特征集 + 1s/2s/4s 退避 + 超次返回 `finish_reason="error"`；非瞬态图片降级重试一次

## 3. OpenAI-compatible 内核

- [ ] 3.1 基于 eino-ext openai 组件实现 `openaicompat` ChatModel：SSE 流式、`stream_options.include_usage` 按 spec 附带、内容与 reasoning（`reasoning_content`/`reasoning`）增量透出
- [ ] 3.2 实现请求侧参数注入：`max_tokens`/`max_completion_tokens` 按模型切换、`reasoning_effort` 与 provider thinking 参数、`model_overrides` 套用、模型名前缀剥离
- [ ] 3.3 实现 `response_format` 不支持时的降级重试一次与 (binding, model) 进程内记忆

## 4. Anthropic 内核与适配

- [ ] 4.1 基于 eino-ext claude 组件实现 Anthropic Messages 流式内核（content_block/input_json_delta 解析）
- [ ] 4.2 实现 OpenAI 形状适配层：内容增量实时透出、tool_call 流后带 index 补发、收尾帧（finish_reason + usage）、对消费方透明
- [ ] 4.3 实现 prompt caching 标记（`supports_prompt_caching` 时按基线策略在 system/tools 放置 cache_control）

## 5. UsageTracker 与 factory

- [ ] 5.1 实现 `UsageTracker`：`AddFromResponse`/`AddEstimated`（chars/3.5）、流式只记账一次（最后一帧 usage、无帧回落估算、记账失败不影响流）、`Summary()` 零调用返回空否则五字段、每千 token 计价
- [ ] 5.2 实现 `factory.go` 按 backend 分派；`openai_codex`/`github_copilot` 返回 `ErrProviderDeferred` 明确后置错误（不静默回落）

## 6. mock/replay provider

- [ ] 6.1 定义 fixture 格式（`contracts/fixtures/llm/*.jsonl`：内容/reasoning 增量、分片 tool_call、usage 帧、finish 帧）与加载器
- [ ] 6.2 实现 `replay` ChatModel：逐 chunk 保序回放进 `StreamReader`、走与真实内核相同的 `Accumulate` 归一路径、`AssertExhausted` 暴露未消费/多余 chunk；确认生产分派表不含该 backend

## 7. 测试与验收

- [ ] 7.1 把 `llm-provider` spec 的全部 Scenario（11 个 Requirement / 14 个 Scenario）逐条落为 Go 测试（注册表全集对照、别名/探测、分片归组、降级重试、退避、usage 单次记账、OAuth 后置错误等）
- [ ] 7.2 契约验收：replay provider 回放录制的 completion/tool_call chunk 序列，归一结果与真实内核路径逐字段一致（对应 acceptance.md §4 M0 矩阵「mock LLM provider」与 §2「LLM 确定性」）
- [ ] 7.3 协议对等验收：OpenAI-compat 与 Anthropic 内核对同一工具调用场景产出结构一致的 chunk 序列（内容增量 → tool_call → 收尾帧）；真实 provider 连通性 e2e 各跑通一例（对应 acceptance.md §4 M1 矩阵「LLM provider」）
