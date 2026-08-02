# 数据库性能优化面试题（2026-08-02）

> 来源说明：以下题目综合整理自互联网面试题库与技术博客，链接指向常见的技术社区和面试平台。

---

## 题目 1：MySQL 中索引失效的场景有哪些？如何避免？

### 参考答案

索引失效的常见场景：

1. **使用函数或运算**：对索引列做 `SELECT * FROM t WHERE YEAR(date) = 2026` 会导致索引失效，应改为范围查询 `date BETWEEN '2026-01-01' AND '2026-12-31'`。
2. **类型转换**：where 条件中字段类型是字符串，但传入的是整数，如 `phone = 13800138000`（phone 为 varchar），隐式类型转换导致全表扫描。
3. **LIKE 以通配符开头**：`LIKE '%abc'` 无法使用索引，应改用 `LIKE 'abc%'` 或使用全文索引。
4. **OR 连接不同类型列**：`WHERE name = 'Tom' OR age = 20`，若 name 和 age 各自有索引但类型不同，MySQL 优化器可能放弃索引。
5. **违背最左前缀原则**：复合索引 `(a, b, c)` 不使用 a 或不连续使用，查询 `WHERE b = 1` 无法使用索引。
6. **使用 NOT、!=、<>**：导致索引失效。
7. **WHERE 子句中对索引列进行表达式计算**：`WHERE id + 1 = 100`。

**避免策略**：尽量在 WHERE 条件中保持列的独立性和类型匹配；设计复合索引时考虑查询的过滤条件顺序；使用覆盖索引减少回表。

> 📎 来源：<https://blog.csdn.net/ThinkWon/article/details/104778621>

---

## 题目 2：如何优化慢查询？请描述定位和分析慢查询的完整流程。

### 参考答案

**定位慢查询的流程：**

1. **开启慢查询日志**：
   ```sql
   -- 临时开启
   SET GLOBAL slow_query_log = 'ON';
   SET GLOBAL long_query_time = 1; -- 超过1秒记录
   SET GLOBAL slow_query_log_file = '/var/lib/mysql/slow.log';
   ```
2. **使用 EXPLAIN 分析执行计划**：
   ```sql
   EXPLAIN SELECT * FROM orders WHERE status = 'paid';
   ```
   重点关注：`type`（ref/all）、`key`（使用的索引）、`rows`（扫描行数）、`Extra`（Using filesort/Using temporary）。

3. **使用 SHOW PROFILE**：
   ```sql
   SET profiling = 1;
   -- 执行查询
   SHOW PROFILES;
   SHOW PROFILE FOR QUERY 1;
   ```

**优化手段：**

- 添加合适索引，覆盖索引减少回表
- 避免 SELECT *，只查需要的字段
- 分解大查询，延迟关联
- 优化子查询，改用 JOIN
- 拆分批量操作，减少锁竞争
- 利用慢查询日志持续监控

> 📎 来源：<https://www.cnblogs.com/shenjianzhang/p/14704223.html>

---

## 题目 3：MySQL InnoDB 与 MyISAM 存储引擎的区别是什么？如何选择？

### 参考答案

| 特性 | InnoDB | MyISAM |
|------|--------|--------|
| 事务支持 | 支持 ACID 事务 | 不支持 |
| 锁粒度 | 行级锁，适合高并发 | 表级锁 |
| 外键 | 支持 | 不支持 |
| 全文索引 | 5.6+ 支持 | 支持 |
| 崩溃恢复 | 自动恢复（redo log） | 较差（.MYD/.MYI） |
| 存储空间 | 较大（双写缓冲等） | 较小 |
| 适用场景 | 写密集、事务需求 | 读密集、不需要事务 |

**选择建议**：
- 绝大多数场景选 **InnoDB**（MySQL 5.7+ 默认引擎）。
- 只有在极端读多写少（如日志表、统计表）且不需要事务时，才考虑 MyISAM。
- 动态表、频繁更新的表必须用 InnoDB。

> 📎 来源：<https://juejin.cn/post/6844903873508638728>

---

## 题目 4：什么是数据库缓存策略？Redis 与 MySQL 如何配合实现多级缓存？

### 参考答案

**缓存策略核心概念**：

- **Cache-Aside（旁路缓存）**：应用先查缓存，未命中再查 DB 并写入缓存；更新时先删缓存再更新 DB（双删策略防脏读）。
- **Read-Through / Write-Through**：缓存层自行加载/写入数据，应用不感知。
- **Write-Behind**：异步写入，性能最高但存在数据丢失风险。

**Redis + MySQL 配合方案**：

```
读：GET(key) → 命中返回
    ↓ 未命中
    SELECT DB → 写入 Redis → 返回

写：DELETE(key) → UPDATE DB（延迟双删：先删，再更新，再删）
```

**常见优化点**：
- 缓存键设计：`user:profile:{user_id}`
- TTL 设置：热点数据短期（如 5 分钟），冷数据长期（如 1 天）
- 缓存穿透：布隆过滤器或空值缓存
- 缓存击穿：互斥锁或热点数据永不过期
- 缓存雪崩：随机 TTL + 多级缓存架构

> 📎 来源：<https://blog.csdn.net/qq_35124703/article/details/123915816>

---

## 题目 5：PostgreSQL 中如何进行性能调优？请从参数配置、查询优化、索引策略三个方面说明。

### 参考答案

**一、参数配置（postgresql.conf）**

```conf
# 共享缓冲区（建议设为系统内存的 25%）
shared_buffers = 8GB

# 工作内存（单个查询操作可用内存）
work_mem = 256MB

# 维护操作内存
maintenance_work_mem = 1GB

# 开启并行查询
max_worker_processes = 8
max_parallel_workers_per_gather = 4

# 开启 WAL 日志异步提交
wal_sync_method = open_datasync
```

**二、查询优化**
- 使用 `EXPLAIN (ANALYZE, BUFFERS)` 分析执行计划
- 避免 SELECT *，使用覆盖索引
- 用 `VACUUM ANALYZE` 定期清理 dead tuples 并更新统计信息
- 合理使用分区表（PARTITION BY RANGE/LIST）

**三、索引策略**
- B-tree 索引：适合等值查询和范围查询（默认索引类型）
- GIN 索引：适合数组、全文检索、JSONB 数据
- Partial Index：只索引满足条件的行，如 `CREATE INDEX idx_active ON orders(total) WHERE status = 'active'`
- 复合索引需遵循最左前缀原则
- 避免过多索引增加写入开销，定期使用 `REINDEX` 重建碎片化索引

> 📎 来源：<https://www.postgresql.org/docs/13/performance-tips.html>

---

*整理 by 数据库面试题助手 · 2026-08-02*
