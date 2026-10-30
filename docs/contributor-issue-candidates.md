# 自主排查出的贡献 issue 候选（旧版汇总）

> 排查日期：2026-10-30  
> 状态：已由更详细的按 issue 文档替代，见 `docs/issues/README.md`。

## 排查方法

1. 使用 CodeGraph 追踪核心调用链：下载/模型目录、语言模型流、Metal/CPU kernel、GUI 附件状态。
2. 检查仓库中的 `TODO/FIXME/#[ignore]` 标记，并区分“已失效想法”和“有明确行为风险的缺口”。
3. 对照现有测试，确认风险是否已有覆盖。
4. 用 GitHub CLI 搜索开放 issue，避免重复提交。

## 候选 1：结构化输出 seed token 可能绕过 grammar 约束

**优先级：高**  
**模块：** `crates/uzu-engine/src/engine/language_model/stream/stream.rs`

### 现象

`LanguageModelStream::generate` 在处理 `DecodingState::Seeded` 时，把 seed token 写入模型状态，但对 grammar 接受结果显式忽略：

```rust
if let Some(grammar) = self.options.grammar.as_mut() {
    let _ = grammar.accept_token(seed_token); // TODO: this should not be ignored
}
```

同一实现中，`ForwardPassChaining::resolve` 会对后续 token 调用 `grammar.accept_token(token_id)?`，错误会正常传播为 `LanguageModelStreamError::Grammar`。

### 风险

当上一轮请求结束后，`LanguageModelState::last_output_token` 保存了最后一个输出 token。下一轮以空输入继续生成时，`LanguageModelStream::new` 会走 `DecodingState::Seeded`。此时：

- 如果 seed token 不符合 grammar，`Grammar::accept_token` 返回 `GrammarError::GrammarReject`；
- 当前代码丢弃该错误；
- 后续 bitmask 和 sampler 仍按 grammar 已接受该 token 的状态继续工作。

可能导致结构化输出约束状态与真实 token 序列脱节，之后接受/回滚的 token 数也不一致。

### 建议修复方向

1. 将 `let _ = grammar.accept_token(seed_token);` 改为 `?`。
2. 补一个最小回归测试：构造 Always grammar，先让状态保存一个违反 grammar 的 `last_output_token`，再用空输入继续流式生成，断言得到 grammar 错误而不是静默脱节。
3. 如果维护者认为 seed token 应该被允许，则应明确注释并同步 `Grammar` 状态/回滚历史，不能只是丢错。

### 验证建议

- `cargo +nightly nextest run -p uzu-engine engine::language_model`
- 如果测试依赖 legacy 桥接：`cargo tools test rust`

## 候选 2：`generate` 的 re-entrancy 会把流置成 Invalid 并导致 panic

**优先级：高**  
**模块：** `crates/uzu-engine/src/engine/language_model/stream/stream.rs`

### 现象

`generate` 一开始执行：

```rust
match replace(&mut self.decoding_state, DecodingState::Invalid) {
    ...
    DecodingState::Invalid => unreachable!(),
}
```

在 `DecodingState::ForwardPassPending` 且 `full_accept == false` 的分支中，它先创建 `DecodingState::Accepting`，然后：

```rust
return Box::pin(self.generate()).await;
```

也就是说 `generate` 是内部递归调用。外层调用尚未完成时，内层再次拿到 `&mut self` 并把 `DecodingState::Accepting` 替换成 `Invalid`。内层成功路径最终会恢复为有效状态，因此正常顺序没有问题。

但如果内层递归在中途返回 `Err`，例如 backend copyout 失败、grammar 错误、speculator 错误，外层调用会直接向上返回。此时 `decoding_state` 可能停留在：

- `Invalid`（最坏情况）；
- 或某个尚未完全提交/接受的中间状态。

后续再次 `generate` 会命中 `unreachable!()`。`close` 对 `Invalid` 也只是返回 `Ok(())` 并把 `last_output_token` 清空，没有暴露错误。

### 风险

一次瞬时 backend/grammar/speculator 错误可能永久污染流对象：

- 第二次调用 `generate` panic；
- `close` 无法正确保存/清理状态；
- 持有该 session state 的调用方可能无法恢复，只能整个 session 重建。

当前 `chat_token_backend.rs` 的确会保存同一 `LanguageModelState` 并复用，因此这个风险不是纯理论。

### 建议修复方向

1. 避免用 `DecodingState::Invalid` 作为 re-entrancy 占位。可以先把状态 move 出来，处理失败时恢复原状态，或引入显式错误恢复路径。
2. 对 `DecodingState::Invalid`：
   - `generate` 返回错误而不是 `unreachable!()`；
   - `close` 返回错误而不是静默成功；
   - Debug 命名可以是 `Abandoned`/`Failed`，让语义更准确。
3. 增加单元测试：模拟第二次 `generate` 在 `ForwardPassPending(full_accept=false)` 递归路径失败，断言状态可恢复且再次 `generate` 不 panic。

### 现有 TODO 佐证

代码中已有：

```rust
DecodingState::Invalid => None, // TODO: proper error handling
```

这说明维护者已经知道该分支缺少正确错误语义，只是未完成。

## 候选 3：跨请求复用状态会泄漏上一次 `LanguageModelStreamOptions`

**优先级：中高**  
**模块：** `crates/uzu-engine/src/engine/language_model/state.rs`

### 现象

`LanguageModelState` 只保存 `last_output_token`，不保存生成配置：

```rust
pub(super) last_output_token: Option<u64>, // TODO: this leaks previous LanguageModelStreamOptions
```

当输入为空时，`LanguageModelStream::new` 只用 `last_output_token` 构造 seed 状态：

```rust
DecodingState::Seeded {
    seed_token: model_state.last_output_token.take().unwrap(),
}
```

而 `LanguageModelStreamOptions` 每次由 legacy 桥接根据新的 `ChatReplyConfig` 构造，包含：

- 新的 `sampling_method`；
- 新的 grammar；
- 新的 timestamps sender。

### 风险

上一轮请求结束时，模型内部状态可能仍带有与旧采样/grammar配置相关的上下文，例如：

- suffix repetition 的 context ring 状态；
- PRNG 或 speculator 相关状态；
- grammar 接受历史在旧流中的语义。

下一轮以空输入继续时，会把这个旧状态与全新的 `LanguageModelStreamOptions` 混合使用。旧 TODO 也明确称其为 “leaks previous options”。

### 建议修复方向

1. 明确跨请求继续生成的语义：允许继续还是必须输入非空。
2. 如果允许继续：
   - 在 `LanguageModelState` 中显式保存与继续相关的旧配置，或把它们折叠成明确的状态快照；
   - 新旧配置不兼容时返回结构化错误，例如“不能在更改 sampling/grammar 后继续空输入请求”。
3. 如果不允许继续：
   - 对空输入且存在旧状态的情况返回明确错误，而不是隐式混合。
4. 补 legacy 桥接级测试，覆盖“第一次带 grammar，第二次无 grammar/不同采样，空输入继续”的场景。

## 候选 4：GUI 附件清理依赖 ID 字符串格式，数据损坏风险偏高

**优先级：中**  
**模块：** `apps/mirai-gui/client/src/features/chat/services/attachment-storage.ts`

### 现象

附件 ID 由 `attachment-files.ts` 生成：

```ts
const generateFileId = (): string => {
  return `${Date.now()}-${Math.random().toString(36).slice(2, 8)}`;
};
```

清理逻辑却从 ID 字符串反解析时间戳：

```ts
const timestamp = Number.parseInt(file.id.split("-")[0] ?? "", 10);
if (Number.isFinite(timestamp) && timestamp < sevenDaysAgo) {
  fileIdsToDelete.push(file.id);
}
```

`AttachedFile` 结构本身没有 `createdAt` 字段：

```ts
export type AttachedFile = {
  id: string;
  name: string;
  mimeType: string;
  size: number;
  content: string;
  extension: string;
};
```

### 风险

- 附件记录是 JSON，没有 schema/版本校验；如果旧版本、未来迁移或测试写入其它格式的 ID，时间戳推断会失效。
- 对无法解析的 ID，当前策略是永久保留，可能无限增长 localStorage。
- 一旦 ID 生成规则改变，历史附件的保留期语义随之改变。
- `Date.now()` 与随机字符串耦合在 ID 中，时间语义依赖当前实现细节。

这不是立即的功能 bug，但属于低成本的可靠性改进，且适合补纯单元测试。

### 建议修复方向

1. 给 `AttachedFile` 增加显式 `createdAt: number`。
2. `generateFileId` 与时间信息解耦，可使用 `crypto.randomUUID()`。
3. `cleanup()` 使用 `file.createdAt` 判断。
4. 对缺少 `createdAt` 的旧数据做一次性迁移：若 ID 前缀可解析则迁移；否则可按“保留”或“删除”明确选择。
5. 增加测试：
   - 超过 7 天的记录被删除；
   - 7 天内记录保留；
   - 新格式 ID 不影响 cleanup；
   - localStorage JSON 异常时不会导致应用初始化崩溃。

## 候选 5：被忽略的 flaky kernel 测试缺少隔离和记录

**优先级：中**  
**模块：** `crates/uzu-engine/unit/backends/common/kernel/gated_act_mul_test.rs`

### 现象

该测试被明确标注：

```rust
#[ignore] // TODO: flaky
fn test_gated_act_mul_interleaved_hadamard_bf16()
```

测试对比 CPU 与非 CPU 后端在 `bf16 + Hadamard + SILU` 下的输出，容差为 `0.02`。

### 风险

- ignored test 长期不跑，掩盖 kernel correctness 差异；
- “flaky”只有注释，没有机器可读的原因、复现频率、失败样本或硬件信息；
- 同文件存在其它 BF16/Hadamard/quantized 测试，无法确定失败是 kernel 差异、异步提交/等待问题，还是测试数据生成问题。

### 建议修复方向

1. 先建立复现脚本，记录失败硬件、backend、输入种子与最大误差。
2. 将测试输入从 `Math.sin` 序列改为确定性 seeded 数据，并在失败信息中输出：
   - backend；
   - index；
   - expected/actual；
   - max/mean error。
3. 明确 CPU 与 Metal 的 BF16 计算精度差异是否符合设计：
   - 若符合，测试应区分数值容差；
   - 若不符合，这是 kernel bug。
4. 在解决前不要直接移除 `#[ignore]`，避免 CI 变成随机红。

### 验证建议

```bash
cargo +nightly test -p uzu-engine test_gated_act_mul_interleaved_hadamard_bf16 -- --ignored
```

必要时在多台 Apple GPU 机器上重复运行并保存结果。

## 优先级建议

| 排名 | 候选 | 原因 |
| --- | --- | --- |
| 1 | 候选 1 | 明确的丢弃错误，修复小，可能影响结构化输出 correctness |
| 2 | 候选 2 | 错误后可能 panic 或永久污染流状态 |
| 3 | 候选 3 | TODO 已承认状态语义泄漏，涉及跨请求采样/grammar correctness |
| 4 | 候选 4 | 数据模型可靠性改进，适合较小 PR |
| 5 | 候选 5 | 需要硬件复现与测试诊断，适合后续专项 |

## 提 issue 前建议

1. 对候选 1、2、3，先在 issue 中引用对应行号和 TODO 注释，说明需要维护者确认设计意图。
2. 对候选 4，单独提出 UI 数据模型改动，避免和清理逻辑修复混在一个 PR。
3. 对候选 5，先收集一次失败样本，否则 issue 可能缺少可复现信息。
4. 每个 issue 只描述一个问题，不要把修复方案的多项重构都绑在一起。

