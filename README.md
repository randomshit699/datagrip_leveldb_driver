# leveldb-jdbc（DataGrip / JDBC 驱动） 
**AI generate content**  

这是一个面向 JetBrains DataGrip 的 LevelDB JDBC 驱动。它把一个 LevelDB 数据库目录映射成一个模拟表 `main.kv`，便于在 DataGrip 里浏览、筛选、编辑 KV。

### 读写驱动

- Driver jar：`leveldb-jdbc-rw-0.1-all.jar`
- Driver class：`leveldb.jdbc.LevelDbDriver`
- JDBC URL：
  - `jdbc:leveldb:<leveldb_dir>`
  - 如果目录不存在并希望创建：`jdbc:leveldb:<leveldb_dir>?create=true`

说明：

- LevelDB 会对 DB 目录加锁；如果该目录已被其它进程占用，连接会直接失败（例如 `OverlappingFileLockException`/`LOCK` 相关异常）。

## 表与列

驱动只暴露 1 张表：

- `main.kv`

列（`SELECT *` 默认 4 列）：

- `key`：文本视图（UTF-8）。在 DataGrip 表格视图中可直接编辑（写入时按 UTF-8 编码成 bytes）。
- `value`：文本视图（UTF-8）。在 DataGrip 表格视图中可直接编辑（写入时按 UTF-8 编码成 bytes）。
- `key_hex`：只读展示列，`key` 的十六进制（小写）编码。
- `value_hex`：只读展示列，`value` 的十六进制（小写）编码。

提示：

- LevelDB 的真实存储是 bytes → bytes。若你的数据不是 UTF-8 文本，请用 `key_hex/value_hex` 来查看/定位，并用 `X'..'` 或 `*_hex` 列写入原始 bytes。

## 支持的 SQL（子集）

驱动只实现 DataGrip 常用的子集语法（大小写不敏感，标识符可用 `"` 或 `` ` `` 引号；表名可写 `kv`、`main.kv`、`"main"."kv"`）。

### SELECT

支持：

- `SELECT 1`
- `SELECT * FROM kv [WHERE ...] [ORDER BY ...] [LIMIT n] [OFFSET n]`
- `SELECT key, value, key_hex, value_hex FROM kv ...`（列可任意组合/别名）
- `SELECT COUNT(*) [AS alias] FROM kv [WHERE ...]`

WHERE（仅支持按 key 范围/等值）：

- `WHERE key = 'text'`
- `WHERE key_hex = '0011aabb'`
- `WHERE key LIKE 'prefix%'`
- `WHERE key_hex LIKE '0011%'`
- `WHERE 1=1` / `WHERE 1=0` / `WHERE true` / `WHERE false`

说明：

- `ORDER BY` 语法会被接受但目前不参与执行计划；结果天然按 LevelDB key 的排序顺序输出。

示例：

```sql
SELECT * FROM main.kv LIMIT 100;
SELECT key_hex, value_hex FROM kv WHERE key_hex LIKE '6b%' LIMIT 10;
SELECT COUNT(*) AS c FROM kv WHERE key LIKE 'user:%';
```

### INSERT

语法：

```sql
INSERT INTO kv [(col1, col2, ...)] VALUES (expr1, expr2, ...);
```

列名只支持：`key` / `value` / `key_hex` / `value_hex`。

表达式（expr）支持：

- `?`（PreparedStatement 参数）
- `NULL`（`value`/`value_hex` 中会被写成空 bytes；`key`/`key_hex` 不允许为 NULL）
- `'text'`（对 `key/value` 按 UTF-8 写入；对 `*_hex` 按十六进制解码写入）
- `X'0011AABB'`（写入原始 bytes）

当同时提供 `key` 与 `key_hex`（或 `value` 与 `value_hex`）时：

- 优先使用 `key/value`；只有当 `key/value` 缺失或为 `NULL` 时，才会回退使用 `*_hex`。

示例：

```sql
INSERT INTO kv(key, value) VALUES ('k1', 'v1');
INSERT INTO kv(key_hex, value_hex) VALUES ('6b31', '7631'); -- bytes: "k1" / "v1"
INSERT INTO kv(key, value) VALUES ('k2', X'000102FF');      -- value 写入原始 bytes
```

### UPDATE

语法（两种 SET 形式都支持；可带表别名；可带 `RETURNING ...` 但会被忽略）：

```sql
UPDATE main.kv t SET t."value" = 'v2' WHERE t."key" = 'k1';
UPDATE kv SET (value_hex) = ('00ff') WHERE key_hex = '6b31';
```

WHERE：

- 必须包含 `key = ...` 或 `key_hex = ...`
- 可选乐观锁：`AND value = ...` 或 `AND value_hex = ...`

示例：

```sql
UPDATE kv SET value = 'v2' WHERE key = 'k1';
UPDATE kv SET value = 'v2' WHERE key = 'k1' AND value = 'v1';
UPDATE kv SET value_hex = '00ff' WHERE key_hex = '6b31';
```

### DELETE

语法（可带表别名；可带 `RETURNING ...` 但会被忽略）：

```sql
DELETE FROM main.kv t WHERE t."key" = 'k1';
DELETE FROM kv WHERE key_hex = '6b31';
```

WHERE：

- 必须包含 `key = ...` 或 `key_hex = ...`
- 可选乐观锁：`AND value = ...` 或 `AND value_hex = ...`

示例：

```sql
DELETE FROM kv WHERE key = 'k1';
DELETE FROM kv WHERE key_hex = '6b31' AND value_hex = '7631';
```

## 不支持的内容（当前）

- 多表 / JOIN / 子查询 / 聚合（除 `COUNT(*)`）/ DDL
- 按 `value` 条件进行 UPDATE/DELETE（WHERE 必须包含 key）
- 事务语义（`commit/rollback` 为 no-op；结果按 LevelDB 的即时写入表现）
- 完整的 JDBC 元数据与全部 SQL 方言

