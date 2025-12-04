# Metadata 清理修复建议

## 修复方案

当发生 `CatalogCommitConflicts` 错误时，需要在错误处理中添加清理逻辑，删除已写入的 staged metadata 文件。

## 修复示例代码

### 1. Glue Catalog 修复

**文件**: `crates/catalog/glue/src/catalog.rs`

```rust
async fn update_table(&self, commit: TableCommit) -> Result<Table> {
    let table_ident = commit.identifier().clone();
    let table_namespace = validate_namespace(table_ident.namespace())?;

    let (current_table, current_version_id) =
        self.load_table_with_version_id(&table_ident).await?;
    let current_metadata_location = current_table.metadata_location_result()?.to_string();

    let staged_table = commit.apply(current_table)?;
    let staged_metadata_location = staged_table.metadata_location_result()?;

    // Write new metadata
    staged_table
        .metadata()
        .write_to(staged_table.file_io(), staged_metadata_location)
        .await?;

    // Persist staged table to Glue with optimistic locking
    let mut builder = self
        .client
        .0
        .update_table()
        .database_name(table_namespace)
        .set_skip_archive(Some(true))
        .table_input(convert_to_glue_table(
            table_ident.name(),
            staged_metadata_location.to_string(),
            staged_table.metadata(),
            staged_table.metadata().properties(),
            Some(current_metadata_location),
        )?);

    // Add VersionId for optimistic locking
    if let Some(version_id) = current_version_id {
        builder = builder.version_id(version_id);
    }

    let builder = with_catalog_id!(builder, self.config);
    let _ = builder.send().await.map_err(|e| {
        let error = e.into_service_error();
        match error {
            UpdateTableError::EntityNotFoundException(_) => Error::new(
                ErrorKind::TableNotFound,
                format!("Table {table_ident} is not found"),
            ),
            UpdateTableError::ConcurrentModificationException(_) => {
                // ⚠️ 修复：清理已写入的 staged metadata 文件
                let file_io = staged_table.file_io();
                if let Err(cleanup_err) = file_io.delete(staged_metadata_location).await {
                    // 记录清理失败，但不影响错误返回
                    // 可以考虑使用 tracing 或 log crate 记录警告
                    eprintln!(
                        "Warning: Failed to cleanup staged metadata file {}: {}",
                        staged_metadata_location, cleanup_err
                    );
                }
                
                Error::new(
                    ErrorKind::CatalogCommitConflicts,
                    format!("Commit failed for table: {table_ident}"),
                )
                .with_retryable(true)
            }
            _ => Error::new(
                ErrorKind::Unexpected,
                format!("Operation failed for table: {table_ident} for hitting aws sdk error"),
            ),
        }
        .with_source(anyhow!("aws sdk error: {error:?}"))
    })?;

    Ok(staged_table)
}
```

### 2. S3Tables Catalog 修复

**文件**: `crates/catalog/s3tables/src/catalog.rs`

```rust
async fn update_table(&self, commit: TableCommit) -> Result<Table> {
    let table_ident = commit.identifier().clone();
    let table_namespace = table_ident.namespace();
    let (current_table, version_token) =
        self.load_table_with_version_token(&table_ident).await?;

    let staged_table = commit.apply(current_table)?;
    let staged_metadata_location = staged_table.metadata_location_result()?;

    staged_table
        .metadata()
        .write_to(staged_table.file_io(), staged_metadata_location)
        .await?;

    let builder = self
        .s3tables_client
        .update_table_metadata_location()
        .table_bucket_arn(&self.config.table_bucket_arn)
        .namespace(table_namespace.to_url_string())
        .name(table_ident.name())
        .version_token(version_token)
        .metadata_location(staged_metadata_location);

    let _ = builder.send().await.map_err(|e| {
        let error = e.into_service_error();
        match error {
            UpdateTableMetadataLocationError::ConflictException(_) => {
                // ⚠️ 修复：清理已写入的 staged metadata 文件
                let file_io = staged_table.file_io();
                if let Err(cleanup_err) = file_io.delete(staged_metadata_location).await {
                    eprintln!(
                        "Warning: Failed to cleanup staged metadata file {}: {}",
                        staged_metadata_location, cleanup_err
                    );
                }
                
                Error::new(
                    ErrorKind::CatalogCommitConflicts,
                    format!("Commit conflicted for table: {table_ident}"),
                )
                .with_retryable(true)
            }
            UpdateTableMetadataLocationError::NotFoundException(_) => Error::new(
                ErrorKind::TableNotFound,
                format!("Table {table_ident} is not found"),
            ),
            _ => Error::new(
                ErrorKind::Unexpected,
                "Operation failed for hitting aws sdk error",
            ),
        }
        .with_source(anyhow::Error::msg(format!("aws sdk error: {error:?}")))
    })?;

    Ok(staged_table)
}
```

## 关键修改点

1. **在错误处理中添加清理逻辑**: 当检测到 `CatalogCommitConflicts` 错误时，调用 `file_io.delete()` 删除已写入的 staged metadata 文件。

2. **错误处理策略**: 
   - 如果清理失败，记录警告但不影响错误返回
   - 确保错误仍然正确传播，以便重试机制可以正常工作

3. **注意事项**:
   - 清理操作应该是异步的（使用 `await`）
   - 清理失败不应该阻止错误返回
   - 可以考虑使用更完善的日志记录机制（如 `tracing` crate）

## 测试建议

1. **单元测试**: 模拟 `CatalogCommitConflicts` 错误，验证文件是否被清理
2. **集成测试**: 测试重试机制是否仍然正常工作
3. **边界情况**: 测试清理失败时的行为

## 相关文件

- `crates/catalog/glue/src/catalog.rs:783-842`
- `crates/catalog/s3tables/src/catalog.rs:600-644`
- `crates/iceberg/src/io/mod.rs` - FileIO trait 定义
