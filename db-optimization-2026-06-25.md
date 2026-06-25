# 数据库性能优化面试题

> 📅 日期：2026-06-25
> 🔧 涵盖：MySQL / PostgreSQL 索引优化、查询优化、存储引擎、缓存策略

---

## 题目一：如何判断SQL是否需要创建索引？如何分析慢查询？

### 参考答案

**判断依据：**
1. `EXPLAIN` 或 `EXPLAIN ANALYZE` 查看执行计划
2. `SHOW GLOBAL STATUS LIKE 'Slow_queries'` 统计慢查询数量
3. 开启 `slow_query_log`，记录超过 `long_query_time` 的SQL

**分析步骤：**
```sql
-- MySQL 查看执行计划
EXPLAIN SELECT * FROM users WHERE name = '张三';

-- PostgreSQL
EXPLAIN (ANALYZE, BUFFERS, FORMAT TEXT) SELECT * FROM users WHERE name = '张三';

-- 查看索引使用情况
SHOW INDEX FROM users;

-- PostgreSQL 查看索引使用统计
SELECT indexrelname, idx_scan, idx_tup_read 
FROM pg_stat_user_indexes 
WHERE idx_scan = 0;
```

**创建索引的原则：**
- WHERE、JOIN、ORDER BY、GROUP BY 涉及的列
- 数据量大（通常 > 10万行）
- 选择性高（Cardinality 高）
- 避免在频繁更新的列上建索引

**来源：** 《高性能MySQL》/ MySQL 8.0 官方文档

---

## 题目二：什么是索引下推（Index Condition Pushdown，ICP）？它如何提升查询性能？

### 参考答案

**ICP 原理：**
索引下推是 MySQL 5.6+ 引入的优化技术。传统方式下，索引查询分两步：
1. 存储引擎通过索引定位所有满足第一条件的行
2. 服务层回表获取完整行数据，再在 Server 层过滤剩余条件

**ICP 优化后：**
- 将过滤条件下推到存储引擎层，在索引遍历过程中就完成过滤
- 减少回表次数，降低 I/O 开销

**示例：**
```sql
SELECT * FROM users WHERE age > 20 AND name LIKE '张%';
-- 索引: (age, name)
```

- **传统方式：** 先用 age > 20 找到所有行，回表后再用 name LIKE '张%' 过滤
- **ICP：** 在索引遍历时直接用 name LIKE '张%' 过滤，只回表满足两条条件的行

**开启方式：** 默认开启，可通过 `SET optimizer_switch = 'index_condition_pushdown=off'` 关闭

**PostgreSQL 对应优化：** 使用**索引条件推导**，MySQL ICP 更激进地将过滤推入存储引擎层执行。

**来源：** MySQL 8.0 Reference Manual - Optimizing IN and EXISTS Subqueries with Semi-Join Transformations

---

## 题目三：MySQL InnoDB 的 MVCC 机制是什么？如何实现隔离级别？

### 参考答案

**MVCC（Multi-Version Concurrency Control）多版本并发控制：**

**核心概念：**
- 每行数据有两个隐藏列：`DB_TRX_ID`（最近修改的事务ID）和 `DB_ROLL_PTR`（指向undo log的指针）
- 每次事务修改数据，都会将旧数据写入 undo log，形成版本链
- 读取时根据事务的 `read_view` 判断哪个版本可见

**Read View（读视图）机制：**
```
read_view 结构：
- m_ids：活跃事务ID列表
- min_trx_id：最小活跃事务ID
- max_trx_id：创建read_view时最大事务ID+1
- creator_trx_id：当前事务ID
```

**各隔离级别表现：**

| 隔离级别 | 读数据方式 | 可重复读（RR） | 读已提交（RC） |
|---------|-----------|--------------|--------------|
| **Read Uncommitted** | 最新版本（未提交也可见） | ❌ | ❌ |
| **Read Committed** | 最新已提交版本（每次SELECT新生成read_view） | ⚠️ | ✅ |
| **Repeatable Read** | 事务开始时read_view，版本链遍历 | ✅ | ❌ |
| **Serializable** | 锁，读操作也加锁 | ✅ | ✅ |

**RR 级别下的幻读问题：**
- 快照读（普通SELECT）通过MVCC解决
- 当前读（SELECT FOR UPDATE / INSERT）通过 Next-Key Lock 解决

**PostgreSQL MVCC vs MySQL InnoDB MVCC：**

| 特性 | PostgreSQL | MySQL InnoDB |
|-----|-----------|-------------|
| 实现方式 | Vacuum 清理旧版本 | undo log 链 |
| 垃圾回收 | 定时 vacuum 进程 | 后台 purge 线程 |
| 可见性判断 | xmin/xmax + 快照 | read_view + trx_id |

**来源：** 《MySQL技术内幕：InnoDB存储引擎》/ PostgreSQL Documentation

---

## 题目四：如何优化分页查询（ LIMIT offset, n ）在大数据量下的性能问题？

### 参考答案

**问题本质：**
```sql
SELECT * FROM orders ORDER BY id LIMIT 1000000, 20;
```
MySQL 需要扫描前 1000020 行，然后丢弃前 1000000 行，效率极低。

**优化方案：**

**方案1：延迟关联（Deferred Join）**
```sql
-- 先通过索引定位ID，再关联获取完整数据
SELECT t.* FROM orders t 
INNER JOIN (
    SELECT id FROM orders ORDER BY id LIMIT 1000000, 20
) AS temp ON t.id = temp.id;
```
子查询只扫描索引列（覆盖索引），大幅减少扫描行数。

**方案2：游标分页（Keyset Pagination）**
```sql
-- 记录上一页最后一条的ID
SELECT * FROM orders 
WHERE id > 1000000 
ORDER BY id 
LIMIT 20;
```
时间复杂度 O(1)，但不支持跳页。

**方案3：区间查询**
```sql
-- 记录每个区间边界值
SELECT * FROM orders 
WHERE id BETWEEN 1000000 AND 1000020;
```

**方案4：使用索引覆盖 + 排序**
```sql
-- 确保 ORDER BY 列在索引中，且 SELECT 列也在索引中
ALTER TABLE orders ADD INDEX idx_id (id);
SELECT id FROM orders ORDER BY id LIMIT 1000000, 20;
```

**PostgreSQL 优化：**
```sql
-- 使用 OFFSET FETCH（SQL Server/PostgreSQL 标准语法）
SELECT * FROM orders ORDER BY id LIMIT 20 OFFSET 1000000;

-- PostgreSQL 游标风格
SELECT * FROM orders WHERE id > last_id ORDER BY id LIMIT 20;
```

**来源：** MySQL Performance Blog / High Performance MySQL, 3rd Edition

---

## 题目五：MySQL 主从复制原理是什么？如何处理复制延迟？

### 参考答案

**主从复制原理：**

**MySQL Binlog 复制流程：**
```
Master                          Slave
  │                               │
  │  1. 事务提交，写入 Binlog      │
  │───────────────────────────────►
  │                               │ 2. I/O Thread 拉取 Binlog
  │                               │ 3. 写入 Relay Log
  │                               │ 4. SQL Thread 重放 Relay Log
  │                               │
```

**三种 Binlog 格式：**
| 格式 | 说明 | 优点 | 缺点 |
|-----|------|-----|------|
| **Row** | 记录行级变更 | 精确，不丢数据 | Binlog 大 |
| **Statement** | 记录SQL语句 | Binlog 小 | 函数/触发器可能不一致 |
| **Mixed** | 混合模式 | 平衡 | 复杂 |

**复制类型：**
- **异步复制**：默认，主库提交后不等从库确认
- **半同步复制（Semi-sync）**：至少一个从库确认后提交
- **GTID 复制**：基于事务ID定位日志位置，避免位点错误

**复制延迟原因与解决方案：**

| 原因 | 解决方案 |
|-----|---------|
| 从库服务器性能弱 | 升级硬件，SSD |
| 主从库配置差异 | 统一配置（尤其是 InnoDB 参数） |
| 大事务拆分 | 分批执行，避免大事务 |
| 缺少索引导致从库执行慢 | 确保从库也有对应索引 |
| 并行复制未开启 | 开启 `slave_parallel_workers` |

**PostgreSQL 主从复制：**

```sql
-- Streaming Replication 配置
wal_level = replica
max_wal_senders = 3
wal_keep_size = 64

-- 物理复制（流复制）
-- 逻辑复制（支持订阅发布）
```

**复制延迟监控：**
```sql
-- MySQL
SHOW SLAVE STATUS\G
-- 关注：Seconds_Behind_Master、Slave_IO_Running、Slave_SQL_Running

-- PostgreSQL
SELECT * FROM pg_stat_replication;
```

**来源：** MySQL 8.0 High Availability / PostgreSQL Documentation - Streaming Replication

---

> 🚀 更多面试题收录于 [interview-questions](https://github.com/qq286158530/interview-questions) 仓库
> 欢迎 Star ⭐ 一起刷题！
