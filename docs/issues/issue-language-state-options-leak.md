# P1: 跨请求续生成会混合旧模型状态与新 LanguageModelStreamOptions

## 元信息

| 项 | 内容 |
| --- | --- |
| Severity | P1 |
| 组件 | `uzu-engine` / Language model state |
| 主要文件 | `crates/uzu-engine/src/engine/language_model/state.rs` |
| 关联文件 | `crates/uzu-engine/src/engine/language_model/stream/stream.rs`、`crates/legacy/uzu/src/engine/bridge/chat_token_backend.rs` |
| 状态 | 待设计确认 / 待修复 |

## 概述

`LanguageModelState` 只保存 `last_output_token`，不保存生成时使用的 `LanguageModelStreamOptions`。代码中的 TODO 明确指出这会泄漏上一次 stream options：

```rust
pub(super) last_output_token: Option<u64>, // TODO: this leaks previous LanguageModelStreamOptions
```

当输入为空且存在 `last_output_token` 时，`LanguageModelStream::new` 会创建 `Seeded` 状态并继续生成。但 legacy 桥接每次都会根据新的 `ChatReplyConfig` 构造新的 options，包括：

- `sampling_method`；
- `grammar`；
- `timestamps`；
- 与 suffix repetition 相关的 context ring 状态。

于是，第二次请求可能把旧模型内部状态与新请求配置混合使用。

## 具体风险

### 1. Sampling 方法不一致

上一轮使用带 `suffix_repetition_length` 的 stochastic sampling 时，`LanguageModelStream::new` 会为模型状态创建并填充 context ring。下一轮如果切换到 Greedy 或不同 suffix 长度：

- context ring 的历史长度和新配置不再对应；
- 新 stream 可能按不同假设继续使用底层状态；
- repetition penalty 的语义可能不可预测。

### 2. Grammar 状态不一致

上一轮生成的 token 已经进入模型状态，但新请求的 grammar 是重新构造的。空输入续生成时：

- 新 grammar 无法知道旧输出已接受多少 token；
- triggered grammar 的 reasoning trigger 匹配历史可能与实际 token 状态不同；
- stop token 语义可能混淆。

### 3. 统计与流生命周期不一致

`timestamps` sender 是每次请求独立的。如果继续生成依赖旧 stream 的异步资源或统计语义，可能出现：

- 旧请求的 command buffer 计时归属错误；
- 新请求 metrics 与实际生成轮次不匹配。

### 4. Session 复用放大问题

`UzuChatTokenBackendInstanceState` 保存 `LanguageModelState`，并由 `ChatSession` 复用。同一 session 中连续多次 `reply` 且消息输入为空时，会直接命中该路径。

## 现有代码依据

`stream.rs`:

```rust
let decoding_state = if !input.is_empty() {
    ...
} else {
    // TODO: this leaks previous LanguageModelStreamOptions
    DecodingState::Seeded {
        seed_token: model_state.last_output_token.take().unwrap(),
    }
};
```

`chat_token_backend.rs`:

```rust
let mut options = self.model.default_stream_options();
options.sampling_method = sampling_method;
options.grammar = grammar;
let stream = self.model.stream(input, state.value.get_mut(), options)
```

## 建议设计方案

### 方案 A：禁止跨配置空输入续生成

在 `LanguageModelState` 中记录生成配置摘要，包括：

- last output token；
- sampling method；
- grammar generation；
- context ring 长度。

新请求与摘要不匹配时返回结构化错误：

```text
Cannot continue an empty-input request with different sampling or grammar settings
```

优点：

- 防止静默 correctness 问题；
- 语义清晰，适合短期修复；
- 对多语言绑定容易暴露明确错误。

缺点：

- 用户必须提供非空输入或保持相同配置。

### 方案 B：显式保存完整 continuation context

保存上一轮 sampling method、context ring 状态或配置、grammar 状态快照、last output token。新请求若配置兼容，则从 continuation context 继续；若不兼容，则重建或报错。

优点：

- 可以支持合法续聊；
- 状态边界明确。

缺点：

- 实现复杂，尤其 grammar 回滚历史和 context ring 的归属。

### 方案 C：总是要求非空输入

如果空输入续生成不是公开 API 承诺，可以直接在 `LanguageModelStream::new` 中拒绝空输入，或改为显式 `continue_generation()` API，而不是让普通 stream 构造函数隐式处理。

## 建议测试

1. **same config can continue**
   - 第一轮指定 stochastic sampling 与 suffix repetition。
   - 第二轮空输入且保持相同配置。
   - 断生成成功且状态一致。

2. **changed sampling config fails**
   - 第一轮 stochastic，第二轮 Greedy。
   - 断得到结构化错误。

3. **changed grammar fails**
   - 第一轮 JSON schema A，第二轮 schema B。
   - 断得到结构化错误。

4. **nonempty input resets continuation assumptions**
   - 传入非空输入。
   - 断新的 prefill 路径不会读取旧 continuation 配置摘要。

5. **legacy session-level behavior**
   - 通过 `ChatSession.reply` 连续请求。
   - 验证用户可见错误信息和 session 状态恢复。

## 验证命令

```bash
cargo +nightly nextest run -p uzu-engine engine::language_model
cargo tools test rust
cargo tools test python
cargo tools test swift
cargo tools test typescript
```

## 排查要点

- 不要只补一个“不同配置时报错”的检查，然后忽略 context ring 的旧状态。
- 修复应覆盖 CPU、Metal、WebGPU 后端的行为。
- 多语言绑定测试很重要，因为空输入请求可能在绑定层被意外构造出来。

