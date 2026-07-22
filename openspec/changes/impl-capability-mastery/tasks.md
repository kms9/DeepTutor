# impl-capability-mastery — Tasks

## 1. learning 确定性引擎（可独立先行）

- [ ] 1.1 实现 `internal/learning/models.go`：Progress / PendingQuestion / RepetitionState / ErrorRecord / NextStep 等结构（JSON 字段名与基线 progress 文件一致）
- [ ] 1.2 实现 `ratcliff.go`：复刻 CPython `difflib.SequenceMatcher.ratio()`（b2j 索引、autojunk、最早最长平局规则）
- [ ] 1.3 实现 `grading.go`：`grade_answer`（choice 去空格精确 / short 精确+≤30 字符模糊 ≥0.85 / open 关键词 `[,;，；。\n]+` 切分覆盖率 ≥0.6 / expected 空恒 false）与 `classify_error`
- [ ] 1.4 实现 `mastery.go`：`compute_mastery`（最近 5 次、权重 (0.5,0.7,0.85,0.95,1.0) 尾部对齐、1/2 次置信封顶 0.5/0.8、固定累加顺序）
- [ ] 1.5 实现 `scheduler.go`：四类型间隔序列、`LEARNING_DEBUG` 秒单位注入、`schedule_next`（连对 +2 跳档/连错 -1 下限 0/clamp）、review queue 重建与优先级（错题 1，MEMORY2/CONCEPT3/PROCEDURE4/DESIGN5）
- [ ] 1.6 实现 `policy.go` 纯函数：`is_mastered`（定量 ≥0.9 / 定性 bool）、`objective_status`、`next_objective`（answer_pending → review → 首个未掌握 probe/assess/practice → complete）、`map_summary`
- [ ] 1.7 用 Python 基线脚本产出 grading/mastery/scheduler 的 golden JSONL（含 1 万条随机 ratio fuzz），checked in `testdata/`，Go 测试逐行断言逐位一致

## 2. 存储与 service

- [ ] 2.1 实现 `storage.go`：`<workspace>/learning/<book_id>.json`、非法 book_id（`/ \ .. :`）拒绝、进程锁 + version++ + 原子写、`get_or_create` 立即落盘、`list_all` 排序
- [ ] 2.2 实现 `service.go`：`grade_and_record` 全流程（attempt → mastery 重算 → 间隔推进 → queue 重建 → 持久化，fail-closed）、`replace_modules`（旧 kp 状态清理 + stage_failure 清空）、redo 语义

## 3. capability 与五工具

- [ ] 3.1 实现 `MasteryPathCapability`：manifest、`mastery_mode` 标记、`mastery_path_id` 解析链（显式 → book_references 首项 → session_id，白名单清洗）、委托 chat pipeline
- [ ] 3.2 实现 `MasteryLoopCapability`：tutor system prompt（`mastery.system` 键可覆盖）、五工具追加、`InjectKwargs` 注入 `_mastery_path_id`/`_session_id`/`_turn_id`；每次工具调用新建 store+service
- [ ] 3.3 实现 `mastery_status`（empty/active 分支）与 `mastery_build`（服务端 id 生成、append 偏移、200 字符截断、无效丢弃、replace_modules、游标重置）
- [ ] 3.4 实现 `mastery_quiz`（choice 完整选项正文校验、expected 标签解析、PendingQuestion 单槽、uuid hex question_id）与 `choices.go` 辅助
- [ ] 3.5 实现 `mastery_grade`（pending 校验、legacy choice 正文从 ask_user 事件恢复、grade_and_record、question bank 同步失败仅告警、返回 3 位小数 mastery）与 `mastery_assess`（仅定性类型、pass→1.0 / fail→min(current,0.4)、Feynman 证据）

## 4. /api/v1/learning REST 面

- [ ] 4.1 实现 8 端点（gin group）：progress 列表（损坏文件跳过记 errors）/详情/map（与 mastery_status 共用策略层）/init-modules（先 cancel 活跃 turn）/import-from-book（`_ch{i}` id、pass_threshold 0.7）/generate-from-notebook（20 条截取、HTML 转义、容错解析 502）/delete（404）/redo
- [ ] 4.2 拷贝 `learning/prompts/{en,zh}.yaml` 并接入 PromptStore（`{records_json}` 注入、语言归一）

## 5. 验收（spec Scenario 落测试 + 协议对等）

- [ ] 5.1 spec 全部 13 个 Scenario 落为 Go 测试：path id 解析、空路径、裸标签拒绝、单次答对 0.5、Feynman 通过、append 建图、short 模糊、两次全对 0.8、连对跳档、pending 优先、非法 book_id、init 前 cancel、zh 提示词
- [ ] 5.2 golden case 对比验收（acceptance.md M3「learning 引擎」行）：grading/mastery/scheduler 对同输入产出与 Python 版一致，CI 常驻
- [ ] 5.3 stage 事件 fixtures 对等验收（acceptance.md M3 矩阵）：录制基线 mastery turn（status→quiz→ask_user→grade 链路）WS fixtures，Go 侧 mock LLM 回放逐事件 diff；`/api/v1/learning` 各端点 contract test 对 golden OpenAPI spec
