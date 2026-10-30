# P2: OpenAI 流在 tool call 不完整时把 Stop 改写为 ToolCalls

## 元信息

| 项 | 内容 |
| --- | --- |
| Severity | P2 |
| 组件 | `backend-remote` / OpenAI stream state |
| 主要文件 | `crates/legacy/backend-remote/src/openai/stream_state.rs` |
| 状态 | 待确认 / 待修复 |

## 概述

OpenAI stream 累积器中有如下逻辑：

```rust
if let Some(reason) = chunk.finish_reason {
    if matches!(reason, ChatReplyFinishReason::Stop) && !self.tool_calls.is_empty() {
        self.finish_reason = Some(ChatReplyFinishReason::ToolCalls)
    } else {
        self.finish_reason = Some(reason);
    }
}
```

也就是说，只要流中出现过 tool call delta，上游最终 `Stop` 会被改写为 `ToolCalls`。

这个改写隐藏了一个重要差异：出现 tool call delta 不等于所有 tool call 都完整。如果参数流被截断、上游异常终止，或 provider 在 partial tool call 后返回 Stop，本地会把它当作完整 tool-call turn。

## 现有缓解

- `ToolCallState::build(finalize)` 可能根据 finalize 构造 candidate；
- `ChatSession::execute_turn` 遇到 unfinished candidate 时不会执行部分批次；
- Pi extension 在 finish 时检查 complete calls 与 streamed calls 数量。

这些缓解能避免部分执行，但 finish reason 语义仍被提前改写，调用方无法知道“上游其实是 Stop/截断，而不是正常 ToolCalls”。

## 影响

1. **错误分类**
   - 网络截断或 provider 异常可能被归类为正常 tool-call request。
   - 上层难以决定是否重试。

2. **工具调用循环提前进入 tool 路径**
   - `ChatSession::execute_turn` 检查 `finish_reason == ToolCalls`。
   - 如果工具 calls 不可执行，它会退出循环，但用户看到的是没有回复的结束，而不是明确的解析错误。

3. **统计与日志误导**
   - finish reason 被改写后，debug 和 benchmark 无法还原上游行为。

## 建议修复

### 1. 只在 tool call 满足基础完整性时改写

为 `ToolCallState` 增加完整性检查，至少要求：

```rust
self.id.is_some() && self.name.is_some()
```

然后：

```rust
if matches!(reason, Stop)
    && !self.tool_calls.is_empty()
    && self.tool_calls.values().all(ToolCallState::is_complete)
{
    self.finish_reason = Some(ToolCalls);
} else {
    self.finish_reason = Some(reason);
}
```

JSON 参数的完整性可在下游 schema 校验时判断。

### 2. 保留上游 finish reason 并增加本地标志

如果需要表达“有工具调用”，可增加 `has_tool_calls`，但这会改变 output 结构，涉及绑定同步。

### 3. 对 partial tool call 明确报错

如果上游 Stop 时存在未完成 tool call，返回：

```text
OpenAI stream ended with an incomplete tool call
```

这对正确性和调试最友好，但可能对某些兼容 provider 造成严格行为变化。

## 建议测试

1. **complete tool call with Stop maps to ToolCalls**
   - id/name/arguments 完整。
   - 断现有行为保留。

2. **partial tool call keeps Stop**
   - 只有 id，没有 name。
   - 断 finish reason 不被改写。

3. **arguments truncated**
   - tool call 有 id/name，但 JSON 不完整。
   - 至少在下游工具 schema 校验时得到错误，而不是静默结束。

4. **Responses API event shape**
   - `ResponseOutputItemAdded` 给 id/name；
   - `ResponseFunctionCallArgumentsDelta` 补 arguments。
   - 断两个 API 都遵守相同完整性规则。

## 验证命令

```bash
cargo +nightly nextest run -p backend-remote
cargo tools test rust
```

## 优先级理由

finish reason 是上层控制流的核心输入。把它从“上游终止原因”改写为“本地工具调用意图”必须在完整性有保证时进行，否则会把异常流伪装成正常工具调用。

