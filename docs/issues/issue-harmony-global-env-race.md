# P1: Harmony 编码加载通过进程级环境变量造成多模型并发污染

## 元信息

| 项 | 内容 |
| --- | --- |
| Severity | P1 |
| 组件 | `hanashi` / Harmony tokenizer |
| 主要文件 | `crates/legacy/hanashi/src/chat/harmony/mod.rs` |
| 关联文件 | `crates/legacy/nagare/src/chat/token.rs` |
| 状态 | 待确认 / 待修复 |
| 平台 | 非 wasm，包含 macOS/iOS/桌面等 |

## 概述

`HarmonyEncodingImpl::new` 通过修改进程级环境变量 `TIKTOKEN_ENCODINGS_BASE` 来指定 tokenizer 加载目录：

```rust
unsafe {
    std::env::set_var("TIKTOKEN_ENCODINGS_BASE", path);
}

let encoding = load_harmony_encoding(encoding_name)?;
```

该值是进程全局状态。多个 Harmony 模型如果使用不同的 tokenizer 目录，并且在同一进程内并发创建 session，后一次 `set_var` 会影响前一次或其它线程正在读取的环境变量，导致：

- 用错误的 tokenizer 目录加载编码；
- session 加载失败；
- token 映射错误；
- 输出解析错误。

## 为什么严重

`ChatSession::new` 与 `with_instance` 可以并发调用。`token::Session::with_instance` 在解析模型 encoding 后创建 `HarmonyEncodingImpl`。在 GUI、CLI server 或多 session 应用中，同时打开两个模型是合理场景。

`std::env::set_var` 本身也是 unsafe 操作，Rust 文档明确警告并发访问环境变量可能造成未定义行为。这不是单纯的逻辑竞态，而是安全性边界问题。

## 受影响场景

1. 同一进程加载两个 Harmony 模型，tokenizer 目录不同。
2. 一个模型正在加载，另一个模型随后开始加载。
3. 一个 Harmony 模型加载期间，另一个线程读取该环境变量。
4. 并发创建多个 `ChatSession` / `ChatInstance`。

## 当前调用路径

```text
ChatSession::new / with_instance
  -> token::Session::with_instance
  -> parse model.encoding
  -> if EncodingConfig::Harmony:
       HarmonyEncodingImpl::new(config, tokenizer_location)
       -> std::env::set_var("TIKTOKEN_ENCODINGS_BASE", path)
       -> load_harmony_encoding(...)
```

## 可能表现

- 模型加载偶发失败，错误为找不到 tokenizer 文件。
- 同一模型在不同加载顺序下行为不同。
- 生成 token 被错误 tokenizer 解析，出现乱码或 tool-call 解析失败。
- 并发测试偶发失败但单线程通过。
- 某些平台可能出现更严重的未定义行为。

## 建议修复

### 首选：避免全局环境变量

与 `openai-harmony` 的加载机制确认是否支持显式路径参数。如果支持：

```rust
let encoding = load_harmony_encoding_from_path(encoding_name, path)?;
```

这是唯一彻底消除竞态的方式。

### 备选：全局串行加载

如果上游依赖不支持显式路径：

1. 引入进程级 `HarmonyLoaderLock`。
2. 在 lock 内执行 set env、load encoding、parse。
3. 加载完成后恢复原 env。
4. 加载后的对象不再依赖该 env。

这能消除大部分竞态，但仍然污染进程环境，并且其它代码若并发读取该变量仍可能受影响。

### 更保守备选：禁止不同 Harmony tokenizer 目录并发加载

全局记录当前已加载的 Harmony tokenizer 目录；若并发请求不同目录，返回明确错误：

```text
Cannot load multiple Harmony tokenizers with different directories in one process
```

这至少把未定义行为变成可诊断错误，但功能受限。

## 建议测试

1. **parallel construction with distinct directories**
   - 创建两个临时 tokenizer 目录。
   - 并发构造两个 `HarmonyEncodingImpl`。
   - 反复运行，断各自解析的 special tokens 正确。

2. **load order does not affect result**
   - 交换加载顺序。
   - 断每个 encoding 的 token id 映射不变。

3. **failed load does not corrupt later model**
   - 第一个加载失败。
   - 第二个正常加载。
   - 断第二个结果正确。

4. **stress test**
   - 20 个并发构造任务。
   - 验证无 panic、无错误目录、无间歇性失败。

## 验证命令

```bash
cargo +nightly nextest run -p hanashi
cargo tools test rust
```

## 注意事项

- 修复时不要简单把 `unsafe` 块移除；关键是移除或串行化全局状态。
- 如果上游库没有显式路径 API，应考虑上游 PR。
- 该问题不应只靠测试重试或延迟初始化掩盖。

