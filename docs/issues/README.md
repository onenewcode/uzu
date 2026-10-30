# 自主排查 issue 候选索引

> 排查日期：2026-10-30  
> 标准：只收录 P1/P2 问题；P1 为 correctness、数据一致性、死锁或崩溃风险，P2 为明显的语义偏差、可观测性缺口或可靠性问题。

## P1

| 文档 | 模块 | 核心风险 |
| --- | --- | --- |
| [issue-grammar-seed-token.md](issue-grammar-seed-token.md) | `uzu-engine` grammar | 空输入续生成时 seed token 的 grammar 拒绝错误被丢弃 |
| [issue-language-stream-invalid-state.md](issue-language-stream-invalid-state.md) | `uzu-engine` stream | 错误路径可能进入 `Invalid`，再次调用 panic |
| [issue-language-state-options-leak.md](issue-language-state-options-leak.md) | `uzu-engine` state | 旧模型状态与新 sampling/grammar 配置混合 |
| [issue-harmony-global-env-race.md](issue-harmony-global-env-race.md) | `hanashi` Harmony tokenizer | 进程级环境变量竞态和 tokenizer 加载污染 |
| [issue-chat-session-error-recovery.md](issue-chat-session-error-recovery.md) | `nagare` ChatSession | 工具调用错误路径可能永久停留 `ToolCalling` |
| [issue-engine-catalog-callback-deadlock.md](issue-engine-catalog-callback-deadlock.md) | `uzu` Engine catalog | 持发布锁执行用户 callback 可能死锁 |
| [issue-download-delete-silent-partial.md](issue-download-delete-silent-partial.md) | `download-manager` | 多文件模型部分删除后报告整体失败 |

## P2

| 文档 | 模块 | 核心风险 |
| --- | --- | --- |
| [issue-openai-sampling-accuracy.md](issue-openai-sampling-accuracy.md) | `backend-remote` OpenAI | `top_k` / `min_p` / repetition 语义静默丢失 |
| [issue-openai-finish-tool-call-mapping.md](issue-openai-finish-tool-call-mapping.md) | `backend-remote` OpenAI stream | partial tool call 仍把 Stop 改写为 ToolCalls |
| [issue-openai-failure-status-mapping.md](issue-openai-failure-status-mapping.md) | `backend-remote` OpenAI Responses | provider 失败被映射为正常 Stop |
| [issue-attachment-storage-id-parsing.md](issue-attachment-storage-id-parsing.md) | `mirai-gui` attachments | 清理逻辑依赖 ID 字符串格式反推时间 |
| [issue-gated-act-mul-flaky-test.md](issue-gated-act-mul-flaky-test.md) | `uzu-engine` kernels | 被忽略的 BF16/Hadamard correctness 测试缺少诊断 |

## 排查方法

1. 使用 CodeGraph 追踪核心调用链：Language model stream、ChatSession tool loop、Engine catalog publication、Harmony tokenizer 加载、OpenAI request/stream bridge、download deletion。
2. 检查高置信度的显式 TODO、`#[ignore]`、`unreachable!()`、错误丢弃和锁内回调。
3. 对照现有测试，确认问题是否已有覆盖。
4. 搜索 GitHub 开放 issue，避免明显重复。

## 尚未创建 GitHub issue

这些文档是 issue 草案。创建正式 issue 前，建议维护者先确认以下设计意图：

- 空输入续生成是否是公开支持的 API 语义；
- OpenAI 后端采样降级应采用 strict error 还是 warning；
- OpenAI Responses `Failed` 是否需要保留 provider error details；
- Harmony tokenizer 加载是否可通过上游显式路径 API 修复。

