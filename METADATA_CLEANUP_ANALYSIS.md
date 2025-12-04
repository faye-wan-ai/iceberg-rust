# CatalogCommitConflicts 后 Metadata 文件清理分析

## 问题描述

当发生 `CatalogCommitConflicts` 错误时（"one or more requirements failed. The client may retry."），需要分析后续流程中 metadata 文件是否及时清理。

## 代码流程分析

### 1. Transaction Commit 流程

**文件**: `crates/iceberg/src/transaction/mod.rs`

```rust
pub async fn commit(self, catalog: &dyn Catalog) -> Result<Table> {
    // ...
    (|mut tx: Transaction| async {
        let result = tx.do_commit(catalog).await;
        (tx, result)
    })
    .retry(backoff)
    .sleep(tokio::time::sleep)
    .context(tx)
    .when(|e| e.retryable())
    .await
    .1
}

async fn do_commit(&mut self, catalog: &dyn Catalog) -> Result<Table> {
    // ...
    catalog.update_table(table_commit).await
}
```

**关键点**:
- `CatalogCommitConflicts` 错误被标记为 `retryable(true)`
- 重试机制会自动重试，但**不会清理已写入的文件**

### 2. 各 Catalog 实现的 update_table 方法

#### 2.1 REST Catalog ✅ (无问题)

**文件**: `crates/catalog/rest/src/catalog.rs:898-962`

```rust
async fn update_table(&self, mut commit: TableCommit) -> Result<Table> {
    // 发送 updates 和 requirements 到服务器
    // 服务器负责写入 metadata 文件
    // 如果返回 CONFLICT，没有本地文件需要清理
}
```

**结论**: REST Catalog 不直接写入文件，由服务器处理，无清理问题。

#### 2.2 Glue Catalog ❌ (存在问题)

**文件**: `crates/catalog/glue/src/catalog.rs:783-842`

```rust
async fn update_table(&self, commit: TableCommit) -> Result<Table> {
    // ...
    let staged_table = commit.apply(current_table)?;
    let staged_metadata_location = staged_table.metadata_location_result()?;

    // ⚠️ 问题：先写入 metadata 文件
    staged_table
        .metadata()
        .write_to(staged_table.file_io(), staged_metadata_location)
        .await?;

    // 然后尝试更新 Glue catalog
    let _ = builder.send().await.map_err(|e| {
        // ...
        UpdateTableError::ConcurrentModificationException(_) => Error::new(
            ErrorKind::CatalogCommitConflicts,
            format!("Commit failed for table: {table_ident}"),
        )
        .with_retryable(true),
        // ...
    })?;

    Ok(staged_table)
}
```

**问题**:
- 第 797-798 行：metadata 文件已写入到 `staged_metadata_location`
- 第 821-839 行：如果 Glue 更新失败（`ConcurrentModificationException`），返回 `CatalogCommitConflicts`
- **已写入的 metadata 文件没有被清理**

#### 2.3 S3Tables Catalog ❌ (存在问题)

**文件**: `crates/catalog/s3tables/src/catalog.rs:600-644`

```rust
async fn update_table(&self, commit: TableCommit) -> Result<Table> {
    // ...
    let staged_table = commit.apply(current_table)?;
    let staged_metadata_location = staged_table.metadata_location_result()?;

    // ⚠️ 问题：先写入 metadata 文件
    staged_table
        .metadata()
        .write_to(staged_table.file_io(), staged_metadata_location)
        .await?;

    // 然后尝试更新 S3Tables
    let _ = builder.send().await.map_err(|e| {
        // ...
        UpdateTableMetadataLocationError::ConflictException(_) => Error::new(
            ErrorKind::CatalogCommitConflicts,
            format!("Commit conflicted for table: {table_ident}"),
        )
        .with_retryable(true),
        // ...
    })?;

    Ok(staged_table)
}
```

**问题**:
- 第 609-612 行：metadata 文件已写入
- 第 623-641 行：如果 S3Tables 更新失败（`ConflictException`），返回 `CatalogCommitConflicts`
- **已写入的 metadata 文件没有被清理**

#### 2.4 HMS Catalog ✅ (无问题)

**文件**: `crates/catalog/hms/src/catalog.rs:606-611`

```rust
async fn update_table(&self, _commit: TableCommit) -> Result<Table> {
    Err(Error::new(
        ErrorKind::FeatureUnsupported,
        "Updating a table is not supported yet",
    ))
}
```

**结论**: HMS Catalog 的 `update_table` 方法尚未实现，暂不存在此问题。

## 问题影响

1. **存储空间浪费**: 孤立的 metadata 文件占用存储空间
2. **存储成本**: 在云存储（S3、GCS）中可能产生不必要的成本
3. **文件管理**: 不符合 Iceberg 规范的最佳实践
4. **潜在问题**: 如果重试成功，新文件会覆盖旧文件，但旧文件仍然存在

## 解决方案建议

### 方案 1: 在错误处理中添加清理逻辑（推荐）

在 `update_table` 方法中，当发生 `CatalogCommitConflicts` 错误时，清理已写入的 staged metadata 文件：

```rust
async fn update_table(&self, commit: TableCommit) -> Result<Table> {
    let staged_table = commit.apply(current_table)?;
    let staged_metadata_location = staged_table.metadata_location_result()?;

    // 写入 metadata 文件
    staged_table
        .metadata()
        .write_to(staged_table.file_io(), staged_metadata_location)
        .await?;

    // 尝试更新 catalog
    let result = builder.send().await.map_err(|e| {
        let error = e.into_service_error();
        match error {
            UpdateTableError::ConcurrentModificationException(_) => {
                // ⚠️ 清理已写入的文件
                let file_io = staged_table.file_io();
                if let Err(cleanup_err) = file_io.delete(staged_metadata_location).await {
                    // 记录清理失败，但不影响错误返回
                    eprintln!("Failed to cleanup staged metadata: {}", cleanup_err);
                }
                
                Error::new(
                    ErrorKind::CatalogCommitConflicts,
                    format!("Commit failed for table: {table_ident}"),
                )
                .with_retryable(true)
            }
            // ...
        }
    })?;

    Ok(staged_table)
}
```

### 方案 2: 使用事务性写入（如果存储支持）

某些存储系统支持事务性写入，可以确保原子性。

### 方案 3: 延迟写入策略

只有在 catalog 更新成功后才写入 metadata 文件（但这可能不符合某些 catalog 的实现要求）。

## 需要修复的文件

1. ✅ `crates/catalog/glue/src/catalog.rs` - `update_table` 方法
2. ✅ `crates/catalog/s3tables/src/catalog.rs` - `update_table` 方法
3. ⚠️ `crates/catalog/hms/src/catalog.rs` - 需要检查 `update_table` 方法

## 测试建议

1. 编写单元测试，模拟 `CatalogCommitConflicts` 错误
2. 验证 staged metadata 文件是否被正确清理
3. 验证重试机制仍然正常工作
4. 验证清理失败不影响错误传播

## 相关代码位置

- Transaction retry: `crates/iceberg/src/transaction/mod.rs:160-184`
- Glue Catalog: `crates/catalog/glue/src/catalog.rs:783-842`
- S3Tables Catalog: `crates/catalog/s3tables/src/catalog.rs:600-644`
- REST Catalog: `crates/catalog/rest/src/catalog.rs:898-962`
- Error definition: `crates/iceberg/src/error.rs:65`
