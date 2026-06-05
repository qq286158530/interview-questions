# 数据库性能优化面试题

> 来源：综合整理自网络经典面试题 + 技术社区高赞讨论
> 日期：2026-06-05

---

## 题目 1：MySQL 中 B+ 树索引的结构是什么？为什么比 B 树更适合数据库？

### 参考答案

**B+ 树的结构特点：**

1. **非叶子节点只存储索引**：不存储实际数据，只有索引键值，这意味着非叶子节点能容纳更多的索引项，树高更矮。
2. **叶子节点包含全部数据**：所有数据行（或指向数据行的指针）都集中在叶子节点，叶子节点之间通过双向链表连接。
3. **查询复杂度稳定**：所有查询都需要走到叶子节点，复杂度固定为 O(log N)，不存在回旋查找。

**为什么比 B 树更适合数据库：**

| 对比点 | B 树 | B+ 树 |
|--------|------|-------|
| 树高 | 较高（数据分散在各层） | 较矮（数据只在叶子层） |
| 范围查询 | 需要中序遍历，效率低 | 叶子链表直接遍历，高效 |
| 查询稳定性 | 不同查询路径不同 | 所有查询路径一致 |
| 磁盘 I/O | 非叶子也存数据，I/O 多 | 非叶子只存索引，I/O 少 |
| 空间利用率 | 较低 | 较高 |

**面试加分点**：InnoDB 的聚簇索引就是基于 B+ 树实现的，数据页大小默认 16KB，通过页目录（Page Directory）实现二分查找，单次磁盘 I/O 能加载更多索引数据。

> 参考来源：
> - https://dev.mysql.com/doc/refman/8.0/en/innodb-storage-engine.html
> - 《高性能MySQL》第3版

---

## 题目 2：如何诊断和优化一条慢查询（SQL 优化的一般流程是什么）？

### 参考答案

**SQL 优化的一般流程：**

**Step 1：开启慢查询日志，捕获慢 SQL**

```sql
-- 查看慢查询配置
SHOW VARIABLES LIKE 'slow_query%';
SHOW VARIABLES LIKE 'long_query_time';

-- 开启慢查询日志
SET GLOBAL slow_query_log = 'ON';
SET GLOBAL long_query_time = 1;  -- 超过1秒记录
SET GLOBAL slow_query_log_file = '/var/log/mysql/slow.log';
```

**Step 2：使用 EXPLAIN 分析执行计划**

```sql
EXPLAIN SELECT ...;
EXPLAIN ANALYZE SELECT ...;  -- MySQL 8.0+ 包含实际成本
```

重点关注字段：
- `type`：最优为 `const/ref/range`，最差为 `ALL`（全表扫描）
- `key`：实际使用的索引
- `rows`：扫描行数，越大越慢
- `Extra`：Using filesort/Using temporary 表示需要额外排序或临时表，效率低

**Step 3：检查索引是否被使用**

```sql
-- 查看表的所有索引
SHOW INDEX FROM orders;

-- 检查索引基数（Cardinality）
SHOW INDEX FROM orders;
-- Cardinality 值过小说明索引区分度低，可能不会被优化器选用
```

**Step 4：分析并改写 SQL**

- 避免 SELECT *，只查询必要字段
- 将子查询改为 JOIN（尤其是 MySQL 5.7 及以前版本）
- 使用覆盖索引避免回表
- 大表分页用延迟关联：

```sql
-- 低效（偏移量越大越慢）
SELECT * FROM orders ORDER BY id LIMIT 100000, 20;

-- 高效（延迟关联）
SELECT o.* FROM orders o
INNER JOIN (SELECT id FROM orders ORDER BY id LIMIT 100000, 20) t
ON o.id = t.id;
```

**Step 5：使用 SQL Tuning Advisor（Oracle）或 Query Rewriter（MySQL）**

```sql
-- MySQL 8.0+ 提示不可用索引
SET optimizer_switch='optimizer_trace=enabled';
```

**Step 6：验证优化效果**

用同样的 SQL 多次执行，对比执行时间。

> 参考来源：
> - https://dev.mysql.com/doc/refman/8.0/en/explain.html
> - https://dev.mysql.com/doc/refman/8.0/en/slow-query-log.html

---

## 题目 3：PostgreSQL 的 MVCC 机制是什么？如何利用它优化读写性能？

### 参考答案

**MVCC（Multi-Version Concurrency Control）多版本并发控制**

PostgreSQL 通过 MVCC 实现并发控制，每个事务看到的是数据库的一致性快照，而不是等待其他事务释放锁。

**核心概念：**

1. **Transaction ID (xmin/xmax)**
   - 每条记录头部存储 `xmin`（插入事务ID）和 `xmax`（删除/更新事务ID）
   - 当前事务ID小于 `xmin` 的记录对该事务可见（已提交）
   - 当前事务ID等于 `xmax` 的记录对该事务不可见（未提交或已删除）

2. **快照（Snapshot）**
   - 快照包含：当前活跃事务ID列表 + 最新已提交事务ID
   - 读已提交（Read Committed）：每个语句重新获取快照
   - 可重复读（Repeatable Read）：整个事务使用同一个快照

3. **表膨胀（bloat）问题**
   - UPDATE 不删除旧版本，而是标记 `xmax`，旧版本堆积导致表膨胀
   - 大量死元组（dead tuples）影响查询性能

**利用 MVCC 优化读写性能的方法：**

**1. 合理控制事务粒度**

```sql
-- 将多个小操作合并为一个事务，减少快照获取次数
BEGIN;
INSERT INTO orders (...) VALUES (...);
UPDATE inventory SET stock = stock - 1 WHERE id = 100;
COMMIT;
```

**2. 使用 VACUUM 清理死元组**

```sql
-- 手动清理（生产环境建议配置自动VACUUM）
VACUUM ANALYZE orders;

-- 查看表膨胀率
SELECT schemaname, tablename,
       round(100 * n_dead_tup::numeric / nullif(n_live_tup + n_dead_tup, 0), 2) AS dead_ratio,
       n_dead_tup
FROM pg_stat_user_tables
ORDER BY n_dead_tup DESC;
```

**3. 使用热表避免索引膨胀**

```sql
-- 对高频更新的字段减少索引
CREATE INDEX CONCURRENTLY idx_orders ON orders(user_id, created_at);

-- 分区表减少 VACUUM 范围
CREATE TABLE orders (...) PARTITION BY RANGE (created_at);
```

**4. 读已提交隔离级别优先**

```sql
SET transaction isolation level read committed;  -- 默认，性能最优
```

**5. 使用 HOT（Heap-Only Tuple）更新**

当 UPDATE 不改变索引列时，PostgreSQL 可以只更新堆表而不更新索引，减少索引膨胀。

> 参考来源：
> - https://www.postgresql.org/docs/current/mvcc.html
> - 《PostgreSQL数据库内核分析》

---

## 题目 4：MySQL InnoDB 的 Buffer Pool 缓存机制是什么？如何调优？

### 参考答案

**Buffer Pool 是 InnoDB 的内存核心组件**，用于缓存数据页和索引页，减少磁盘 I/O。

**核心架构：**

```
Buffer Pool
├── 页面管理
│   ├── Free List（空闲页）
│   ├── LRU List（最近最少使用）
│   │   ├── Young Sublist（热数据，热点页）
│   │   └── Old Sublist（冷数据，新读取的页）
│   └── Flush List（脏页，等待刷新到磁盘）
├── 哈希表（快速定位缓存页）
└── 监控统计区
```

**LRU 淘汰策略（变种）：**

- 传统 LRU 容易导致预读数据污染缓存
- InnoDB 使用冷热分离 LRU：新增页先进入 Old Sublist（占 3/8），在 Old 区停留 `innodb_old_blocks_time` 毫秒后若被访问才进入 Young 区

**关键调优参数：**

```sql
-- 1. 设置合适的 Buffer Pool 大小（建议机器内存的60-80%）
SET GLOBAL innodb_buffer_pool_size = 134217728;  -- 128MB

-- 2. 开启内存页大页支持（提高TLB命中率）
innodb_buffer_pool_instances = 4;  -- 减少竞争

-- 3. 控制冷数据在热区的停留时间（避免一次查询污染缓存）
SET GLOBAL innodb_old_blocks_blocks = 1000;
SET GLOBAL innodb_old_blocks_time = 1000;  -- 毫秒

-- 4. 开启快速关闭（生产禁用）
innodb_flush_log_at_trx_commit = 1;  -- 每次提交刷盘
```

**查看 Buffer Pool 命中率：**

```sql
SHOW ENGINE INNODB STATUS\G

-- 关键指标：
-- Buffer pool hit rate: 命中率应 > 95%
-- Pages made young: 晋升到热区的页数
-- Pages not made young: 未晋升（访问频率不够）
```

**使用 mem_info 监控：**

```sql
SELECT pool_id, hit_rate,
       pages_made_young, pages_not_made_young,
       rows_inserted, rows_updated, rows_deleted
FROM information_schema.INNODB_BUFFER_POOL_STATS;
```

**常见问题排查：**

| 现象 | 原因 | 解决方案 |
|------|------|----------|
| 命中率低 | Buffer Pool 太小 | 扩大 Buffer Pool |
| 大量 new page | 一次性扫描太多冷数据 | 增加 `innodb_old_blocks_time` |
| dirty page 堆积 | 刷新速度跟不上 | 调整 `innodb_max_dirty_pages_pct` |
| 缓存页不足 | 连接数过多 | 减少 `max_connections` |

> 参考来源：
> - https://dev.mysql.com/doc/refman/8.0/en/innodb-buffer-pool.html
> - https://dev.mysql.com/doc/refman/8.0/en/innodb-parameters.html#sysvar_innodb_buffer_pool_size

---

## 题目 5：如何设计分库分表方案？分布式 ID 生成策略有哪些？

### 参考答案

**分库分表设计思路：**

**1. 何时需要分库分表？**

- 单表数据量超过 500 万行或单表磁盘空间超过 20GB
- QPS 超过万级，单库成为瓶颈
- 业务逻辑上数据有明显的隔离边界

**2. 水平拆分策略**

| 拆分维度 | 适用场景 | 缺点 |
|----------|----------|------|
| 用户 ID 取模 | 用户中心、订单表 | 热点用户数据不均 |
| 时间（月份/日期） | 日志表、流水表 | 历史数据查询跨多表 |
| 地区 | 电商配送相关 | 数据分布不均 |
| 哈希取模 | 均衡性要求高 | 扩容困难 |
| 查表法（路由表） | 灵活映射 | 多一次路由查询 |

**3. 扩容方案（避免数据迁移）**

- 方案一：一致性哈希（Circle Hash），节点环状排列，扩缩容只影响邻居节点
- 方案二：扩缩时做双写，新旧表并行后切换

**分布式 ID 生成策略：**

**1. UUID**
```sql
-- 简单但无序，字符串存储，MySQL B+ 树插入性能差
SELECT UUID();  -- e.g., 550e8400-e29b-41d4-a716-446655440000
```

**2. 数据库自增**
```sql
-- 单库单表有序整数，但无法水平扩展
ALTER TABLE orders MODIFY id BIGINT AUTO_INCREMENT;
```

**3. Redis INCR**
```sql
-- 性能高，需要确保 Redis 高可用
INCR order_id_counter  -- 返回整数ID
```

**4. Snowflake 算法（主流方案）**

```
| 1bit | 41bit timestamp | 10bit machine_id | 12bit sequence |
```

- 41bit 时间戳：可支持 69 年
- 10bit 机器ID：支持 1024 个节点
- 12bit 序列号：每节点每毫秒可生成 4096 个 ID

**主流实现：**

| 实现 | 特点 |
|------|------|
| 百度 UidGenerator | 支持动态 WorkerID，热更新 |
| 美团 Leaf | 双号段 + Snowflake，双保险 |
| 滴滴 TinyID | 基于数据库号段，轻量 |
| 雪花算法（原生） | 无中心，性能好 |

**5. 混合方案：Snowflake + Redis 修正时钟漂移**

```java
// 伪代码：时钟回拨时从Redis获取最近使用的最大ID继续递增
if (currentTimestamp < lastTimestamp) {
    // 时钟回拨，从Redis获取最大ID继续
    sequence = redis.incr("max_id") & 0xFFF;
}
```

**分库分表中间件选择：**

| 中间件 | 开发语言 | 生态 | 适用场景 |
|--------|----------|------|----------|
| ShardingSphere-JDBC | Java | 完善 | Java应用，无感知 |
| ShardingSphere-Proxy | Java | 完善 | 多语言，代理模式 |
| MyCat | Java | 老牌 | 兼容旧项目 |
| Vitess | Go | 云原生 | Kubernetes 部署 |
| TiDB | Go | 强 | HTAP 场景 |

> 参考来源：
> - https://shardingsphere.apache.org/
> - https://github.com/Meituan-Dianping/Leaf
> - 《数据库系统概论》

---

**往期题目回顾：**

- 2026-06-04：[数据库性能优化面试题](db-optimization-2026-06-04.md)
- 2026-06-03：[数据库性能优化面试题](db-optimization-2026-06-03.md)

---

> 📌 使用说明：建议先独立思考解答，再对照答案复习关键知识点。