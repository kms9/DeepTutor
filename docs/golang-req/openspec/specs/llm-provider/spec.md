# LLM Provider Specification

## Purpose
本模块定义 deeptutor-go 的 LLM 供应商层：provider 注册表（元数据、别名、匹配与探测）、两类流式协议内核（OpenAI-compatible 与 Anthropic，统一归一为 OpenAI chat.completions 形状）、usage/cost 统计（UsageTracker 语义），以及用于契约测试的 mock/replay provider。OAuth 类 provider（`openai_codex`、`github_copilot`）标记为后置 requirement，M2 不实现。

- 参考实现（基线）：`deeptutor/services/provider_registry.py`、`deeptutor/services/llm/provider_core/`（`base.py`、`openai_compat_provider.py`、`anthropic_provider.py`）、`deeptutor/services/llm/executors.py`、`deeptutor/services/llm/client.py`、`deeptutor/core/agentic/client.py`、`deeptutor/core/agentic/usage.py`
- 依赖 spec / 里程碑：依赖：foundation-config（settings/模型配置读取）；里程碑：M2

## Requirements

### Requirement: provider 注册表与 backend 分类
系统 SHALL 内置与基线一致的 provider 注册表，注册顺序即匹配优先级（gateway 优先）。backend 枚举为 `openai_compat`、`anthropic`、`azure_openai`、`openai_codex`、`github_copilot`。基线 provider 名单 SHALL 至少包括：

- direct：`custom`、`custom_anthropic`、`azure_openai`、`ovms`
- gateway：`openrouter`、`aihubmix`、`siliconflow`、`volcengine`、`volcengine_coding_plan`、`byteplus`、`byteplus_coding_plan`、`nvidia_nim`
- standard：`anthropic`、`openai`、`deepseek`、`gemini`、`zhipu`、`dashscope`、`moonshot`、`minimax`、`minimax_anthropic`、`mistral`、`stepfun`、`xiaomi_mimo`、`groq`、`qianfan`
- local：`vllm`、`ollama`、`lm_studio`、`llama_cpp`、`lemonade`
- oauth（后置）：`openai_codex`、`github_copilot`

每个条目 SHALL 携带基线同义的元数据：`env_key`、`default_api_base`、`strip_model_prefix`、`supports_max_completion_tokens`、`supports_prompt_caching`、`supports_stream_options`、`thinking_style`（`thinking_type`/`enable_thinking`/`reasoning_split`）、`model_overrides`（如 Moonshot 对 `kimi-k2.5`/`kimi-k2.6` 强制 `temperature=1.0`）、`reasoning_model_patterns` 等。

#### Scenario: 注册表覆盖基线全集
- **WHEN** 枚举 Go 版注册表
- **THEN** 上述 provider 名与其 backend 分类和 Python 基线一一对应

### Requirement: provider 名归一化与别名
`canonical_provider_name` SHALL 把输入转 snake_case（`-` 视为 `_`）后套用别名表，别名 SHALL 至少包括：`azure`/`azure-openai`/`azureopenai`→`azure_openai`、`google`/`google_genai`→`gemini`、`claude`→`anthropic`、`openai_compatible`→`custom`、`anthropic_compatible`→`custom_anthropic`、`github-copilot`→`github_copilot`、`openai-codex`→`openai_codex`、`lm-studio`→`lm_studio`、Volc/BytePlus coding plan 的驼峰变体。空输入返回空。

#### Scenario: 别名解析
- **WHEN** 输入 `"Claude"` 或 `"azure-openai"`
- **THEN** 分别归一为 `anthropic` 与 `azure_openai`

### Requirement: provider 匹配与自动探测
系统 SHALL 复刻基线三条匹配路径：`find_by_name`（按 canonical 名精确匹配）；`find_by_model`（仅在 standard provider 中，先按 `provider/` 前缀等值匹配、再按 keyword 子串匹配，`-`/`_` 归一比较）；`find_gateway`（显式名命中 gateway/local 直接返回，否则按 `detect_by_key_prefix`（如 OpenRouter `sk-or-`、NVIDIA NIM `nvapi-`）与 `detect_by_base_keyword`（如 `openrouter`、`volces`、`11434`）探测）。`strip_provider_prefix` SHALL 仅在 spec 标记 `strip_model_prefix` 时剥掉模型名中的 `provider/` 前缀。

#### Scenario: 按 key 前缀识别 OpenRouter
- **WHEN** api_key 以 `sk-or-` 开头、未显式指定 provider
- **THEN** 探测结果为 `openrouter`

#### Scenario: 模型名前缀剥离
- **WHEN** provider 为 `aihubmix`（`strip_model_prefix=true`）、模型名为 `anthropic/claude-x`
- **THEN** 发往上游的模型名为 `claude-x`

### Requirement: 统一响应形状
两类协议内核 SHALL 归一输出同一响应结构（对应基线 `LLMResponse`）：`content`（可空）、`tool_calls`（`{id, name, arguments(object)}` 列表，可带 provider_specific_fields）、`finish_reason`（默认 `"stop"`）、`usage`（`prompt_tokens`/`completion_tokens`/`total_tokens` 整数表）、`reasoning_content`（可空）。工具调用的 `arguments` SHALL 是解析后的 JSON object；上游给出的 malformed JSON 字符串 SHALL 经容错修复（json repair）解析，仍失败时回落空 object 而非报错。

#### Scenario: 坏 JSON 参数被修复
- **WHEN** 上游 tool_call arguments 为缺尾括号的 JSON 字符串
- **THEN** 归一结果中 `arguments` 为修复后的 object，调用不失败

### Requirement: OpenAI-compatible 流式内核
系统 SHALL 实现 OpenAI chat.completions SSE 流式协议（`stream=true`，按 spec 支持时附 `stream_options.include_usage`）：逐 chunk 提取 `choices[0].delta.content` 作为内容增量、`delta.reasoning_content`（或 `delta.reasoning`）作为思考增量；tool_call chunk SHALL 按 `delta.tool_calls[].index` 归组累积——`id`/`function.name` 取首个非空值、`function.arguments` 字符串逐段拼接，流结束后统一解析为 object。`finish_reason` 与最后一帧 `usage` SHALL 从流中捕获。请求侧 SHALL 支持 `max_tokens` 与 `max_completion_tokens` 的按模型切换、`reasoning_effort` 及 provider 特定 thinking 参数注入、上游拒绝 `response_format` 时降级重试一次并记忆该 (binding, model) 后续不再携带。

#### Scenario: 分片 tool_call 归一
- **WHEN** 流中同一 `index=0` 的 tool_call 分三个 chunk 送达（先 id+name、后两段 arguments 片段）
- **THEN** 归一结果为一个完整 tool_call，arguments 为拼接后解析的 object

#### Scenario: response_format 不支持时自动降级
- **WHEN** 上游对带 `response_format` 的请求返回 400 且错误文案匹配不支持特征
- **THEN** 去掉 `response_format` 重试一次，且该 (binding, model) 被记忆为不支持

### Requirement: Anthropic 流式内核与 OpenAI 形状适配
系统 SHALL 实现 Anthropic Messages 流式协议，并经适配层向上暴露与 OpenAI-compatible 内核一致的 chunk 序列（对应基线 `_ProviderOpenAIAdapter`/`_ProviderOpenAIStream`）：内容增量实时以 `delta.content` chunk 透出；工具调用在流结束后以带 `index` 的 tool_call chunk 依次补发；最后 SHALL 发出一帧带 `finish_reason` 与 `usage` 的收尾 chunk。适配层 SHALL 支持 prompt caching 标记（spec 的 `supports_prompt_caching` 为 true 时按基线策略在 system/tools 上放置 cache_control）。消费方（agentic loop）SHALL 无需分辨底层是哪类内核。

#### Scenario: Anthropic 流对消费方透明
- **WHEN** 同一 agentic loop 分别接 OpenAI-compatible 与 Anthropic 内核运行同一轮工具调用
- **THEN** 两者产出的 chunk 事件序列结构一致（内容增量 → tool_call → 收尾帧）

### Requirement: 消息清洗与请求兼容
请求发出前系统 SHALL 做基线同款清洗：仅保留 provider 安全的消息键（`role`/`content`/`tool_calls`/`tool_call_id`/`name`/`reasoning_content` 等白名单）；空字符串 content SHALL 替换（assistant 且带 tool_calls 时置 null，否则置 `"(empty)"`）；多模态 content 数组中空 text 分片剔除；tool_call id SHALL 按 provider 约束归一且同一 id 的映射在消息间保持一致。

#### Scenario: 空 assistant 内容不触发上游 400
- **WHEN** 历史中存在 content 为空串且带 tool_calls 的 assistant 消息
- **THEN** 请求中该消息 content 为 null，上游不报错

### Requirement: 瞬态错误重试与图片降级
`chat`/`chat_stream` 的带重试变体 SHALL 复刻基线策略：错误文案命中瞬态特征集（`429`、`rate limit`、`500`、`502`、`503`、`504`、`overloaded`、`timeout`、`timed out`、`connection`、`server error`、`temporarily unavailable`）时按默认退避序列 1s/2s/4s 重试，超出次数返回错误响应（`finish_reason="error"`）而非抛异常；非瞬态错误且消息含图片、且调用方允许图片降级（模型不在已知 vision 白名单）时，SHALL 剥离图片重试一次。

#### Scenario: 限流自动退避
- **WHEN** 上游连续两次返回 429、第三次成功
- **THEN** 调用最终成功，期间按 1s、2s 退避

### Requirement: UsageTracker 用量与成本统计
系统 SHALL 提供每 turn 的 `UsageTracker`：精确路径 `add_from_response` 读取 provider 返回的 `usage`（prompt/completion/total tokens）累加并计 1 次调用；估算路径 `add_estimated` 以 `chars / 3.5` 折算 token。流式调用 SHALL 只记账一次：保留最后一帧 usage、流结束后提交，无 usage 帧时回落字符估算；记账失败 SHALL NOT 影响已完成的流。`summary()` 在零调用时返回空；否则返回 `{total_cost_usd, total_tokens, total_calls, prompt_tokens, completion_tokens}`，其中 `total_cost_usd` 按模型定价表以每千 token 计算。capability 收尾时 SHALL 把非空 summary 注入结果事件的 `metadata.cost_summary`。

#### Scenario: 多 usage 帧不重复计数
- **WHEN** 某 provider 在一条流中对多个 chunk 都附了 usage
- **THEN** 该流仅按最后一帧记账一次，`total_calls` 加 1

#### Scenario: cost_summary 注入
- **WHEN** 一个 chat turn 内发生了至少一次 LLM 调用
- **THEN** 该 turn 的 `result` 事件 `metadata.cost_summary` 含五个统计字段

### Requirement: mock/replay provider（契约测试用）
系统 SHALL 提供一个仅测试可用的 mock/replay provider：从录制文件（contracts/ 下 golden fixtures）回放预先录制的流式 chunk 序列（内容增量、reasoning 增量、分片 tool_call、usage 帧、finish 帧），经与真实内核相同的归一路径输出，使 agentic loop、turn runtime 与 WS 层的契约测试可离线、确定性运行。回放 SHALL 逐 chunk 保序；fixture 中未消费完或多余的 chunk SHALL 使断言可见。基线 Python 无独立 replay provider（测试用脚本化注入客户端），此为 Go 版新增测试设施，SHALL NOT 出现在生产 provider 选择路径中。

#### Scenario: 回放分片 tool_call fixture
- **WHEN** 契约测试加载一段录制了分片 tool_call 的 fixture 运行 agentic loop
- **THEN** loop 观察到的归一 tool_call 与真实 provider 路径逐字段一致，测试不需要网络

### Requirement: OAuth provider 后置
`openai_codex`（backend `openai_codex`，OAuth 到 `https://chatgpt.com/backend-api`）与 `github_copilot`（backend `github_copilot`，OAuth 到 `https://api.githubcopilot.com`）SHALL 保留注册表条目与元数据（供状态展示与配置校验），但其协议实现为后置 requirement（M2 之后单独排期）。M2 内选择这两个 provider 发起调用时 SHALL 返回明确的「未实现/后置」错误，SHALL NOT 静默回落其他 provider。

#### Scenario: 选择 Codex 得到明确错误
- **WHEN** 用户在 M2 版本把模型 provider 配为 `openai_codex` 并发消息
- **THEN** turn 以明确的未支持错误失败，错误文案指明该 provider 为后置项
