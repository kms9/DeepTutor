# Tasks: impl-foundation-stream

## 1. StreamEvent 信封

- [ ] 1.1 实现 `internal/core/event.go`：`StreamEvent` 结构体（9 个固定 JSON 键）、`EventType` 15 枚举常量、构造器默认值（空串/空 object/`seq=0`/创建时刻 timestamp）
- [ ] 1.2 实现 `ToDict` 与 NDJSON 序列化（`SetEscapeHTML(false)`、nil Metadata 归一为 `{}`、单行输出、非 ASCII 不转义）
- [ ] 1.3 实现 `internal/core/emit.go` 便捷发射器：`tool_call`/`tool_result`/`progress`/`sources`/`result` 的 metadata 形状约定与 trace 元数据合并

## 2. StreamBus 扇出

- [ ] 2.1 实现 `StreamBus` 骨架：mutex + history + 订阅者无界队列（slice + `sync.Cond`），`Emit` 非阻塞投递
- [ ] 2.2 实现 `Subscribe`：同一临界区内完成 history 快照与订阅者注册，出流 goroutine 先回放快照再消费实时队列（不重不漏）
- [ ] 2.3 实现关闭语义：`Close` 投递终止哨兵、closed 后 `Emit` 静默丢弃、closed bus 新订阅回放完即结束流；`ctx` 取消释放订阅者

## 3. wait_for_input 与注册表

- [ ] 3.1 实现 `WaitForInput`/`SubmitInput`：先 emit `wait_for_input` 事件再注册等待者、广播给全部等待者并清空、有限 timeout 超时返回空串不报错、零值 timeout 无限等待
- [ ] 3.2 实现 `internal/core/busregistry.go`：`RegisterBus`/`UnregisterBus`/`GetBus`（不存在返回 nil），并发安全
- [ ] 3.3 实现消费端 seq 去重辅助（`seq <= last_seq` 跳过），并在包文档中固化 turn runtime 的 seq 分配约定

## 4. 测试与验收

- [ ] 4.1 把 `foundation-stream` spec 的全部 Scenario（9 个 Requirement / 12 个 Scenario）逐条落为 Go 测试，含 `-race` 下的并发 emit/订阅回放压力用例
- [ ] 4.2 协议对等验收：对 `contracts/fixtures/*.jsonl` 中的基线 WS 事件样本做逐字段序列化对比（忽略白名单仅 `timestamp`），覆盖 15 种事件类型与中文/HTML 字符内容（对应 acceptance.md §3.1 与 §4 M0 矩阵「WS fixtures 录制器/回放器」的信封侧）
- [ ] 4.3 非功能抽查：单订阅者回放 1 万事件耗时 < 1s 的基准测试（对应 acceptance.md §3.4「WS 回放吞吐」）
