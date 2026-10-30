# P1: 结构化输出续聊时 seed token 可能绕过 grammar 约束

## 元信息

| 项 | 内容 |
| --- | --- |
| Severity | P1 |
| 组件 | `uzu-engine` / Language model stream / Grammar |
| 主要文件 | `crates/uzu-engine/src/engine/language_model/stream/stream.rs` |
| 状态 | 待确认 / 待修复 |
| 前置条件 | `capability-grammar` 构建特性 |

## 概述

`LanguageModelStream::generate` 在 `DecodingState::Seeded` 分支中接受 seed token 时，显式丢弃了 grammar 的拒绝结果。后续生成 token 的同类错误则会正常传播。因此，在跨请求复用模型状态并以空输入继续生成时，一个不符合 grammar 的 seed token 会进入模型历史，但 grammar 状态不会正确同步。

这属于 correctness 问题：结构化输出的核心保证是采样结果符合给定 JSON schema 或 regex；如果 seed token 被静默绕过，后续 bitmask、接受历史和回滚状态可能与实际 token 流脱节。

## 复现路径

1. 创建一个 `ChatSession` / `LanguageModelState`，先完成一次带 `Grammar` 的生成。
2. 结束时 `LanguageModelState::last_output_token` 保存最后一个输出 token。
3. 发起第二次空输入请求，并指定一个新的 grammar。
4. `LanguageModelStream::new` 走 `DecodingState::Seeded`。
5. `generate` 调用 `let _ = grammar.accept_token(seed_token);`。
6. 若 seed token 被 grammar 拒绝，错误被丢弃，生成继续。

## 相关代码

`crates/uzu-engine/src/engine/language_model/stream/stream.rs`:

```rust
DecodingState::Seeded {
    seed_token,
} => {
    self.model_state.tokens.push(seed_token);
    #[cfg(grammar)]
    if let Some(grammar) = self.options.grammar.as_mut() {
        let _ = grammar.accept_token(seed_token); // TODO: this should not be ignored
    }
    ...
}
```

同一文件中，`ForwardPassChaining::resolve` 对后续 token 的处理是正确的：

```rust
grammar.accept_token(token_id)?;
```

`Grammar::accept_token` 会在 token 不被接受时返回 `GrammarError::GrammarReject`。

## 影响

- 结构化输出可能产生与 grammar 状态不一致的历史。
- 后续 `rollback` 和 `fill_next_token_bitmask` 可能基于错误的接受历史计算。
- 对需要续聊的 JSON/regex 输出场景，用户可能收到不完整、非法或语义错误的输出。
- 错误没有被暴露给调用方，调试困难。

## 建议修复

最小修复：

```rust
#[cfg(grammar)]
if let Some(grammar) = self.options.grammar.as_mut() {
    grammar.accept_token(seed_token)?;
}
```

需要同时确认两个设计边界：

1. **stop token**：`Grammar::accept_token` 已包含 terminated matcher 例外，允许 stop token。
2. **triggered grammar**：seed token 位于 reasoning 阶段时应不影响 engagement state；如果 seed token 正好触发 grammar，应按现有状态机逻辑处理。

如果维护者认为特定 seed token 不应导致失败，必须显式建模该例外，而不是丢弃错误。

## 建议测试

在 `uzu-engine` 单元测试中增加：

1. **always grammar rejects invalid seed**
   - 构造 WordLevel tokenizer 和 regex grammar。
   - 设置 `LanguageModelState::last_output_token` 为 grammar 不允许的 token。
   - 以空输入创建 stream。
   - 调用 `generate`。
   - 断言返回 `LanguageModelStreamError::Grammar(GrammarError::GrammarReject)`。

2. **valid seed updates grammar state**
   - 设置符合 grammar 的 seed token。
   - 继续生成。
   - 断言 grammar 的下一状态与 seed 已接受一致。

3. **terminated grammar allows stop seed**
   - 将 matcher 置为 terminated。
   - 使用 stop token 作为 seed。
   - 断言不报错。

4. **triggered grammar remains dormant before trigger**
   - 构造 reasoning 阶段的 seed token。
   - 断言 seed 不提前触发 grammar。

## 验证命令

```bash
cargo +nightly nextest run -p uzu-engine engine::language_model
cargo tools test rust
```

## 风险与注意

- `LanguageModelState::last_output_token` 目前只保存 token，不保存旧 stream options；本 issue 只处理 grammar 错误丢弃，不应与采样状态语义重构混在一个 PR。
- 修复后，某些原本“看似成功”的续聊可能变为显式错误。这是预期行为，但需要在 release note 或变更说明中说明。

