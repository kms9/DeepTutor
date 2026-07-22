# Tasks: impl-memory

## 1. 存储基座（L1/L2/L3）

- [ ] 1.1 `internal/memory/paths.go` + `ids.go`：per-user 根解析、partner scope 覆盖、Crockford base32 ULID（与基线 `ids.py` 逐例对齐）
- [ ] 1.2 `internal/memory/trace.go`：JSONL append（per-surface 锁、永不抛错）、按时间遍历（since/cutoff）、按 id 集合回查、坏行跳过
- [ ] 1.3 `internal/memory/document.go`：L2/L3 markdown 解析/渲染（`m_<ULID>` entry、footnote 引用、四类引用值）、基线样例 round-trip 字节对比测试
- [ ] 1.4 `internal/memory/ops.go` + `store.go`：ops 原子应用与逐条报告、OverwriteDoc/DeleteEntry/ResetDoc（临时文件+rename）、per-path 互斥锁表
- [ ] 1.5 `internal/memory/migrate.go`：v1 松散文件归档 `backup/<UTC ts>/`（幂等）
- [ ] 1.6 `internal/memory/settings.go`：`memory:` 设置子树 defaults 合并读写

## 2. consolidator 四操作与 run 生命周期

- [ ] 2.1 拷贝基线 `{en,zh}` prompts 至 `prompts/memory/`；`consolidator/chunker.go` + `meta.go`（seen-ids 增量）
- [ ] 2.2 `consolidator/parse.go` + `references.go`：LLM 输出 → ops、引用池校验拒绝
- [ ] 2.3 `consolidator/consolidator.go`：update/audit/dedup（iterations）/merge 四模式，eino ChatModel 调用与 `llm_selection` 覆盖
- [ ] 2.4 preferences 门禁：`update`/`audit` 对 `L3/preferences` 返回 405 语义
- [ ] 2.5 `consolidator/runs.go`：run manager（单文档单活动 run 409、断连继续执行、seq ring buffer、cancel、undo 栈）

## 3. snapshot 工作区镜像

- [ ] 3.1 `snapshot/snapshot.go`：per-surface 实体派生 adapter、与持久化状态 diff 出 `pending_changes`
- [ ] 3.2 refresh 提交 `changes.jsonl` + `last_refresh` 更新；changes 分页读取与清空

## 4. memory REST 工作台（gin route group /api/v1/memory）

- [ ] 4.1 总览组：`GET /overview`（11 份文档存在性/更新时间/条目数/backlog + backup 列表）、`GET /backup`
- [ ] 4.2 文档组：`GET/PUT /doc/{layer}/{key}`、`GET .../lines`、`DELETE .../entry/{id}`、`POST .../reset`（活动 run 409）、`POST .../apply`（preferences 405）；layer/key 校验（未知 404、非法 400）
- [ ] 4.3 解析组：`GET /resolve_entry/{entry_id}`（固定 surface 顺序扫 7 份 L2，首命中返回，未命中 404）
- [ ] 4.4 run 组：`POST /runs/start`、`GET /runs/{id}`、`GET /runs`、`GET /runs/{id}/events`（SSE since 重放）、`POST /runs/{id}/cancel`、`POST /runs/{id}/undo`；legacy `POST /doc/{layer}/{key}/{update|audit|dedup}` 薄包装内联 SSE
- [ ] 4.5 设置组 `GET/PUT /settings`；trace 组 `GET /trace/{surface}`（分页）、`DELETE /trace/{surface}`、`DELETE /trace/{surface}/day/{date}`
- [ ] 4.6 snapshot 组：`GET /snapshot/{surface}`、`POST .../refresh`、`GET/DELETE .../changes`

## 5. 工具存储接口（供 impl-tools-builtin）

- [ ] 5.1 `ReadL3Concatenated`：固定顺序拼接（`\n\n---\n\n` 分隔、尾部换行）、全空返回固定 no-memory 提示
- [ ] 5.2 `WritePreference`：trace-first（先 `preference_stated` 事件后写 preferences）、来源引用为传入 trace id、edit 缺/未命中 target 的 `accepted=false` 报告、`reason` 仅记日志

## 6. 测试与验收（spec Scenario 落测试 + 协议对等）

- [ ] 6.1 将本 spec 全部 13 个 Scenario 落为 Go 测试：免迁移续写、v1 归档幂等、trace 写失败静默、entry 删除/404、preferences 405、update 增量（seen-ids）、run 断线重连（since 重放）、overview backlog、snapshot refresh、空记忆提示、edit 缺 target_id、并发写串行化
- [ ] 6.2 数据兼容验收：用基线 `data/memory/` 快照跑「读→写→回读」双向兼容（acceptance.md 3.2 B 类；Go 写入后 Python 侧解析不破坏引用链）
- [ ] 6.3 `/api/v1/memory` 全路由 contract test 对 OpenAPI golden spec 逐字段 diff（acceptance.md M2 矩阵「Memory」项 + 3.1 A 类 REST 对等）
- [ ] 6.4 consolidator 端到端（mock LLM 回放）：update/audit/dedup/merge 产出结构合法且带来源引用（M2 矩阵通过标准）
