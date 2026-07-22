## ADDED Requirements

### Requirement: mastery_path capability 装配
系统 SHALL 注册名为 `mastery_path` 的 capability：manifest 声明 `stages=["responding"]`、`tools_used=[mastery_status, mastery_quiz, mastery_grade, mastery_assess, mastery_build, rag, read_source, ask_user]`、`cli_aliases=["mastery"]`。`run()` SHALL：置 `context.metadata["mastery_mode"]=true`；解析 `mastery_path_id`——优先取前端显式传入的 `metadata.mastery_path_id`，其次 `metadata.book_references` 首项（字符串或 dict 的 `book_id`/`id`），最后回退 `session_id`；id 经 `[A-Za-z0-9_-]` 白名单清洗（与存储层路径守卫一致）；随后运行标准 chat loop。MasteryLoopCapability（`is_active` 由 `mastery_mode` 判定）SHALL 注入 tutor system prompt（`capabilities/mastery/prompts/{en,zh}/system.md`，可被 chat prompts 的 `mastery.system` 键覆盖），并对五个 `mastery_*` 工具调用服务端注入 `_mastery_path_id`、`_session_id`、`_turn_id`——模型永不提供这些 id。结果 envelope SHALL 与 chat 相同，经 `emit_capability_result()` 收敛（response payload + `metadata.cost_summary`）。

入口探查工具 `mastery_status` SHALL 无参数，读取当前 path：路径尚无任何 knowledge point 时返回 `{status: "empty", message: 提示先设计并调用 mastery_build}`；否则返回 `{status: "active", next: next_objective(progress).to_dict(), map: map_summary(progress)}`。每次工具调用 SHALL 新建 store + service 实例（与 REST 路由一致，避免共享对象竞态）。

#### Scenario: 从 book 引用解析 path id
- **WHEN** turn 无显式 `mastery_path_id` 但 `metadata.book_references=[{"book_id": "bk_42"}]`
- **THEN** 本 turn 的 mastery path id 为 `bk_42`（清洗后），五个工具调用均携带该 id

#### Scenario: 空路径首次进入
- **WHEN** 学习者在未建路径的会话中发起 mastery turn，模型调用 `mastery_status`
- **THEN** 返回 `status: "empty"`，指引模型基于材料设计模块并调用 `mastery_build`

### Requirement: mastery_quiz 工具（客观题注册）
`mastery_quiz` SHALL 为 MEMORY / PROCEDURE 目标注册问题：必填 `knowledge_point_id`、`question`、`expected_answer`；`question_type ∈ {choice, short, open}` 默认 `short`。choice 类型 SHALL 校验 `options` 携带完整选项正文（如 `["A: ...", "B: ..."]`）——仅有裸标签（A/B/C/D）时返回失败并要求重试；`expected_answer` SHALL 解析为稳定标签（可为标签或唯一匹配某选项正文），解析失败返回失败。未知 `knowledge_point_id` SHALL 失败并指引调 `mastery_status`。成功时 SHALL 以服务端生成的 `question_id`（uuid hex）构造 `PendingQuestion` 持久化到 progress（`pending_question` 单槽覆盖），返回 `{status: "registered", knowledge_point_id, question, options, instruction}`，instruction 要求随后经 `ask_user` 呈现问题、再调 `mastery_grade`。

#### Scenario: 裸标签选项被拒
- **WHEN** 模型传 `question_type="choice"`、`options=["A","B","C","D"]`
- **THEN** 工具返回 `success=false`，要求携带与 ask_user 一致的完整选项正文重试

### Requirement: mastery_grade 工具（确定性判分）
`mastery_grade` SHALL 只接受 `answer`：无 pending question 时失败提示先 `mastery_quiz`。choice 题 SHALL 先重解析注册时的选项正文（legacy 路径只存了裸标签时，从该 turn 的 `ask_user` 事件恢复正文），并把 expected 归一为标签。判分 SHALL 走 `grade_and_record` 全流程：记录 attempt → 重算 mastery → 推进间隔重复状态 → 重建 review queue → 持久化；判分 fail-closed（expected 为空一律判错）。判分后 SHALL 把该题同步到 session 的 question bank（notebook entries，题型映射 choice→choice、open→written、其余→short_answer；失败仅告警不阻断），清除 pending question，返回 `{is_correct, knowledge_point_id, mastery(3 位小数), threshold, mastered, next}`。

#### Scenario: 答对后门未过
- **WHEN** 学习者首次答对某 MEMORY 目标的问题
- **THEN** 返回 `is_correct: true`、`mastery: 0.5`（单次尝试置信上限）、`threshold: 0.9`、`mastered: false`，`next` 仍指向同一目标

### Requirement: mastery_assess 工具（定性门）
`mastery_assess` SHALL 只对 CONCEPT / DESIGN 类型生效：对定量类型调用 SHALL 失败并指引改用 `mastery_quiz` + `mastery_grade`。成功时 SHALL 记录定性结论：`qualitative_mastery[kp_id] = passed`；显示掌握度同步——pass 时抬到 1.0，fail 时压到 ≤ 0.4（`min(current, 0.4)`）；`feedback` 非空时存为 Feynman 证据。返回 `{knowledge_point_id, passed, mastered, mastery, next}`。

#### Scenario: Feynman 讲解通过
- **WHEN** 学习者讲解 CONCEPT 目标后模型调用 `mastery_assess(passed=true, feedback="...")`
- **THEN** 该目标 `mastered: true`、地图显示掌握度 1.0，`next` 前进到下一个未掌握目标

### Requirement: mastery_build 工具（建图）
`mastery_build` SHALL 接受模型设计的 `modules`（每项 `{name, knowledge_points: [{name, type}]}`）与 `mode ∈ {replace, append}`（默认 replace，非法值归 replace）。id SHALL 服务端生成：module 为 `<path_id>_m<i>`（append 时 i 从现有模块数偏移），kp 为 `<module_id>_kp<j>`；模块名/kp 名截断 200 字符，kp 名长度 < 2 的丢弃，未知 type 回退 `concept`，无有效 kp 的模块丢弃；全部无效时返回失败。成功时 SHALL 调 `replace_modules`（清理不在新集合中的旧 kp 状态：mastery_levels、knowledge_types、repetition_states、error_records、feynman_*、review_queue，并清空 stage_failure_*），重建后 SHALL 置空 pending_question、把游标指向首模块，持久化并返回 `{status: "built", mode, modules_added, knowledge_points_added, map}`。

#### Scenario: append 模式扩展路径
- **WHEN** 已有 2 个模块的路径上调用 `mastery_build(mode="append", modules=[一个新模块])`
- **THEN** 新模块 id 为 `<path_id>_m2`，旧模块与其掌握状态保留，返回 `modules_added: 1`

### Requirement: 判分算法逐位一致（grading）
`grade_answer(user, expected, question_type)` SHALL 与基线逐位一致，Go 实现对相同输入 SHALL 产出相同布尔结果：双方先 `strip + lower`；expected 为空恒 false。`choice`：去除全部空格后精确相等。`short`：精确相等为 true；否则当 expected 长度 ≤ 30 时用 Ratcliff-Obershelp 相似度（Python `difflib.SequenceMatcher.ratio()` 的算法，Go SHALL 复刻同一算法而非任意编辑距离）≥ 0.85 判对；更长的 expected 不做模糊匹配。`open`：把 expected 按 `[,;，；。\n]+` 切成关键词，非空关键词中出现在 user 答案里的比例 ≥ 0.6 判对；无关键词恒 false。错误分类 `classify_error`：空答案 → `METACOGNITIVE`，其余 → `APPLICATION_ERROR`。

#### Scenario: short 题模糊匹配
- **WHEN** expected 为 `photosynthesis`（≤30 字符），user 答 `photosynthesys`
- **THEN** Ratcliff-Obershelp 相似度 ≥ 0.85，判定为正确，且 Go 与 Python 结果一致

### Requirement: 掌握度算法逐位一致（mastery）
`compute_mastery(correctness)` SHALL 与基线一致：取按时间顺序的最近至多 5 次结果，用近因权重 `(0.5, 0.7, 0.85, 0.95, 1.0)`（权重尾部对齐最近的尝试）做加权正确率；结果再套置信上限——1 次尝试封顶 0.5，2 次封顶 0.8，3 次及以上不封顶；空历史返回 0.0。浮点计算顺序 SHALL 保持一致以保证对同输入输出相同（3 位小数展示层四舍五入）。

#### Scenario: 两次全对仍未过门
- **WHEN** 某 kp 的历史为 `[true, true]`
- **THEN** 加权正确率为 1.0 但被置信上限压到 0.8，`is_mastered` 为 false（< 0.9）

### Requirement: 间隔重复调度算法逐位一致（scheduler）
调度器 SHALL 与基线一致：间隔序列（单位天）为 MEMORY `[0,1,3,7,14,30,60]`、CONCEPT `[3,7,14,30]`、PROCEDURE `[3,7,14]`、DESIGN `[14,28]`；环境变量 `LEARNING_DEBUG ∈ {1,true,yes}` 时单位改为秒。初始状态 `interval_index=0`，`next_review_at = now + intervals[0]*unit`。`schedule_next`：答对时连续错误清零、连续正确 +1，连续正确达 2 时 index +2 并清零连续正确，否则 index +1；答错时连续正确清零、连续错误 +1、index 减 1（下限 0），连续错误达 2 时清零连续错误；最后 index clamp 到 `[0, len-1]` 并重算 `next_review_at`。review queue 重建 SHALL 覆盖所有有 repetition state 的 kp：优先级为 1（该 kp 有 active/retrying 错题记录）否则按类型 MEMORY=2 / CONCEPT=3 / PROCEDURE=4 / DESIGN=5；到期任务按优先级升序返回。

#### Scenario: 连续两次答对跳档
- **WHEN** 某 MEMORY kp 状态为 `interval_index=1, consecutive_correct=1`，再次答对
- **THEN** `interval_index` 变为 3（+2 跳档）、`consecutive_correct` 清零，下次复习时间为 now + 7 天

### Requirement: 门控策略与 next_objective
策略层 SHALL 为纯函数（无 LLM、无 I/O）：`is_mastered` —— MEMORY/PROCEDURE 为 `mastery_levels[kp] ≥ 0.9`；CONCEPT/DESIGN 为 `qualitative_mastery[kp] == true`。`objective_status` ∈ `{mastered, learning, new}`（有任意 attempt 或定性记录即 learning）。`next_objective` SHALL 按固定优先序决策并返回 `NextStep`（字段 `action/module_id/module_name/knowledge_point_id/knowledge_point_name/knowledge_point_type/status/gate/mastery/threshold/reason/pending_prompt`）：(1) 有 pending question → `answer_pending`；(2) 有到期复习 → `review`；(3) 按 module.order、kp 顺序找到首个未掌握目标——`new` 状态给 `probe`，定性门给 `assess`，其余给 `practice`（已掌握目标被跳过，即 "test out"）；(4) 全部掌握且无到期 → `complete`。`map_summary` SHALL 返回 `{counts{mastered,learning,new,total}, due_reviews, complete, modules[{id,name,order,mastered,total,knowledge_points[{id,name,type,status,mastery}]}]}`。

#### Scenario: 未答问题优先于一切
- **WHEN** progress 同时存在 pending question 与到期复习任务
- **THEN** `next_objective` 返回 `action: "answer_pending"` 且 `pending_prompt` 为原问题文本

### Requirement: learning 存储
`LearningStore` SHALL 把每个 path 存为 `<workspace>/learning/<book_id>.json`：`book_id` 含 `/`、`\`、`..`、`:` 之一 SHALL 拒绝；保存 SHALL 在进程级锁内进行，每次保存递增 `version`、刷新 `updated_at`，并用原子写（临时文件 + rename）落盘；`load` 不存在返回空；`list_all` 返回全部非隐藏 `*.json` 的 stem 排序。`get_or_create` 对新 path SHALL 立即持久化空 progress（防并发竞态）。

#### Scenario: 非法 book_id 被拒
- **WHEN** 以 `../etc/passwd` 作为 book_id 调用存储
- **THEN** 存储层抛出校验错误，不产生任何文件访问

### Requirement: /api/v1/learning REST 面
系统 SHALL 提供以下端点（均先做 book_id 校验：空、含 `..`/`/`/`\`/`:` → 400）：
- `GET /progress`：全部路径摘要 `{summaries: [{book_id, name, modules_count, kp_count, current_stage, avg_mastery_pct, updated_at}], errors}`（avg 为当前 kp 的平均掌握度百分比，损坏文件跳过并记入 errors）。
- `GET /progress/{book_id}`：完整 progress dump（不存在则创建空）。
- `GET /progress/{book_id}/map`：`{book_id, next: next_objective(...), map: map_summary(...)}`——与 tutor 的 `mastery_status` 共用同一策略层，保证 dashboard 与 tutor 一致。
- `POST /progress/{book_id}/init-modules`：解析并校验模块（无模块或存在空 kp 模块 → 400，pydantic 级字段错误 → 422），SHALL 先取消该 path 的活跃 turn，replace 语义初始化并把游标指向首模块。
- `POST /progress/{book_id}/import-from-book`：由章节列表构造 concept 型 kp 模块（id `"{book_id}_ch{i}"` / `"..._kp{j}"`，pass_threshold 0.7），其余同 init。
- `POST /progress/{book_id}/generate-from-notebook`：截取至多 20 条记录（字段 HTML 转义并截断），调 LLM 生成模块 JSON（容错解析，无效结构 → 502），构造规则同 mastery_build（id 前缀 `_nb`），其余同 init。
- `DELETE /progress/{book_id}`：不存在 → 404。
- `POST /progress/{book_id}/redo`：保留模块结构，清空全部学习状态（mastery、attempts、errors、repetition、queue、pending、feynman、stage failure、diagnostic），stage 重置 DIAGNOSTIC、游标回首模块。

#### Scenario: init-modules 前取消活跃 turn
- **WHEN** 某 path 存在进行中的 mastery turn 时调用 `init-modules`
- **THEN** 该 turn 先被 cancel，再替换模块并保存——避免 turn 收尾用旧 progress 覆盖新图

### Requirement: prompts 资源复用
tutor system prompt SHALL 来自 `capabilities/mastery/prompts/{en,zh}/system.md`；`generate-from-notebook` 的生成提示词来自 `learning/prompts/{en,zh}.yaml`（经 `learning/prompts.py` 键寻址）。Go 版 SHALL 原样拷贝资源文件并实现相同的 `{var}` 模板注入与语言归一（`zh*` → zh，其余 en）。

#### Scenario: 中文 UI 下的建图提示词
- **WHEN** UI 语言为 `zh-CN` 时调用 `generate-from-notebook`
- **THEN** 使用 `learning/prompts/zh.yaml` 的 system/user 模板，记录 JSON 注入 `{records_json}` 占位符
