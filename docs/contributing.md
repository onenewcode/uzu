# Uzu 贡献入门指南

这份指南面向第一次向 `trymirai/uzu` 贡献、或想在动手前快速建立全局认识的贡献者。它基于当前仓库的配置与 CI 整理，帮你从环境准备、代码定位、修改验证，一直走到提交 PR。

## 0. 五分钟理解项目

**uzu 是什么**：一个高性能本地推理引擎，核心由 Rust 实现，并提供 Python、Swift、TypeScript 绑定。它支持 Metal、WebGPU、CPU 后端，目标是让应用直接部署 AI 模型，获得低延迟、数据隐私和零推理成本。

**仓库结构**：

| 路径 | 内容 | 适合的贡献 |
| --- | --- | --- |
| `crates/uzu-engine/` | 新核心引擎，包含 backend、kernel、config、parameters、speculators 等 | 性能、算子、后端、核心模型支持 |
| `crates/uzu-engine-macros/` | 核心引擎的 Rust 过程宏 | 编译期代码生成 |
| `crates/legacy/` | 现阶段仍在使用的历史实现，包括对外 API、CLI、下载、多语言绑定、tokenizer/parser 等 | 大多数日常功能和修复 |
| `apps/mirai-gui/` | Tauri + React 桌面应用 | UI、交互、前端测试 |
| `apps/uzu-pi-extension/` | Pi coding agent 扩展 | 扩展集成 |
| `apps/playground/` | Swift playground | Swift 示例与验证 |
| `scripts/benchmarks/` | benchmark 任务与引擎对比 | 性能测试与报告 |
| `docs/` | Mintlify 文档站源文件 | 文档、示例、快速上手 |
| `.github/` | 贡献规则、CODEOWNERS、CI/CD 工作流 | CI、发布、自动化 |

**当前重要背景**：仓库同时维护新 `uzu-engine` 与 `crates/legacy/`。不要因为目录名叫 legacy 就跳过它——对外 API、CLI 和绑定仍然在那里，很多实际问题都应该从 legacy 侧修改或接线。

## 1. 准备你的 Fork 与本地分支

当前仓库已经配置了两个远端，适合直接贡献：

```bash
git remote -v
# origin    git@github.com:onenewcode/uzu.git
# upstream  git@github.com:trymirai/uzu.git
```

推荐流程：

```bash
git switch main
git fetch upstream
git merge --ff-only upstream/main
git switch -c my-feature
```

约定：

- 分支名建议体现意图，例如 `fix-metal-kernel-sync`、`docs-contributing-guide`。
- 小 PR 优先；一个 PR 聚焦一个问题。
- 不要把格式化无关文件混进功能 PR。
- 提交前确认 `git status --short` 只包含你有意修改的文件。

## 2. 安装环境

### 2.1 推荐方式：`cargo tools setup`

项目提供了统一工具入口：

```bash
cargo tools setup
```

它会安装或更新 rustup、uv、pnpm 及所需 Rust target。仓库的 Cargo alias 定义在 `.cargo/config.toml`：

```toml
[alias]
tools = "run --quiet -p cli-tools --"
```

如果你在 Apple 平台开发 Metal，需要额外平台工具时可以执行：

```bash
cargo tools setup --include-platform-specific
```

这会处理 Xcode 首次启动、Metal toolchain、CMake 等依赖。

### 2.2 Rust 版本

`rust-toolchain.toml` 指定 `nightly`，并包含所有主要 target。常用检查建议直接使用：

```bash
cargo +nightly fmt --check
cargo +nightly clippy --workspace --all-targets --features uzu-engine/webgpu -- -D warnings
```

CI 对 Rust 代码使用 `-Dwarnings`，本地保持同样标准能减少返工。

### 2.3 Nix 用户

仓库提供 `flake.nix`，CI 也通过 Nix 环境运行。进入开发 shell：

```bash
nix develop
```

如果需要使用本机 Xcode 而不是 Nix toolchain，可以尝试：

```bash
nix develop .#host-xcode
```

### 2.4 多语言绑定依赖

按需安装：

```bash
cargo tools install python
cargo tools install swift
```

TypeScript 使用 pnpm，通常在对应包目录执行：

```bash
pnpm install --frozen-lockfile
```

## 3. 修改前先理解代码

本仓库已建立 CodeGraph 索引（`.codegraph/`）。定位符号、调用关系或理解某个实现时，优先使用它，而不是直接全局 grep。

命令行用法：

```bash
codegraph explore "Engine::chat 的调用链和相关实现"
```

常见问题可以这样问：

```bash
codegraph explore "ChatSession reply 的实现和调用路径"
codegraph explore "Metal backend 中 kernel 调度相关符号"
codegraph explore "TypeScript binding 如何导出 Engine"
```

如果只是找文本或文件，仍然可以使用 `rg`，例如：

```bash
rg "TODO|FIXME" crates apps scripts
rg -n "struct ChatSession" crates
rg --files apps/mirai-gui
```

修改建议：

1. 先用 CodeGraph 阅读调用链，确认应该改核心、legacy、绑定还是 CLI。
2. 再看 `.github/CODEOWNERS`，了解负责评审的模块负责人。
3. 找到相邻代码的测试，按同样风格补测试。
4. 避免“顺手重构”不相关代码；把可维护性建议拆成独立 issue/PR。

## 4. 找一个合适的贡献方向

### 4.1 从官方 issue 开始

查看当前开放 issue：

```bash
gh issue list --repo trymirai/uzu --state open --limit 30
```

如果你想处理某个 issue，先在 issue 下留言说明计划，避免重复工作。修复 PR 的描述中请写 `Fixes #issue-number`。

以下是截至 2026-10-30 仍开放的方向，适合阅读后判断是否匹配自己的能力：

| Issue | 主题 | 难度/适合人群 |
| --- | --- | --- |
| #1007 | Qwen 3.8 flash next 支持 | 模型支持/核心算法，偏高 |
| #1003 | Ubuntu headless + NVIDIA 40/50 GPU 官方支持 | 后端/平台部署，偏高 |
| #941 | semantic Router D3 model 支持 | 模型支持，偏中高 |
| #917 | Decision API | API 设计与实现，中等 |
| #865 | QTIP-S 在 M4 输出错误，疑似 race kernel 与 M5 MXU layout 冲突 | 并发/算子 correctness，高 |
| #824 | 每请求 reasoning/thinking token 预算 | API/采样行为，中等 |
| #823 | 视觉/图像输入支持 | 端到端模型能力，高 |
| #792 | 跨会话持久化 prefilled prompt state | 状态管理/性能，中高 |
| #791 | Experts Paging | 显存/内存调度，高 |

这些 issue 只是方向参考；难度会随上下文变化。对不了解的实现，不要直接提交自己无法完全解释的 PR。

### 4.2 从小型工作开始

适合第一贡献的切入点：

- 修正文档或示例中不准确的说明。
- 为没有覆盖的边界条件补测试。
- 改善错误信息、日志或类型标注。
- 清理一个明确 TODO，并说明行为影响。
- 在某个平台手动跑通构建，把失败原因整理成 issue。

不建议第一贡献就选择：

- 大范围架构迁移。
- 多平台后端重构。
- 新模型支持。
- 安全、发布、签名流程变更。

### 4.3 看 TODO，但不要只看 TODO

当前仓库约有 103 个 `TODO/FIXME` 标记：

```bash
rg -n "TODO|FIXME" --glob '!target' --glob '!Cargo.lock'
```

TODO 不是任务列表。处理前要确认：

- 它描述的是仍然有效的需求。
- 有明确验收标准。
- 不和正在进行的 issue/PR 冲突。

## 5. 常用构建、测试与检查命令

### 5.1 Rust

格式：

```bash
cargo +nightly fmt --check
```

Lint：

```bash
cargo +nightly clippy --workspace --all-targets --features uzu-engine/webgpu -- -D warnings
```

构建：

```bash
cargo tools build rust --targets apple
```

测试：

```bash
cargo tools test rust
```

如果你想按 CI 更细的方式运行：

```bash
cargo +nightly nextest run --profile=ci
```

注意 `Cargo.lock` 必须与 `Cargo.toml` 一致：

```bash
cargo metadata --locked --format-version=1 > /dev/null
git diff --exit-code -- Cargo.lock
```

### 5.2 多语言绑定

统一入口：

```bash
cargo tools build python
cargo tools test python
cargo tools build swift
cargo tools test swift
cargo tools build typescript
cargo tools test typescript
```

如果只改绑定，优先运行对应语言测试，而不是盲目全量构建。

### 5.3 GUI

`apps/mirai-gui/package.json` 定义了完整检查：

```bash
cd apps/mirai-gui
pnpm install --frozen-lockfile
pnpm check
```

它包含：

- TypeScript typecheck
- ESLint
- Vitest
- Rust lint/test

格式：

```bash
pnpm format:check
```

构建：

```bash
pnpm build
```

### 5.4 Uzu Pi 扩展

```bash
cd apps/uzu-pi-extension
pnpm install --frozen-lockfile
pnpm check
```

`check` 包含 typecheck、Node 测试和构建打包验证。

### 5.5 文档与同步检查

仓库要求多语言示例、README、版本号等生成内容保持同步：

```bash
cargo tools sync --check
```

如果修改了示例或版本相关配置，运行：

```bash
cargo tools sync
```

再检查 diff，确认生成文件符合预期。

### 5.6 跨语言验证

CI 中的 `cargo tools verify` 会构建各语言并检查构建后工作树是否出现未提交变化：

```bash
cargo tools verify
```

它通常比语言测试更重，适合作提交前最终检查。

## 6. 提交前检查清单

按影响范围选择，但至少完成前四项：

```bash
git status --short
cargo +nightly fmt --check
cargo +nightly clippy --workspace --all-targets --features uzu-engine/webgpu -- -D warnings
cargo tools sync --check
```

按模块追加：

- Rust：`cargo tools test rust`
- Python：`cargo tools test python`
- Swift：`cargo tools test swift`
- TypeScript：`cargo tools test typescript`
- GUI：`cd apps/mirai-gui && pnpm check`
- Pi 扩展：`cd apps/uzu-pi-extension && pnpm check`

如果 CI 失败但你本地缺少环境，把失败信息、平台、已执行命令写到 PR 描述中，方便维护者复现。

## 7. 提交 PR

项目硬性贡献规则在 `.github/CONTRIBUTING.md`，尤其是：

1. PR 标题和描述必须人工撰写。
2. 你必须完全理解自己提交的 PR 内容；不理解时先开 issue 讨论。
3. 第一次贡献者必须在 PR 描述末尾声明已阅读该贡献规则。

PR 描述建议包含：

```markdown
## Summary
- Explain what changed and why.
- Link the issue or discussion.

## Implementation notes
- Where the change lives.
- Any design tradeoffs.

## Verification
- Commands run and their results.
- Platforms tested.
```

第一次贡献者在末尾添加：

```markdown
I have read the contributing guide.
```

提交：

```bash
git add <specific-files>
git commit
git push -u origin my-feature
```

然后从你的 fork 分支向 `trymirai/uzu:main` 创建 PR。可以使用：

```bash
gh pr create --repo trymirai/uzu --base main --head onenewcode:my-feature
```

## 8. 性能相关贡献

性能 PR 需要避免“看起来更快”的结论：

1. 说明硬件、后端、模型、输入和采样配置。
2. 使用相同输入多次运行。
3. 对照 release/debug 配置。
4. 报告均值与波动，而不是单次最优值。
5. 如适用，使用 `scripts/benchmarks/` 或 CI 的 `Run Benchmark` 工作流。

本地参考：

```bash
cargo run --release -p cli -- bench {MODEL_PATH} {TASK_PATH} {OUTPUT_PATH}
```

不要提交大型模型文件或私人 benchmark 输出；只保留必要、可复现的摘要或任务描述。

## 9. 模型格式与新模型支持

新模型支持通常不只是加一个名字。大致路径是：

1. 确认模型结构与 tokenizer 语义。
2. 通过 `lalamo` 导出/转换模型。
3. 在引擎中实现或复用对应算子与配置。
4. 用参考实现验证输出 correctness。
5. 补充分平台测试与性能数据。

参考：

```bash
git clone https://github.com/trymirai/lalamo.git
cd lalamo
uv run lalamo list-models
uv run lalamo convert meta-llama/Llama-3.2-1B-Instruct
```

这类工作建议先开设计 issue，拿到维护者方向确认后再写大量代码。

## 10. 安全、依赖与许可证

- 许可证是 MIT，贡献遵循仓库现有许可。
- `cargo deny` 配置在 `deny.toml`。
- 新增依赖前先确认许可证兼容、必要性和维护状态。
- 依赖更新应保持 `Cargo.lock` 一致，并跑完整 lint/test。

本地检查：

```bash
cargo deny --exclude mirai-gui check -D warnings
```

GUI 有单独检查：

```bash
cd apps/mirai-gui/desktop
cargo deny check -D warnings
```

## 11. 常见问题

### 我应该改 `uzu-engine` 还是 `legacy`？

先用 CodeGraph 追调用链。如果功能属于新核心引擎的后端/算子，改 `crates/uzu-engine/`；如果涉及对外 API、CLI、下载、绑定或大量现有功能，通常在 `crates/legacy/`。不确定时先开 issue 或在 Discord 讨论。

### `cargo tools test rust` 为什么很慢？

测试配置启用了优化，Rust 测试本身偏重，且部分平台会构建多 target。日常开发可以缩小到具体 crate 或测试名，最终再用 CI 等价命令验证。

### 我改了 README/示例，为什么 `cargo tools sync --check` 失败？

README、多语言示例和部分文档片段由 `.config/platforms.toml` 与生成任务同步。改源示例后运行 `cargo tools sync`，检查生成的 diff；不要手改生成片段。

### Metal/WebGPU 修改应该怎么验证？

至少验证一个对应后端和目标平台。Metal 修改要在 Apple 平台跑；WebGPU 修改需要跑 `uzu-engine/webgpu` 相关检查。涉及跨平台时，清楚说明你无法覆盖的平台。

### 我可以引入 AI 生成补丁吗？

可以借助工具，但 PR 标题、描述和最终代码必须由你理解并负责。项目明确要求 PR 内容不能是你无法解释的内容。

### PR 冲突了怎么办？

```bash
git fetch upstream
git rebase upstream/main
```

解决冲突后重新运行受影响检查，再 force push 到自己的分支。

## 12. 最小工作流速查

```bash
# 1. 保持 main 最新
git fetch upstream
git switch main
git merge --ff-only upstream/main

# 2. 新分支
git switch -c my-first-contribution

# 3. 定位代码
codegraph explore "相关符号或调用链"

# 4. 修改并补测试
# ...

# 5. 基础检查
git status --short
cargo +nightly fmt --check
cargo +nightly clippy --workspace --all-targets --features uzu-engine/webgpu -- -D warnings
cargo tools sync --check

# 6. 按范围测试
cargo tools test rust

# 7. 提交 PR
git add <specific-files>
git commit
git push -u origin my-first-contribution
```

## 13. 外部资源

- 官方文档：<https://docs.trymirai.com>
- Discord：<https://discord.com/invite/trymirai>
- 贡献规则：<https://github.com/trymirai/uzu/blob/main/.github/CONTRIBUTING.md>
- Issue 列表：<https://github.com/trymirai/uzu/issues>
- 模型转换工具：<https://github.com/trymirai/lalamo>

祝贡献顺利。如果不确定，优先开一个简短 issue 或在 Discord 提问，这比自己猜测实现方向更有效。
