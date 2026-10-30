# P2: OpenAI Responses 失败状态被映射为正常 Stop

## 元信息

| 项 | 内容 |
| --- | --- |
| Severity | P2 |
| 组件 | `backend-remote` / OpenAI Responses bridge |
| 主要文件 | `crates/legacy/backend-remote/src/openai/bridging/responses/finish_reason.rs` |
| 状态 | 待确认 / 待修复 |

## 概述

OpenAI Responses API 的 `Status::Failed` 当前被映射为：

```rust
Status::Completed | Status::Failed => Some(ChatReplyFinishReason::Stop),
```

这意味着 provider 明确报告失败时，本地把它视为正常停止。调用方无法区分正常完成、生成失败、是否需要重试。

现有测试还明确固化了这个行为：

```rust
assert_eq!(build(&response("failed", None)), Some(ChatReplyFinishReason::Stop));
```

## 影响

1. **错误被吞掉**
   - 上游 `ResponseFailed` 事件带有错误信息，但 finish reason 只有 Stop。
   - 如果没有其它 error event，上层可能把失败当成功。

2. **重试策略失真**
   - 自动化调用方通常依赖 finish reason / error 决定是否重试。
   - Failed 被映射为 Stop 会阻止正确重试。

3. **日志和 benchmark 不准确**
   - 失败请求可能被计入正常请求，影响成功率统计。

## 建议修复

### 1. 将 Failed 映射为 Rejected 或新增错误

最小改动：

```rust
Status::Completed => Some(ChatReplyFinishReason::Stop),
Status::Failed => Some(ChatReplyFinishReason::Rejected),
```

`ChatReplyFinishReason` 已有 `Rejected`，语义上比 `Stop` 更接近失败。

### 2. 保留错误详情

仅改 finish reason 仍不足以调试。应在 `StreamChunk` / `Output` 中保存：

- provider error code；
- message；
- response id；
- status。

或将其转为 `Err(BackendError)` 发送给上层。

### 3. 区分可重试错误

如果上游错误信息表明 quota、timeout、network 等可重试错误，应暴露 `Rejected` 并在错误详情中标记 transient。

## 建议测试

1. **completed maps to stop**
   - 保留现有正常路径。

2. **failed maps to rejected/error**
   - 更新当前测试。
   - 断言失败不会被报告为 Stop。

3. **failure event contains error details**
   - 构造 `response.failed` fixture，包含 `error` 字段。
   - 断错误详情被保留或转换为错误事件。

4. **stream end behavior**
   - 通过真实 stream accumulator 跑完整事件序列。
   - 断上层收到错误/rejected，而不是正常输出。

## 验证命令

```bash
cargo +nightly nextest run -p backend-remote
cargo tools test rust
```

## 兼容性注意

- 修正后，依赖 `Failed -> Stop` 的调用方会看到行为变化；这是必要的 correctness 修复，但应写入变更说明。
- `Status::Incomplete` 已有长度/拒绝映射，`Failed` 不应比 incomplete 更宽松。

