# CatalogCommitConflicts 后 Metadata 清理问题分析总结

## 问题概述

当发生 `CatalogCommitConflicts` 错误时（"one or more requirements failed. The client may retry."），**已写入的 staged metadata 文件没有被及时清理**，导致：

1. 存储空间浪费
2. 潜在的存储成本增加（云存储）
3. 孤立的 metadata 文件

## 问题分析

### 受影响的 Catalog 实现

| Catalog | 状态 | 文件位置 | 问题描述 |
|---------|------|----------|----------|
| **Glue Catalog** | ❌ 存在问题 | `crates/catalog/glue/src/catalog.rs:783-842` | 先写入 metadata，后更新 Glue；冲突时未清理 |
| **S3Tables Catalog** | ❌ 存在问题 | `crates/catalog/s3tables/src/catalog.rs:600-644` | 先写入 metadata，后更新 S3Tables；冲突时未清理 |
| **REST Catalog** | ✅ 无问题 | `crates/catalog/rest/src/catalog.rs:898-962` | 不直接写入文件，由服务器处理 |
| **HMS Catalog** | ✅ 无问题 | `crates/catalog/hms/src/catalog.rs:606-611` | `update_table` 方法尚未实现 |

### 问题流程

```
1. Transaction.commit() 调用 catalog.update_table()
2. update_table() 执行：
   a. 计算新的 staged_metadata_location
   b. 写入 metadata 文件到 staged_metadata_location  ← ⚠️ 文件已写入
   c. 尝试更新 catalog（Glue/S3Tables）
   d. 如果失败（CatalogCommitConflicts）：
      - 返回错误（标记为 retryable）
      - ❌ 但没有清理已写入的 metadata 文件
3. 重试机制会重试，但旧文件仍然存在
```

### 代码示例（Glue Catalog）

```rust
// crates/catalog/glue/src/catalog.rs:794-839

// ⚠️ 第 797-798 行：先写入 metadata 文件
staged_table
    .metadata()
    .write_to(staged_table.file_io(), staged_metadata_location)
    .await?;

// 第 821-839 行：尝试更新 Glue
let _ = builder.send().await.map_err(|e| {
    // ...
    UpdateTableError::ConcurrentModificationException(_) => {
        // ❌ 返回错误，但没有清理已写入的文件
        Error::new(ErrorKind::CatalogCommitConflicts, ...)
            .with_retryable(true)
    }
})?;
```

## 解决方案

### 修复方法

在错误处理中添加清理逻辑，当发生 `CatalogCommitConflicts` 时删除已写入的 staged metadata 文件：

```rust
UpdateTableError::ConcurrentModificationException(_) => {
    // ✅ 清理已写入的文件
    let file_io = staged_table.file_io();
    if let Err(cleanup_err) = file_io.delete(staged_metadata_location).await {
        eprintln!("Warning: Failed to cleanup staged metadata: {}", cleanup_err);
    }
    
    Error::new(ErrorKind::CatalogCommitConflicts, ...)
        .with_retryable(true)
}
```

### 需要修复的文件

1. ✅ `crates/catalog/glue/src/catalog.rs` - `update_table` 方法（第 828-832 行）
2. ✅ `crates/catalog/s3tables/src/catalog.rs` - `update_table` 方法（第 626-630 行）

### 修复验证

- ✅ FileIO trait 已提供 `delete()` 方法（`crates/iceberg/src/io/file_io.rs:91`）
- ✅ 修复方案可行，无需额外依赖

## 相关代码位置

- Transaction retry 机制: `crates/iceberg/src/transaction/mod.rs:160-184`
- Error 定义: `crates/iceberg/src/error.rs:65`
- FileIO delete 方法: `crates/iceberg/src/io/file_io.rs:91`

## 详细文档

- **完整分析**: 参见 `METADATA_CLEANUP_ANALYSIS.md`
- **修复建议**: 参见 `FIX_SUGGESTION.md`

## 建议

1. **立即修复**: Glue 和 S3Tables catalog 的 `update_table` 方法
2. **添加测试**: 验证清理逻辑在冲突场景下正常工作
3. **未来考虑**: HMS catalog 实现 `update_table` 时，确保包含清理逻辑
