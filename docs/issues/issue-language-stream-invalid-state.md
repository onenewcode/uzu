# P1: LanguageModelStream 错误路径可能把流置入 Invalid 状态并导致后续 panic

## 元信息

| 项 | 内容 |
| --- | --- |
| Severity | P1 |
| 组件 | `uzu-engine` / Language model stream |
| 主要文件 | `crates/uzu-engine/src/engine/language_model/stream/stream.rs` |
| 状态 | 待确认 / 待修复 |

## 概述

`LanguageModelStream::generate` 使用 `DecodingState::Invalid` 作为 re-entrancy 过程中的临时占位状态。该分支目前被标记为不可达：

```rust
DecodingState::Invalid => unreachable!(),
```

但在 `ForwardPassPending(full_accept == false)` 分支里，`generate` 会通过递归重新进入自身。递归路径中存在多个可失败点，例如 backend copyout、grammar 接受、speculator tree proposal、command buffer 创建等。一旦内层递归返回错误，外层函数直接向上传播，流对象可能停留在 `Invalid` 或其它未恢复状态。此时再次调用 `generate` 会 panic，`close` 也会把 `last_output_token` 清空并返回成功，无法暴露异常。

这会把一次可恢复的瞬时错误升级为永久损坏的 session/stream 状态。

## 关键路径

`generate` 的入口：

```rust
match replace(&mut self.decoding_state, DecodingState::Invalid) {
    ...
}
```

低概率 full-accept 路径：

```rust
self.decoding_state = DecodingState::Accepting { ... };
return Box::pin(self.generate()).await;
```

再次调用 `generate` 时：

```rust
DecodingState::Invalid => unreachable!(),
```

`close` 的处理：

```rust
DecodingState::Invalid => None, // TODO: proper error handling
...
self.model_state.last_output_token = last_output_token;
Ok(())
```

## 失败场景

1. 当前状态为 `ForwardPassPending` 且 `full_accept == false`。
2. 第一次 `generate` 等待 pending command。
3. copyout sampled tokens 成功，构造 `DecodingState::Accepting`。
4. 递归调用 `generate`。
5. 递归过程中发生 backend/grammar/speculator 错误。
6. 外层返回错误，`decoding_state` 可能停留为 `Invalid`。
7. 调用方如果重试或调用 `close`：
   - 重试 `generate` 会 `unreachable!()` panic；
   - `close` 会丢失 `last_output_token` 并返回 `Ok(())`。

## 影响

- 一次错误可能永久污染 stream。
- 调用方无法安全重试或清理。
- panic 可能发生在多语言绑定进程中，用户看到的是进程崩溃而非可处理的错误。
- `ChatSession` 状态最终可能恢复 Idle，但底层 `LanguageModelState` 已经不一致。

## 建议修复

### 1. 消除 `Invalid` 作为临时状态

不要用 `replace(&mut self.decoding_state, DecodingState::Invalid)`。改为：

- 先把原状态 move 出来；
- 处理过程中维护局部状态；
- 成功时写入最终状态；
- 失败时恢复原状态或写入明确的 `Failed` 状态。

### 2. 为不可恢复状态定义显式错误

如果保留类似状态，建议引入 `DecodingState::Failed`，并：

- `generate(Failed)` 返回 `InvalidState` / `Abandoned` 错误；
- `close(Failed)` 返回错误；
- 不使用 `unreachable!()`。

### 3. 修复递归实现

可以用状态机函数替代递归，避免 re-entrancy：

1. `resolve_pending` 只处理 pending 到 accepting；
2. 返回“需要继续处理”的信号；
3. 外层 loop 继续 `Accepting` 分支。

这样错误路径更容易保证状态一致。

## 建议测试

1. **recursive failure preserves stream state**
   - 使用可控 backend 模拟 `ForwardPassPending(full_accept=false)`。
   - 让第二次 copyout 或后续 encode 失败。
   - 调用 `generate` 收到错误。
   - 再调用 `generate`，断言返回错误而不是 panic。

2. **close reports abandoned state**
   - 将流置入失败/中断状态。
   - 调用 `close`。
   - 断言返回错误，且不默默覆盖 `last_output_token`。

3. **successful partial acceptance remains callable**
   - 构造 sampled tokens 与 trie 的部分接受序列。
   - 连续调用 `generate`。
   - 断言不 panic 且输出正确。

## 验证命令

```bash
cargo +nightly nextest run -p uzu-engine engine::language_model
cargo +nightly clippy --workspace --all-targets --features uzu-engine/webgpu -- -D warnings
```

## 风险与注意

- 该状态机与 speculator、command buffer 和 backend 提交顺序耦合，改动需要谨慎。
- 不建议只删除 `unreachable!()` 并返回错误；必须同时修复失败时的状态恢复语义。
- 修复应在 core engine 层完成，避免只在 legacy 桥接层捕获 panic。

