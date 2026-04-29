# Zero-Downtime Database Migrations Guide

The Frugal Fortress requires 24/7 uptime. Therefore, all database schema changes managed via `Alembic` must be strictly **Zero-Downtime** compatible.

## 1. The Golden Rules of Schema Evolution

1.  **Additive Changes Only:** You may add tables, add columns, or add non-blocking indexes. You may **never** drop a column, drop a table, or rename a column in a single deployment.
2.  **The 3-Phase Drop:** To remove or rename a column, you must follow three deployments:
    *   *Phase 1:* Add the new column. Update the application to write to both the old and new columns, but read from the old.
    *   *Phase 2:* Backfill data. Update the application to read from the new column.
    *   *Phase 3:* Stop writing to the old column. Drop the old column in the database.
3.  **No Default Values on Existing Large Tables:** Adding a column with a `DEFAULT` value to a table with millions of rows (e.g., `feedback_log`) will lock the table and cause an outage. 
    *   *Fix:* Add the column as `NULL`, backfill the data asynchronously, and then apply the `NOT NULL` constraint.

## 2. Indexing Safely
Creating an index on a large table blocks writes in PostgreSQL.
*   **Rule:** Always use `CREATE INDEX CONCURRENTLY` for production migrations.
*   *Note:* Alembic does not support concurrent index creation inside a transaction. You must set `op.get_bind().execution_options(isolation_level='AUTOCOMMIT')` in your migration script.

## 3. Vector Indexes (pgvector)
When creating HNSW indexes for `pgvector` (e.g., on `issuer_embedding`), ensure the `m` and `ef_construction` parameters are tuned for our specific RAM constraints. Re-indexing vectors is CPU-intensive and should be scheduled during low-traffic windows.

## 4. Running Migrations
Migrations are automatically applied during the CI/CD pipeline before the new application containers are spun up.
```bash
# Generate a new migration locally
make migration msg="add_payg_flag_to_users"

# Apply locally to test
make migrate
```