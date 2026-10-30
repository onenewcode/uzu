# P1: Catalog 回调可能在持有发布锁时执行用户代码造成死锁

## 元信息

| 项 | 内容 |
| --- | --- |
| Severity | P1 |
| 组件 | `uzu` / Engine catalog publication |
| 主要文件 | `crates/legacy/uzu/src/engine/mod.rs` |
| 状态 | 待确认 / 待修复 |

## 概述

`Engine::handle_registry_refresh` 在持有 `catalog_publish` guard 的情况下直接调用用户提供的 callback：

```rust
let _publish = self.catalog_publish.lock().await;
...
let _ = self.catalog_events.send(());
result?;
if let Some(callback) = self.callback.lock().await.as_ref().cloned() {
    callback.on_event();
};
```

虽然代码在调用 callback 前释放了 `published_catalog`，但 `catalog_publish` guard 仍然存活。callback 是用户/绑定层提供的任意代码，可能在同一异步 runtime 中：

- 调用 `engine.models()`；
- 调用 `engine.refresh_models()`；
- 订阅或处理 catalog events；
- 在 Python/JS/Swift 绑定中触发跨线程等待；
- 重新进入 Engine catalog 相关方法。

如果 callback 同步等待一个需要 `catalog_publish` 的任务，就会死锁。

## 触发示例

1. registry refresh 持有 `catalog_publish`。
2. callback 被调用。
3. callback 提交一个 task 到同一 runtime，并同步 join。
4. 该 task 调用 `engine.refresh_models()`。
5. `refresh_models` 需要等待 catalog publication。
6. callback 无法返回，publication 无法完成。
7. 死锁。

多语言绑定的 callback 常把事件调度到宿主线程。如果实现为了保证顺序，同步等待宿主线程处理，而宿主线程又在等待 engine 操作，也会形成跨线程死锁。

## 影响

- catalog refresh 永远无法完成。
- `catalog_refreshes` 计数不归零，`catalog_snapshot` 永远报告不完整。
- 下游 `models()`、下载、GUI 模型列表卡住。
- 偶发死锁难以复现，尤其出现在多语言绑定或 GUI 集成中。

## 建议修复

### 1. 完全移出锁外调用 callback

重构为：

```rust
let event = {
    let _publish = ...;
    ... // publish snapshot
    event
};

// No Engine locks held here.
if let Some(callback) = self.callback... {
    callback.on_event();
}
```

`callback` 锁也应在进入发布临界区前 clone，或至少确保调用 callback 时不持有它。

### 2. 限制 callback 语义

在 API 文档中明确：

- callback 必须 fast、non-blocking；
- 不得在 callback 中调用 Engine；
- 只能 enqueue event。

仅靠文档不足以避免 P1 风险，代码仍应移出锁。

### 3. 避免同步 callback

如果 callback 只需要通知事件，可统一改为 `catalog_events`，让绑定层订阅事件流，而不是直接调用任意用户代码。

## 建议测试

1. **callback calling models must not deadlock**
   - callback 中调用 `engine.models()`。
   - 使用 timeout 断 refresh 和 callback 均能完成。

2. **callback calling refresh_models must not deadlock**
   - callback 中触发或等待 `refresh_models()`。
   - 使用 timeout 检测。

3. **callback lock does not block publication**
   - 并发设置 callback 与刷新 catalog。
   - 断 refresh 不被 callback lock 饿死。

4. **multi-language integration**
   - 在 Python/JS/Swift 绑定层模拟 callback 同步等待宿主事件循环。
   - 断无死锁。

## 验证命令

```bash
cargo +nightly nextest run -p uzu engine::catalog
cargo tools test python
cargo tools test swift
cargo tools test typescript
```

## 优先级理由

这不是性能优化问题，而是潜在的永久挂起风险。用户 callback 一旦不满足隐含的 non-blocking 约束，核心模型目录功能就会停摆，且问题难以从外部诊断。

