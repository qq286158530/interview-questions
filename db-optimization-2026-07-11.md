# 数据库性能优化面试题

> 📅 更新时间：2026-07-11  
> 📚 来源：精选自网络公开技术文章与经典面试题库

---

## 题目一：MySQL 查询执行流程是什么？

### 参考答案

MySQL 查询执行的基本流程如下：

1. **连接层**：客户端通过连接器（Connector）建立 TCP 连接，验证用户身份，维持连接。

2. **查询缓存（MySQL 8.0 之前）**：Server 层先检查查询缓存是否已有该查询的结果（以 SQL 文本为 key）。命中则直接返回，绕过解析/优化阶段。注意 MySQL 8.0 已移除此功能。

3. **解析器（Parser）**：对 SQL 进行词法分析（Lex）和语法分析，生成**解析树（AST）**。

4. **预处理器（Preprocessor）**：检查表/列是否存在、权限是否足够，生成新的解析树。

5. **查询优化器（Optimizer）**：根据**成本模型（CBO）**选择最优执行计划。核心工作包括：
   - 决定 `ORDER BY`、`GROUP BY` 等子句使用索引的顺序
   - 决定 `JOIN` 的顺序（嵌套循环连接、哈希连接等）
   - 决定是否使用索引覆盖扫描
   - 将子查询改写为联接（JOIN）等价形式

6. **执行器（Executor）**：调用存储引擎接口，按执行计划逐行获取数据（Index Range Scan / Full Table Scan 等），返回结果集。

7. **结果返回**：如果需要排序则调用 `filesort`，最后将结果返回客户端。

> 📎 来源：https://dev.mysql.com/doc/refman/8.0/en/mysql-processing.html

---

## 题目二：如何判断 SQL 是否使用了索引？

### 参考答案

判断 SQL 是否使用索引，常用以下几种方法：

**① EXPLAIN 查看执行计划**

```sql
EXPLAIN SELECT * FROM users WHERE name = '张三';
```

关键字段说明：

| 字段 | 含义 |
|------|------|
| `type` | 连接类型，性能从优到差：`const > eq_ref > ref > range > index > ALL` |
| `key` | 实际使用的索引名 |
| `rows` | 预估扫描行数，越少越好 |
| `Extra` | 附加信息，常见值：`Using index`（覆盖索引）、`Using filesort`（需额外排序）、`Using temporary`（使用临时表） |

**② PROFILE 跟踪（MySQL）**

```sql
SET profiling = 1;
SELECT * FROM users WHERE name = '张三';
SHOW PROFILES;
```

**③ 强制使用指定索引**

```sql
SELECT * FROM users FORCE INDEX(idx_name) WHERE name = '张三';
```

**④ 避免索引失效的典型场景**

- 使用 `LIKE '%abc'`（前导通配符）
- 索引列参与运算或函数：`WHERE YEAR(create_time) = 2026`
- 类型隐式转换：`WHERE phone = 13800138000`（phone 为 varchar）
- OR 条件中包含非索引列
- 组合索引不符合最左前缀原则

> 📎 来源：https://dev.mysql.com/doc/refman/8.0/en/explain-output.html

---

## 题目三：什么是回表查询？如何避免？

### 参考答案

**回表（Back to Table）**是指在 InnoDB 中，使用二级索引（辅助索引）查询时，索引树只存储了索引列和主键值，拿到主键 ID 后还需要再回到主键索引（B+Tree）查完整行数据的过程。

**示例：**

```sql
-- 给 name 建了普通索引
SELECT * FROM users WHERE name = '张三';
```

执行过程：
1. 在 `name` 索引树中搜索，找到 `name='张三'` 对应的主键 ID = 15
2. 回到主键索引树，用 ID=15 查找完整行记录
3. 返回所有列数据

**如何避免回表？——使用覆盖索引**

如果查询的列都包含在索引中，MySQL 可以直接从索引返回结果，无需回表：

```sql
-- 索引：(name, age)，覆盖了查询列
SELECT name, age FROM users WHERE name = '张三';
```

此时 `Extra` 列会显示 `Using index`，称为**索引覆盖扫描（Index Covering Scan）**。

**设计原则：**
- 区分度高的列放前面
- 经常 WHERE / JOIN / ORDER BY / GROUP BY 的列优先建索引
- 考虑将 SELECT 字段尽量纳入组合索引实现覆盖索引

> 📎 来源：https://dev.mysql.com/doc/refman/8.0/en/create-index.html

---

## 题目四：InnoDB 与 MyISAM 存储引擎的核心区别是什么？

### 参考答案

| 特性 | InnoDB | MyISAM |
|------|--------|--------|
| **事务支持** | 支持 ACID 事务（MVCC + Undo Log） | 不支持事务 |
| **锁粒度** | 行级锁（Row Lock），并发高 | 表级锁（Table Lock） |
| **外键** | 支持外键约束 | 不支持外键 |
| **崩溃恢复** | 自动通过 Redo Log 恢复 | 需手动修复（myisamchk） |
| **索引结构** | 聚集索引（Clustered Index），数据与主键索引同存 | 非聚集索引（Non-Clustered），数据与索引分离 |
| **COUNT(*) 优化** | 全表扫描（MVCC 下无计数缓存） | 有专门计数器，O(1) 复杂度 |
| **全文索引** | MySQL 5.6+ 支持（FULLTEXT） | 原生支持全文索引 |
| **适用场景** | 写入密集、事务需求、高并发 | 读密集、无事务需求、空间占用小 |

**核心区别详解：**

1. **事务与锁**：InnoDB 采用行级锁 + MVCC 支持高并发写入；MyISAM 只有表级锁，写入时会锁整个表。

2. **数据存储**：InnoDB 所有数据默认以主键为索引组织存放（聚集索引），主键查询只需一次磁盘 IO；MyISAM 数据和索引分离，需要两次 IO。

3. **崩溃恢复**：InnoDB 通过 Redo Log + Undo Log 实现 ARIES 恢复算法，异常崩溃后可自动恢复；MyISAM 异常退出可能丢失数据或损坏。

4. **并发控制**：InnoDB 的 MVCC（Multi-Version Concurrency Control）通过 Read View 机制实现快照读，避免脏读/不可重复读，适合高并发 OLTP 场景。

> 📎 来源：https://dev.mysql.com/doc/refman/8.0/en/innodb-storage-engine.html

---

## 题目五：如何优化慢查询？具体步骤是什么？

### 参考答案

优化慢查询的系统化步骤：

**Step 1：定位慢查询**

```sql
-- 开启慢查询日志
SET GLOBAL slow_query_log = 'ON';
SET GLOBAL long_query_time = 1;  -- 超过 1 秒记录
SET GLOBAL slow_query_log_file = '/var/log/mysql/slow.log';

-- 查看当前慢查询数量
SHOW STATUS LIKE 'Slow_queries';
```

生产环境推荐使用 `pt-query-digest`（Percona Toolkit）分析慢查询日志。

**Step 2：EXPLAIN 分析执行计划**

重点关注：
- `type`：是否为 `ALL`（全表扫描）→ 需加索引
- `key`：是否用到了索引
- `rows`：扫描行数是否过大
- `Extra`：`Using filesort`、`Using temporary` → 需要优化

**Step 3：检查索引**

```sql
-- 查看表的所有索引
SHOW INDEX FROM orders;

-- 查找缺失索引（MySQL 8.0+）
SELECT * FROM performance_schema.table_io_waits_summary_by_index_usage
WHERE object_name = 'orders' AND count_star = 0;
```

**Step 4：优化 SQL 语句**

- 避免 `SELECT *`，只查需要的列
- 大表分页优化：`WHERE id > last_id LIMIT 20` 替代 `LIMIT 10000, 20`
- 批量操作替代循环单条：`INSERT INTO t VALUES (...), (...), (...)`
- 适当使用 `EXPLAIN FORMAT=JSON` 获取更详细的成本分析

**Step 5：业务层优化**

- 添加 Redis/Memcached 缓存热点数据
- 读写分离：主库写入，从库读取
- 分库分表：按业务 ID 哈希拆分，或按时间范围拆分历史数据
- 使用 ES（ElasticSearch）承担复杂查询和全文检索

**Step 6：数据库配置调优**

```ini
# my.cnf 关键参数
innodb_buffer_pool_size = 70% RAM    # 热点数据缓存
innodb_log_file_size = 1G            # Redo Log 大小
innodb_flush_log_at_trx_commit = 2   # 性能 vs 安全权衡
max_connections = 2000               # 最大连接数
slow_query_log = 1
long_query_time = 1
```

> 📎 来源：https://www.mysql.com/products/enterprise/audit.html + https://www.percona.com/blog/slow-query-log-analysis/

---

## 📌 知识点速查表

| 高频考察点 | 掌握等级 |
|-----------|---------|
| MySQL 查询执行流程 | ⭐⭐⭐ 必须掌握 |
| EXPLAIN 分析执行计划 | ⭐⭐⭐ 必须掌握 |
| 索引失效场景 & 最左前缀原则 | ⭐⭐⭐ 必须掌握 |
| 回表与覆盖索引 | ⭐⭐⭐ 必须掌握 |
| InnoDB vs MyISAM | ⭐⭐⭐ 必须掌握 |
| 慢查询优化步骤 | ⭐⭐ 熟悉 |
| 组合索引设计原则 | ⭐⭐ 熟悉 |
| 分库分表策略 | ⭐ 了解 |
