# 数据库性能优化面试题

> 📅 整理日期：2026-07-14
> 🏷️ 技术栈：MySQL / PostgreSQL
> 💡 来源：综合整理自网络公开技术文章与面试经验

---

## 题目一：MySQL 索引失效的场景有哪些？如何避免？

### 答案

**常见的索引失效场景：**

1. **使用 `LIKE` 以通配符开头**
   ```sql
   -- 失效
   SELECT * FROM users WHERE name LIKE '%张%';
   -- 生效
   SELECT * FROM users WHERE name LIKE '张%';
   ```

2. **WHERE 子句中对索引列进行计算或使用函数**
   ```sql
   -- 失效
   SELECT * FROM orders WHERE YEAR(create_time) = 2026;
   -- 生效：改为范围查询
   SELECT * FROM orders WHERE create_time >= '2026-01-01' AND create_time < '2027-01-01';
   ```

3. **使用 `OR` 连接条件，且OR两端不全是索引列**
   ```sql
   -- 失效（phone有索引，name无索引）
   SELECT * FROM users WHERE phone = '13800138000' OR name = '张三';
   -- 生效：拆分为两条SQL + UNION
   ```

4. **数据类型隐式转换**
   ```sql
   -- phone 是 VARCHAR 类型，传入数字会隐式转换导致索引失效
   SELECT * FROM users WHERE phone = 13800138000; -- 失效！
   ```

5. **`!=` 或 `<>` 导致索引失效**
   大多数情况下 MySQL 无法利用 B+Tree 范围查询的特性。

6. **字段值为 NULL 导致索引失效**
   MySQL 中 NULL 值不等于任何值，包括自身，`IS NULL/IS NOT NULL` 可能不走索引。

7. **复合索引未遵循最左前缀原则**
   ```sql
   -- 索引为 (a, b, c)
   -- 失效
   SELECT * FROM t WHERE b = 1;        -- 跳过了a
   SELECT * FROM t WHERE c = 1;        -- 跳过了a、b
   -- 生效
   SELECT * FROM t WHERE a = 1;        -- 用到a
   SELECT * FROM t WHERE a = 1 AND b = 1; -- 用到a、b
   ```

**避免索引失效的建议：**
- 分析 `EXPLAIN` 执行计划，关注 `type`、`key`、`Extra` 列
- 避免在索引列上使用函数、运算、类型转换
- 尽量使用复合索引时遵循最左前缀原则
- 对于大表批量导入前删除索引，导入后重建

---

## 题目二：如何排查和优化一条慢 SQL？

### 答案

**排查步骤：**

1. **开启慢查询日志**
   ```sql
   -- 查看是否开启
   SHOW VARIABLES LIKE 'slow_query_log';
   -- 开启（临时）
   SET GLOBAL slow_query_log = 'ON';
   SET GLOBAL long_query_time = 1; -- 超过1秒记录
   ```

2. **分析执行计划 `EXPLAIN`**
   ```sql
   EXPLAIN SELECT ...;
   ```
   关键字段：
   - `type`: 最好达到 `ref/range`，避免 `ALL`（全表扫描）
   - `key`: 实际使用的索引
   - `rows`: 扫描行数，越少越好
   - `Extra`: 避免出现 `Using filesort`、`Using temporary`

3. **检查索引是否合理**
   - 是否有对应字段的索引
   - 索引是否被正确使用（最左前缀、数据类型等）

4. **常见优化手段：**

   | 问题 | 优化方案 |
   |------|---------|
   | 全表扫描 | 添引覆盖索引或联合索引 |
   | Using filesort | 优化 ORDER BY，引入索引覆盖 |
   | Using temporary | 优化 GROUP BY，拆分子查询 |
   | 深分页 | 改用延迟关联或游标分页 |
   | 大量JOIN | 拆分为简单查询，避免过多关联 |

5. **分页优化示例**
   ```sql
   -- 深分页：改前（慢）
   SELECT * FROM orders LIMIT 1000000, 10;
   -- 改后（快）：延迟关联
   SELECT o.* FROM orders o
   INNER JOIN (SELECT id FROM orders LIMIT 1000000, 10) t
   ON o.id = t.id;
   ```

---

## 题目三：MySQL InnoDB 与 MyISAM 存储引擎的区别？如何选择？

### 答案

| 特性 | InnoDB | MyISAM |
|------|--------|--------|
| **事务支持** | 支持 ACID 事务 | 不支持事务 |
| **锁粒度** | 行级锁（高并发） | 表级锁 |
| **外键** | 支持 | 不支持 |
| **崩溃恢复** | 自动恢复（redo log） | 较差（需手动修复） |
| **全文索引** | 5.6+ 支持 | 原生支持 |
| **存储结构** | 聚簇索引（主键和数据在一起） | 非聚簇索引（索引与数据分离） |
| **COUNT(*)** | 全表扫描（需手动计数缓存） | 保存行数，快速返回 |
| **适用场景** | 核心业务、需要事务、高并发 | 读多写少、不需要事务、日志表 |

**选择建议：**
- **选 InnoDB**：几乎所有业务场景，特别是写入频繁、需要事务、数据可靠性要求高的系统
- **选 MyISAM**：只读场景、历史数据归档、日志系统（已被越来越多场景用 铁木存储替代）

**补充：InnoDB 行数查询优化**
```sql
-- 近似值（快）
SELECT TABLE_ROWS FROM information_schema.TABLES WHERE TABLE_NAME = 'orders';
-- 精确值（慢）
SELECT COUNT(*) FROM orders;
```

---

## 题目四：PostgreSQL 如何进行性能调优？有哪些关键参数和工具？

### 答案

**关键配置参数（postgresql.conf）：**

```ini
# 内存相关
shared_buffers = 1/4 RAM        # 共享缓冲区，建议为系统内存的1/4
work_mem = 64MB                  # 排序/哈希操作内存
effective_cache_size = 1/2 RAM  # 规划器估算的可用缓存

# 写日志
wal_buffers = 16MB               # WAL 缓冲区
checkpoint_completion_target = 0.9  # 检查点平滑度

# 并行查询
max_worker_processes = 8
max_parallel_workers_per_gather = 4

# 连接数
max_connections = 200
```

**常用诊断工具：**

1. **`EXPLAIN ANALYZE`** — 执行计划分析
   ```sql
   EXPLAIN (ANALYZE, BUFFERS, FORMAT TEXT) SELECT ...;
   ```
   重点关注：Seq Scan（顺序扫描）过多、Hash Join / Nested Loop 效率。

2. **`pg_stat_statements`** — 慢查询统计
   ```sql
   -- 开启扩展
   CREATE EXTENSION pg_stat_statements;
   -- 查询最慢的SQL
   SELECT query, calls, mean_time, total_time
   FROM pg_stat_statements
   ORDER BY mean_time DESC LIMIT 10;
   ```

3. **`pg_stats`** — 索引统计信息
   ```sql
   SELECT attname, n_distinct, correlation
   FROM pg_stats
   WHERE tablename = 'orders';
   ```

4. **`vacuumdb`** — 垃圾回收与 ANALYZE
   ```bash
   vacuumdb -z -h localhost -U postgres mydb  # 顺便更新统计信息
   ```

**日常优化建议：**
- 定期 `VACUUM ANALYZE` 保持统计信息新鲜
- 合理使用索引（`CREATE INDEX CONCURRENTLY` 避免锁表）
- 使用连接池（PgBouncer）减少连接开销
- 开启 `auto_explain` 记录慢查询执行计划

---

## 题目五：如何设计分库分表方案？说说你的实践思路？

### 答案

**分库分表的动机：**
- 单表数据量超过千万级，查询性能急剧下降
- 单库并发达到瓶颈（连接数、QPS）
- 数据冷热分离，减少热数据竞争

**常见的分片策略：**

1. **按主键 ID 哈希分片**
   ```sql
   -- 优点：数据分布均匀
   -- 缺点：按时间/用户查询需跨分片
   shard_id = hash(id) % N
   ```

2. **按时间/月份分片**
   ```sql
   -- 适合日志、订单等有明显时间特征的表
   -- 优点：历史数据归档方便
   -- 缺点：热点分片容易写倾斜
   shard_id = MONTH(create_time) % N
   ```

3. **按业务维度（用户ID、店铺ID）分片**
   ```sql
   -- 适合按用户维度查询的业务
   shard_id = user_id % N
   ```

**分库分表带来的挑战：**

| 问题 | 解决方案 |
|------|---------|
| 跨分片查询 | ES/ClickHouse 搜索引擎，多级汇聚 |
| 跨分片聚合（COUNT/SUM） | 应用层合并结果，或预计算 |
| 分片键无法更改 | 引入逻辑分片ID + 映射表（不推荐，复杂） |
| 主键唯一性 | 分布式ID（Snowflake / UUID / 美团Leaf） |
| 数据迁移 | 使用 Binlog 同步，双写期间对比校验 |

**工具选型：**
- **ShardingSphere**（Java）：ShardingJDBC / Proxy 模式
- **MyCat**（老牌）：基于 Proxy，支持跨库JOIN
- **Vitess**（YouTube 开源）：MySQL 分片 + 水平扩展方案
- **Citus**（PostgreSQL）：分布式 PostgreSQL，支持分片

**最佳实践：**
> 分库分表是最后手段，优先考虑：读写分离 → 表分区 → 缓存 → 再考虑分片。分片会极大增加系统复杂度，引入分布式事务问题和运维成本。

---

> 📌 **更多面试题**：[qq286158530/interview-questions](https://github.com/qq286158530/interview-questions)
> 
> 如有错误或补充，欢迎提交 Issue 或 PR！
