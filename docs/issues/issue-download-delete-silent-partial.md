# P1: 多文件模型删除可能部分成功并向用户报告整体失败

## 元信息

| 项 | 内容 |
| --- | --- |
| Severity | P1 |
| 组件 | `download-manager` / Group download deletion |
| 主要文件 | `crates/legacy/download-manager/src/group_download_task.rs` |
| 关联文件 | `crates/legacy/download-manager/src/file_download/file_download_worker.rs` |
| 状态 | 待确认 / 待修复 |

## 概述

`GroupDownloadTask::delete` 会依次删除所有子文件任务：

```rust
let mut result = Ok(());
for child in self.children.iter() {
    result = result.and(Box::pin(child.delete()).await);
}
result
```

如果某个子任务删除成功，而后续子任务失败，函数会返回错误，但不会回滚已删除的文件，也不会向调用方返回“哪些文件已删除、哪些仍存在”的详情。

这会造成两个用户可见问题：

1. UI/调用方收到错误，认为删除失败。
2. 磁盘上实际处于部分删除状态，模型无法加载，也不能被当作完整模型继续管理。

在 GUI 的“取消下载/清理模型”场景中，这会留下不一致的模型目录。

## 关键路径

```rust
pub async fn delete(&self) -> Result<(), DownloadError> {
    self.stop_download_loop().await;
    if let Some(manager_id) = self.foreign_owner().await {
        return Err(LockError::LockedByOther {
            manager_id,
        }
        .into());
    }

    let mut result = Ok(());
    for child in self.children.iter() {
        result = result.and(Box::pin(child.delete()).await);
    }
    result
}
```

子文件删除：

```rust
async fn delete(&mut self) -> Result<(), FileDownloadError> {
    self.wants_download = false;
    let _lock = ...;
    self.backend.remove_files(&self.config).await?;
    self.set(DownloadPhase::NotDownloaded {}, 0, None);
    Ok(())
}
```

一旦 `remove_files` 失败，`set(NotDownloaded)` 不会执行；但之前成功的兄弟文件已经被删除。

## 影响

### 数据与状态不一致

- 模型 checkpoint 缺少文件。
- `DownloadState` 可能对部分子任务显示 `NotDownloaded`，对失败子任务保持原状态。
- 组任务状态由子任务聚合，可能呈现错误或不可预测状态。

### 用户操作误导

- 用户点击删除，收到失败，再点删除，可能只删除剩余文件，而 UI 无法明确显示“部分已删除”。
- 用户以为模型还在，但实际不可加载。

### 清理功能风险

GUI 清理逻辑会对 `Downloaded` / `Error` 状态执行删除。部分删除后，状态可能变成 `NotDownloaded`，导致该模型从清理列表中消失，但残留目录仍占磁盘。

## 建议修复

### 1. 返回结构化结果

```rust
pub struct GroupDeleteResult {
    pub deleted: Vec<DownloadId>,
    pub failed: Vec<(DownloadId, DownloadError)>,
}
```

调用方可以据此展示“已删除 3/5 个文件”。

### 2. 尽力恢复一致状态

删除失败时，将整个 group 标记为：

```text
DownloadPhase::Error {
    message: "Partial delete: some model files could not be removed"
}
```

这样 GUI 可以继续重试，不会把残留目录当作已清理。

### 3. 先统一校验，后执行删除

删除前检查所有子任务：

- 是否有 foreign lock；
- destination 是否可写；
- parent 目录权限是否足够。

不能完全消除文件系统错误，但可以减少“第 1 个成功、第 2 个权限失败”的常见路径。

### 4. 重试策略

对文件删除失败提供有限次 retry、每个文件的具体错误和可选的事务式删除。如果维护者认为 delete 不需要事务语义，也必须明确文档并返回部分成功信息。

## 建议测试

1. **second child delete fails**
   - 构造两个子文件。
   - 第一个可删，第二个因为权限或模拟 backend 失败。
   - 断返回结构化错误，group 状态不是“完全未删除”。

2. **retry after partial delete**
   - 第一次部分失败。
   - 修复权限。
   - 再次 delete。
   - 断剩余文件被删除，最终 `NotDownloaded`。

3. **foreign lock prevents any deletion**
   - 任一文件被锁时，断不会删除任何子文件。

4. **state remains visible after failure**
   - 部分失败后查询 `state()`。
   - 断可重试状态并包含错误信息。

## 验证命令

```bash
cargo +nightly nextest run -p download-manager
cargo +nightly nextest run -p uzu storage
cargo tools test rust
```

## 优先级理由

删除操作直接涉及用户数据和磁盘状态。静默部分成功会让 GUI、引擎和用户对同一模型产生三种不同认知，后续清理与重试都会被误导。

