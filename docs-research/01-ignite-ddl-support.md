# Apache Ignite 2.17.0 DDL 支持与 API 入口

> 研究对象:`vendors/ignite` 子模块(Apache Ignite,固定在 tag `2.17.0`)
> 研究范围:Ignite 支持哪些 DDL 操作、怎么使用、API 入口在哪
> 所有结论均落实到源码 file:line,可在本地 `vendors/ignite/` 下点击跳转验证。

---

## 0. 最关键的结论

Ignite 2.17 **同时存在两套 SQL 引擎**,DDL 支持范围差别很大:

| 引擎 | 注册类 | 状态 |
|---|---|---|
| **H2**(经典,默认) | `IndexingQueryEngineConfiguration` | 稳定,DDL 很窄 |
| **Calcite**(新版,可选) | `CalciteQueryEngineConfiguration`(`@IgniteExperimental`) | 实验性,DDL 宽 |

因此 `docs/_docs/sql-reference/ddl.adoc` 里列出的"完整 DDL 清单"(USER / VIEW / ANALYZE…)其实是 **Calcite 引擎**的能力;纯 H2 引擎只支持下面矩阵中标 ✅ 的 6 个核心操作。

---

## 1. DDL 支持矩阵

| 语句 | H2 引擎 | Calcite 引擎 |
|---|:--:|:--:|
| `CREATE TABLE` / `DROP TABLE` | ✅ | ✅(原生) |
| `ALTER TABLE ADD COLUMN` / `DROP COLUMN` | ✅ | ✅(原生) |
| `ALTER TABLE ALTER COLUMN`(改类型 / 重命名 / 默认值 / NOT NULL…) | ❌(解析阶段拒绝) | ❌ |
| `CREATE INDEX` / `DROP INDEX` | ✅ | ✅(转交 H2) |
| `CREATE USER` / `ALTER USER` / `DROP USER` | ❌ | ✅(转交 H2) |
| `CREATE VIEW` / `DROP VIEW` | ❌ | ✅(Calcite 原生) |
| `ANALYZE` / `REFRESH STATISTICS` / `DROP STATISTICS` | ❌ | ✅(原生) |
| `KILL QUERY/SCAN/SERVICE/COMPUTE/TRANSACTION` | ❌ | ✅(原生) |
| `COMMIT` / `ROLLBACK`(SQL 事务控制) | ❌ | ✅(原生) |
| `CREATE/DROP SCHEMA` | ❌ | ❌ |
| `CREATE/DROP SEQUENCE` | ❌ | ❌ |
| `CREATE/ALTER/DROP ZONE` | ❌ | ❌ |

> **没有 Ignite-3 的 `CREATE ZONE`**:在 2.17 源码里搜 `zone`,只有 `TIMESTAMP WITH TIME ZONE` 这类数据类型语法(见 `IgniteSqlParserImpl.java:6015`),不存在 Zone DDL 的解析方法 / AST 节点 / 转换分支。

---

## 2. 代码路径:DDL 从解析到执行

### 2.1 H2 引擎路径

**解析分派** —— `modules/indexing/src/main/java/org/apache/ignite/internal/processors/query/h2/sql/GridSqlQueryParser.java`

- 顶层分派:`parse(Prepared)` 在 `:1888`,只对 5 种 H2 AST 建分支;其余在 `:1926` 抛 `Unsupported statement`。
- CREATE TABLE 的 `WITH` 参数:18 个常量定义在 `:472-526`,未知参数在 `:1673` 抛 `Unsupported parameter`。

| 语句 | 解析方法(行) | AST 类 |
|---|---|---|
| CREATE INDEX | `parseCreateIndex` `:1020` | `GridSqlCreateIndex:28` |
| DROP INDEX | `parseDropIndex` `:1004` | `GridSqlDropIndex:25` |
| CREATE TABLE | `parseCreateTable` `:1077` | `GridSqlCreateTable:31` |
| DROP TABLE | `parseDropTable` `:1280` | `GridSqlDropTable:23` |
| ALTER COLUMN | `parseAlterColumn` `:1298`(ADD/DROP 之外的子类型在 `:1306-1330` 全部拒绝,提示 "ALTER COLUMN is not supported") | — |
| ADD COLUMN | `parseAddColumn` `:1398` | `GridSqlAlterTableAddColumn:23` |
| DROP COLUMN | `parseDropColumn` `:1448` | `GridSqlAlterTableDropColumn:23` |

**执行分派** —— `modules/indexing/.../h2/CommandProcessor.java` 的 `runCommandH2`(`:239`)按 AST 类型调用 `GridQueryProcessor` 的动态方法:

| 语句 | `CommandProcessor` 行 | `GridQueryProcessor` 方法(行) |
|---|---|---|
| CREATE TABLE | `:243`(→ `toQueryEntity` `:449`) | `dynamicTableCreate` `:2215`(若带 `CACHE_NAME=<existing>` 走 `dynamicAddQueryEntity` `:3495`) |
| DROP TABLE | `:299` | `dynamicTableDrop` `:2314` |
| ADD COLUMN | `:317` | `dynamicColumnAdd` `:3459` |
| DROP COLUMN | `:372` | `dynamicColumnRemove` `:3477` |
| CREATE / DROP INDEX | native:`SqlCreateIndexCommand` / `SqlDropIndexCommand` | `dynamicIndexCreate` `:3426` / `dynamicIndexDrop` `:3443` |

最终落到 `modules/core/.../schema/management/SchemaManager.java` 做真正的表 / 索引创建。

### 2.2 CREATE TABLE 的 `WITH` 参数

解析自 H2 `CreateTableData.tableEngineParams`,在 `GridSqlQueryParser.java:1185-1219` 拆分,由 `processExtraParam`(`:1481-1676`)分派:

| 常量 | 值 | 映射 |
|---|---|---|
| `PARAM_TEMPLATE` | `TEMPLATE` | 模板名(`TEMPLATE_PARTITIONED` / `TEMPLATE_REPLICATED`) |
| `PARAM_BACKUPS` | `BACKUPS` | 备份数 |
| `PARAM_ATOMICITY` | `ATOMICITY` | `CacheAtomicityMode` |
| `PARAM_CACHE_GROUP` | `CACHE_GROUP`(老名 `CACHEGROUP`) | 缓存组 |
| `PARAM_AFFINITY_KEY` | `AFFINITY_KEY`(老名 `AFFINITYkey`) | 仿射键列 |
| `PARAM_WRITE_SYNC` | `WRITE_SYNCHRONIZATION_MODE` | 写同步模式 |
| `PARAM_CACHE_NAME` | `CACHE_NAME` | 关联已有缓存 |
| `PARAM_KEY_TYPE` / `PARAM_VAL_TYPE` | `KEY_TYPE` / `VALUE_TYPE` | key/value 类型名 |
| `PARAM_WRAP_KEY` / `PARAM_WRAP_VALUE` | `WRAP_KEY` / `WRAP_VALUE` | 是否包装 |
| `PARAM_DATA_REGION` | `DATA_REGION` | 数据区 |
| `PARAM_ENCRYPTED` | `ENCRYPTED` | 加密 |
| `PARAM_PARALLELISM` | `PARALLELISM` | 查询并行度 |
| `PARAM_PK_INLINE_SIZE` / `PARAM_AFFINITY_INDEX_INLINE_SIZE` | `PK_INLINE_SIZE` / `AFFINITY_INDEX_INLINE_SIZE` | 索引内联大小 |

### 2.3 Calcite 引擎路径

入口 `CalciteQueryProcessor.query(...)`(`modules/calcite/.../calcite/CalciteQueryProcessor.java:397`)→ JavaCC 生成的 `IgniteSqlParserImpl` → 两个转换器:

- `DdlSqlToCommandConverter.java:156` —— 原生 DDL:CREATE/DROP TABLE、ADD/DROP COLUMN、COMMIT/ROLLBACK。
- `SqlToNativeCommandConverter.java:88` —— 转交 H2 执行:CREATE/DROP INDEX、CREATE/ALTER/DROP USER、ALTER TABLE(非 column 形式,如 LOGGING)、CREATE/DROP VIEW、KILL、ANALYZE/STATISTICS。

> Calcite 引擎对部分 DDL **不重新实现**,而是转换成 native command 交给经典 H2 引擎;因此 `README.txt:42-43` 说明使用 Calcite 时 `ignite-indexing`(H2)模块也必须在 classpath 上。

---

## 3. 怎么使用 + API 入口

### 3.1 核心 Java API —— `SqlFieldsQuery`

无单独 DDL 标志,DDL 字符串直接当普通 SQL 传入。

- 类:`modules/core/.../cache/query/SqlFieldsQuery.java` —— 构造 `:135`、`setSql` `:165`、`setArgs` `:188`、`setSchema` `:401`。
- 执行:`IgniteCache.query(SqlFieldsQuery)` —— `modules/core/.../IgniteCache.java:440`。
- ⚠️ 通过 `IgniteCache` 走 SQL **需要一个 "dummy cache" 作入口**(见 `examples/.../sql/SqlDdlExample.java:54-55` 注释)。

```java
IgniteCache<?, ?> cache = ignite.getOrCreateCache(
    new CacheConfiguration<>("dummy_cache").setSqlSchema("PUBLIC"));

// CREATE TABLE
cache.query(new SqlFieldsQuery(
    "CREATE TABLE city (id BIGINT PRIMARY KEY, name VARCHAR) " +
    "WITH \"template=replicated\"")).getAll();

// CREATE INDEX
cache.query(new SqlFieldsQuery(
    "CREATE INDEX person_idx ON Person (city_id)")).getAll();

// DROP TABLE
cache.query(new SqlFieldsQuery("DROP TABLE Person")).getAll();
```

### 3.2 其余三个入口

传输层不同,服务端执行路径相同,且**都不需要 dummy cache**:

| 入口 | 类 / 文件 | 用法 |
|---|---|---|
| **JDBC thin(推荐)** | `IgniteJdbcThinDriver`;URL 前缀 `JdbcThinUtils.URL_PREFIX = "jdbc:ignite:thin://"`(`:48`);默认端口 10800 | `DriverManager.getConnection("jdbc:ignite:thin://127.0.0.1/")` 后 `stmt.executeUpdate("CREATE TABLE …")` |
| **JDBC client(旧)** | `IgniteJdbcDriver`;`CFG_URL_PREFIX = "jdbc:ignite:cfg://"`(`:261`);内嵌 client 节点 | 较重(需整个 `libs/` 在 classpath),thin 优先 |
| **SQLLine CLI** | `modules/sqlline/bin/sqlline.sh`(`:70` 拼 thin URL) | `./sqlline.sh -u jdbc:ignite:thin://127.0.0.1/` |

### 3.3 切换到 Calcite 引擎

- **启动注册**:`SqlConfiguration.setQueryEnginesConfiguration(...)`(`modules/core/.../configuration/SqlConfiguration.java:196`),给 `CalciteQueryEngineConfiguration` 设 `setDefault(true)`;需把 `optional/ignite-calcite` 放进 `libs/`。
- **单语句 hint**:`SELECT /*+ QUERY_ENGINE('calcite') */ ...`(定义在 `modules/calcite/.../hint/HintDefinition.java:34`)。
- **JDBC 连接级**:`jdbc:ignite:thin://host:port?queryEngine=calcite`。

---

## 4. 参考文件索引(均在 `vendors/ignite/` 下)

- DDL 参考文档:`docs/_docs/sql-reference/ddl.adoc`
- SQL API 概览:`docs/_docs/SQL/sql-api.adoc`
- SQLLine 使用:`docs/_docs/tools/sqlline.adoc`
- 示例:`examples/src/main/java/org/apache/ignite/examples/sql/SqlDdlExample.java`(DDL 全流程)、`SqlJdbcExample.java`(JDBC 路径)
- 启用 SQL 的配置示例:`examples/config/example-sql.xml`
- Calcite 模块说明:`modules/calcite/README.txt`

---

## 5. 一句话总结

日常 DDL(`CREATE/DROP TABLE`、`ADD/DROP COLUMN`、`CREATE/DROP INDEX`)走默认 **H2 引擎**,API 入口是 `IgniteCache.query(new SqlFieldsQuery(sql))` 或 JDBC thin(`jdbc:ignite:thin://`);需要 `USER` / `VIEW` / `ANALYZE` / `KILL` / SQL 事务等更全的 DDL,就启用实验性 **Calcite 引擎**。
