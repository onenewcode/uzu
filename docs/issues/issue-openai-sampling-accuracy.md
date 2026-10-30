# P2: OpenAI 采样参数映射会静默丢失 top_k/min_p/repetition 语义

## 元信息

| 项 | 内容 |
| --- | --- |
| Severity | P2 |
| 组件 | `backend-remote` / OpenAI request bridge |
| 主要文件 | `crates/legacy/backend-remote/src/openai/bridging/chat/request.rs`、`crates/legacy/backend-remote/src/openai/bridging/responses/request.rs` |
| 状态 | 待设计确认 / 待修复 |

## 概述

本地 `SamplingMethod::Stochastic` 包含：

- `temperature`
- `top_k`
- `top_p`
- `min_p`
- `repetition_penalty`
- `suffix_repetition_length`

OpenAI Chat Completions 和 Responses 请求构建只映射 `temperature` 和 `top_p`，其余字段全部用 `_` 忽略：

```rust
SamplingMethod::Stochastic {
    temperature,
    top_k: _,
    top_p,
    min_p: _,
    repetition_penalty: _,
    suffix_repetition_length: _,
} => {
    request.temperature = temperature.map(|v| v as f32);
    request.top_p = top_p.map(|v| v as f32);
}
```

这会在跨引擎迁移时产生语义差异。用户使用同一份 `ChatReplyConfig`，本地模型与云模型的行为不同，而 API 没有警告。

## 影响

### 1. 结果分布不同

`top_k`、`min_p` 会显著改变采样分布。如果这些字段被忽略，云端输出可能比本地更随机或更保守。

### 2. Repetition 行为不同

本地模型可能应用 repetition penalty；OpenAI 请求不包含该设置。长文本续写、列表生成或代码输出的重复率会不同。

### 3. 误导 benchmark 与应用

统一 API 的意义是让调用方在不同 backend 间迁移。静默忽略采样语义会削弱 benchmark 对比和用户体验的一致性。

### 4. 类型系统没有提示

使用 `_` 忽略字段使编译器和 API 文档都无法提醒调用方这些配置不适用于 OpenAI。

## 可能设计方向

### 方案 A：严格校验不支持字段

如果 OpenAI 后端不支持这些参数，请求构建时返回错误：

```rust
if top_k.is_some() || min_p.is_some() || repetition_penalty.is_some() {
    return Err(Error::UnsupportedSampling);
}
```

优点：

- 语义明确；
- 不会静默降级。

缺点：

- 兼容性破坏；现有用户可能已在统一配置中设置这些字段。

### 方案 B：返回 warning / metadata

不阻止请求，但在 stream 或 reply stats 中暴露：

```text
unsupported_remote_sampling: ["top_k", "min_p", "repetition_penalty"]
```

优点：

- 兼容现有调用；
- 用户和 GUI 可以提示。

缺点：

- stats 需要扩展，绑定层也要同步。

### 方案 C：文档化明确降级

在 `ChatReplyConfig` 和语言绑定文档中说明 OpenAI 只支持 temperature/top_p。

优点：

- 成本最低。

缺点：

- 无法解决运行时误导，P2 风险仍在。

建议优先 A 或 B；如果为了兼容选择 B，可以提供 strict mode。

## 建议测试

1. **temperature and top_p map**
   - 构造 config。
   - 序列化 request。
   - 断字段值和类型。

2. **greedy sends no sampling parameters**
   - 断默认或显式 Greedy 不设置 temperature/top_p。

3. **unsupported fields are rejected or reported**
   - 根据选方案断言请求构建失败、stats 包含 warning，或文档化行为。

4. **both API types stay consistent**
   - Chat Completions 与 Responses 的映射行为一致。

## 验证命令

```bash
cargo +nightly nextest run -p backend-remote
cargo tools test rust
```

## 相关注意

- `f64 -> f32` 转换也存在精度差异，但对大多数采样参数影响小于语义丢失；可在测试中固定边界值。
- 如果 OpenAI 将来支持更多参数，应按字段逐一映射，并保留 unsupported 报告机制。

