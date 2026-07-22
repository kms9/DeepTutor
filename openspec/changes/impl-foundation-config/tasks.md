# Tasks: impl-foundation-config

## 1. 模块骨架与 PathService

- [ ] 1.1 建立 `internal/config` 包骨架与 `atomicio.go`（temp + rename 原子写，含单测）
- [ ] 1.2 实现 `PathService` 全部路径 API（workspace root、`data/user/` 布局、chat feature 子目录、`SettingsFile`/`RuntimeConfigFile` 扩展名补全、`ForUser` 多用户实例化）
- [ ] 1.3 实现 `EnsureAllDirectories` 幂等创建与 legacy memory（`data/user/workspace/memory/*.md`）单向复制

## 2. settings 加载层

- [ ] 2.1 实现 `SettingsStore` load-or-create 通道：viper 读取、defaults 合并、归一化钩子、损坏 JSON 容错回落默认、变更才原子写回
- [ ] 2.2 实现 `system.json` 默认值与字段级归一化（端口钳制、truthy/falsy 布尔、附件限额钳制与 total 抬升、`cors_origins` 归一化）
- [ ] 2.3 实现 `auth.json`、`integrations.json`、`pageindex.json` 等其余基线设置文件族的默认值与归一化

## 3. env override 与环境渲染

- [ ] 3.1 实现 `envoverride.go`：基线全量 override 键表（system/auth/MinerU/PageIndex 类）、仅内存生效、`DEEPTUTOR_IGNORE_PROCESS_ENV_OVERRIDES` 全局禁用
- [ ] 3.2 实现 `ExportEnvironment` 与 self-echo 抑制（导出登记表、外部预先存在或值不同才生效）
- [ ] 3.3 实现 `RenderEnvironment`（子进程环境变量表，含 `DEEPTUTOR_API_BASE_URL` 三级解析顺序）

## 4. document_parsing 与公共产物判定

- [ ] 4.1 实现 `document_parsing.json` v2 结构、引擎/mode/model_version 枚举校验与回落
- [ ] 4.2 实现 legacy 迁移：`mineru.json` 原地重命名 + v1 扁平结构迁入 `engines.mineru` 并钉 active engine
- [ ] 4.3 实现 `IsPublicOutputPath`（`data/user/` 边界 + 私有后缀黑名单 + 相对子树白名单，含路径穿越防护）

## 5. 测试与验收

- [ ] 5.1 把 `foundation-config` spec 的全部 Scenario（8 个 Requirement / 13 个 Scenario）逐条落为 Go 测试（`internal/config/*_test.go`，表驱动）
- [ ] 5.2 数据兼容验收：用 Python v1.5.2 生成的 `data/` 快照跑「打开既有数据目录」「settings 直接生效」测试（对应 acceptance.md §3.2 B 类与 §4 M0 矩阵「Go 骨架：config 读取现有 settings JSON」）
- [ ] 5.3 反向兼容抽查：Go 写回的 settings JSON 能被 Python 基线正常加载（acceptance.md §3.2 第 2 条回滚前提）
