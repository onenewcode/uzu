# P1: ChatSession 工具调用错误路径可能永久停留 ToolCalling 或污染历史

## 元信息

| 项 | 内容 |
| --- | --- |
| Severity | P1 |
| 组件 | `nagare` / ChatSession tool-call loop |
| 主要文件 | `crates/legacy/nagare/src/chat/mod.rs` |
| 状态 | 待确认 / 待修复 |

## 概述

`ChatSession::execute_turn` 使用显式状态机管理 `Idle -> Generation -> ToolCalling -> Generation -> Idle` 转换。但工具调用阶段存在多个提前 `return` / `break` 路径，错误恢复依赖后续清理代码是否被执行。

关键问题：

1. `tokio::select!` 在 `execute_tool_calls`、`cancel_token.cancelled()`、`sender.closed()` 之间竞争。
2. `execute_tool_calls` 是并发执行工具的 future，一旦 select 进入其它分支，该 future 被丢弃。
3. 状态转换失败分支直接 `return`，绕过最终 `state = Idle`。
4. 如果 `execute_tool_calls` 内部 panic 或状态转换 `try_transition` 失败，同样可能绕过恢复逻辑。
5. `sender.closed()` 分支会取消整个 turn，但消费者被 dropped 时无法收到 `Error` chunk；调用方只能观察到流结束，无法区分成功与失败。

这些路径组合起来可能导致 session 永久停留 `ToolCalling`，后续所有请求都返回 `UnableToPerformOperationInCurrentState`。

## 相关代码

### 状态转换

```rust
if self.try_transition(Generation, ToolCalling).await {
    ...
} else {
    return;
}
```

该 `return` 会绕过函数尾部的：

```rust
*self.state.lock().await = ChatSessionState::Idle;
```

### 工具执行竞争

```rust
let tool_messages = tokio::select! {
    tool_messages = self.execute_tool_calls(tool_calls) => tool_messages,
    _ = cancel_token.cancelled() => break,
    _ = sender.closed() => {
        cancel_token.cancel();
        break;
    },
};
```

如果 `execute_tool_calls` 被取消：

- 已启动的工具 future 被丢弃；
- JS/Python 绑定中的 cancellation callback 依赖 future drop；
- Rust 原生 async tool 未必保证外部副作用停止；
- 工具结果不会写入历史；
- 状态清理依赖后续代码。

### 消费者 dropped

`send_input` 中：

```rust
if sender.send(ChatSessionStreamChunk::Replies { ... }).is_err() {
    interrupted = true;
    break;
}
```

此时状态最终可以恢复，但用户没有任何 `Error` chunk 可读。流只是 `None` 结束。

## 影响

- `ChatSession` 可能永久不可用，用户只能重建 session。
- GUI 或绑定层看到流结束但不知道实际原因。
- 工具外部副作用与历史状态可能不一致。
- 多次重试可能因为状态未恢复而全部失败。

## 建议修复

### 1. 统一清理路径

将核心循环放入 `execute_turn_inner`，外层 task 统一执行：

```rust
let result = self.execute_turn_inner(...).await;
self.restore_idle_state(...).await;
result
```

确保所有 `return` / `break` 都经过统一 cleanup。

### 2. 处理 select 中的 cancelled tool future

- 若工具 future 支持 cancellation，显式等待 cancellation；
- 若不支持，记录 abandoned invocation 并保证状态与历史一致；
- 在 API 文档中明确取消语义。

### 3. 消费者 dropped 时记录错误

当 receiver 已 dropped，无法发送 chunk。可以：

1. tracing error / warn；
2. 取消生成；
3. 保证 state 恢复；
4. 在 API 文档中明确 dropped stream 等价于 cancel。

### 4. 状态转换失败应返回错误 chunk

如果 `try_transition` 失败，发送：

```rust
ChatSessionStreamChunk::Error {
    error: ChatSessionError::UnableToPerformOperationInCurrentState,
}
```

然后进入统一 cleanup，而不是直接 return。

## 建议测试

1. **tool future panics**
   - `execute_tool_calls` 内部 panic 被 catch 后转为错误结果；当前代码已有 `catch_unwind`，但应测试状态恢复。

2. **transition to Generation fails**
   - 人为修改 state 使 `try_transition(ToolCalling, Generation)` 失败。
   - 断言最终 state 为 Idle，而不是停留 ToolCalling。

3. **receiver dropped during tool call**
   - 已有类似测试，但应断言 state Idle、后续 reply 可用、历史没有被污染。

4. **cancel during native blocking tool**
   - 模拟不能及时响应 cancellation 的 async tool。
   - 明确当前取消语义并断言不 panic。

5. **error stream remains recoverable**
   - backend 返回错误。
   - 下一轮请求应能重新执行。

## 验证命令

```bash
cargo +nightly nextest run -p nagare chat
cargo tools test rust
cargo tools test python
cargo tools test typescript
cargo tools test swift
```

## 优先级理由

这属于状态机正确性问题，不是单纯的清理瑕疵。它影响用户可见的 session 可用性，且错误路径与并发取消交织，实际故障可能表现为偶发问题。

