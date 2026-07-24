# 数据库性能优化面试题 — 2026-07-24

> 本次收录 5 道高质量数据库性能优化面试题，涵盖 MySQL 与 PostgreSQL 核心知识点。

---

## 题目 1：什么是覆盖索引？如何利用覆盖索引避免回表查询？

**参考答案：**

### 回表查询的概念

InnoDB 采用 B+Tree 索引结构，数据行存储在主键索引（聚簇索引）的叶子节点中。普通二级索引叶子节点只存储索引列值和主键 ID。

当查询的字段不在二级索引中时，需要先通过二级索引找到主键，再去主键索引回表查询完整行数据，这个过程称为**回表**。

```sql
-- 假设有表：users(id PRIMARY KEY, name, age, email)
-- 以及索引：INDEX idx_age (age)

-- 需要回表（email 不在 idx_age 中）
SELECT id, name, email FROM users WHERE age = 25;

-- 不需要回表（id 和 age 都在索引中，属于覆盖索引）
SELECT id, age FROM users WHERE age = 25;
```

### 覆盖索引的定义

如果一个索引包含了 SELECT、WHERE、ORDER BY 中所有需要的字段，那么查询只需要扫描索引，无需回表，这种索引称为**覆盖索引（Covering Index）**。

### 覆盖索引的优势

1. **减少 IO 操作**：索引节点远小于完整数据行，减少磁盘读写
2. **减少内存访问**：索引结构更小，可缓存更多在内存中
3. **提升查询速度**：索引树高更小，遍历成本更低

### 实战技巧

```sql
-- 场景：查询用户最近一笔订单的订单号和金额
-- 反例：每次都回表
SELECT id, order_no, amount
FROM orders
WHERE user_id = 10086
ORDER BY created_at DESC
LIMIT 1;

-- 正例：利用覆盖索引
-- 建立索引：INDEX idx_user_time (user_id, created_at, order_no, amount)
CREATE INDEX idx_covering ON orders(user_id, created_at, order_no, amount);

-- 覆盖索引直接返回所有字段，无需回表
SELECT id, order_no, amount
FROM orders
WHERE user_id = 10086
ORDER BY created_at DESC
LIMIT 1;
```

### EXPLAIN 中的判断方法

执行计划 `Extra` 列显示 `Using index` 表示使用了覆盖索引：

```
+----+-------------+--------+------------+-------+---------------+---------+---------+------+------+----------+-------------+
| id | select_type| table  | type       | key   | key_len       | ref     | rows    | rts  | Extra                        |
+----+-------------+--------+------------+-------+---------------+---------+---------+------+------+----------+-------------+
|  1 | SIMPLE     | orders | ref        | idx_age | 4           | const   |   1000  | rts  | Using index                  |
+----+-------------+--------+------------+-------+---------------+---------+---------+------+------+----------+-------------+
```

### 注意事项

- 覆盖索引会增加索引维护成本（写操作更慢），需权衡查询频率与更新成本
- 不要 `SELECT *`，尽量只查需要的字段
- 联合索引的字段顺序很重要，覆盖字段要放在最后

> 📎 来源：[MySQL 官方文档 — Optimizing Queries with Indexes](https://dev.mysql.com/doc/refman/8.0/en/indexes.html)

---

## 题目 2：MySQL InnoDB 的 Buffer Pool 是什么？如何调优？

**参考答案：**

### Buffer Pool 的作用

Buffer Pool 是 InnoDB 在内存中开辟的一块缓存区域，用于缓存表数据和索引，减少磁盘 IO，提升查询性能。是 InnoDB 最核心的内存组件。

默认大小：`innodb_buffer_pool_size = 128M`（生产环境通常设为机器内存的 60%~80%）

### Buffer Pool 的数据结构

```
Buffer Pool
├── Page Hash（哈希表）：通过 (space_id, page_no) 快速定位缓存页
├── Free List（空闲页链表）：管理空闲缓存页
├── Flush List（脏页链表）：记录被修改但未刷盘的缓存页
└── LRU List（最近最少使用链表）：淘汰最久未被访问的缓存页
```

### 缓存页淘汰：LRU 算法

InnoDB 采用**改进版 LRU**（Least Recently Used），分为两部分：

```
LRU List
├── New Subpool (37%)：热数据区，频繁访问的缓存页
└── Old Subpool (63%)：冷数据区，新读入或偶尔访问的缓存页
```

**为什么改进？** 避免大表扫描污染 Buffer Pool：
- 全表扫描时，数据页只进入 Old 区，后续被淘汰不影响热数据
- 热点数据始终在 New 区，不会被挤出

### 关键配置参数

```sql
-- 设置 Buffer Pool 大小（建议机器内存的 60~80%）
SET GLOBAL innodb_buffer_pool_size = 8589934592; -- 8GB

-- 官方推荐：chunk size 为 1GB 的倍数
innodb_buffer_pool_chunk_size = 134217728; -- 128MB

-- Buffer Pool 实例数（可减少锁竞争）
innodb_buffer_pool_instances = 4; -- 建议设置为 CPU 核心数

-- 脏页刷新策略（影响写入性能）
innodb_flush_neighbors = 1; -- 刷脏时顺带刷新邻居页，减少磁盘离散 IO
```

### 监控与诊断

```sql
-- 查看 Buffer Pool 状态
SHOW ENGINE INNODB STATUS\G

-- Information Schema 查询
SELECT pool_id, pool_size, free_buffers, database_pages
FROM information_schema.INNODB_BUFFER_POOL_STATS;

-- 查看缓存命中率
SHOW GLOBAL STATUS LIKE 'Innodb_buffer_pool_read%';
-- 计算：1 - (read_requests / reads) 越高越好
```

### 常见调优思路

| 问题 | 解决方案 |
|------|----------|
| Buffer Pool 太小 | 增大 `innodb_buffer_pool_size`，确保数据能缓存 |
| 缓存命中率低 | 检查是否频繁全表扫描，分析慢查询 |
| 脏页堆积导致刷新抖动 | 调小 `innodb_max_dirty_pages_pct`，或开启 `innodb_flush_neighbors` |
| 锁竞争严重 | 增加 `innodb_buffer_pool_instances` |
| 预热时间长 | 使用 `innodb_buffer_pool_dump_at_shutdown` 持久化缓存，下次启动快速恢复 |

> 📎 来源：[MySQL 官方文档 — InnoDB Buffer Pool](https://dev.mysql.com/doc/refman/8.0/en/innodb-buffer-pool.html)

---

## 题目 3：什么是最左前缀原则？复合索引失效的常见场景有哪些？

**参考答案：**

### 最左前缀原则的定义

复合索引（多列索引）遵循**最左前缀原则**：查询条件必须从索引最左列开始，并且连续不间断，才能使用索引。

```
建立索引：INDEX idx_name (col_a, col_b, col_c)

-- 可以使用索引（从最左开始，连续）
WHERE col_a = 'xxx'
WHERE col_a = 'xxx' AND col_b = 'yyy'
WHERE col_a = 'xxx' AND col_b = 'yyy' AND col_c = 'zzz'

-- 部分使用（从最左开始，但不连续）
WHERE col_a = 'xxx' AND col_c = 'zzz'  -- 只用到 col_a

-- 无法使用索引（没有从最左开始）
WHERE col_b = 'yyy'
WHERE col_c = 'zzz'
```

### 底层原理

B+Tree 索引按照索引列从左到右构建树结构：

```
(col_a, col_b) 复合索引结构示意：
         col_a=1
        /      \
   col_a=1      col_a=2
   /     \       /     \
col_b=1 col_b=2 col_b=3 col_b=4

索引排序：(1,1) < (1,2) < (2,3) < (2,4)
```

从左到右逐列排序，先按 col_a 排序，col_a 相同时按 col_b 排序。如果跳过最左列，后续列就无法有序查找。

### 复合索引失效的常见场景

| 场景 | SQL 示例 | 索引使用情况 |
|------|----------|-------------|
| **跳过最左列** | `WHERE col_b = 'xxx'` | ❌ 完全失效 |
| **中间列缺失** | `WHERE col_a = 'xxx' AND col_c = 'zzz'` | ⚠️ 只用到 col_a |
| **最左列使用范围** | `WHERE col_a > 1 AND col_b = 'xxx'` | ⚠️ col_a 范围后断链，col_b 失效 |
| **LIKE 左侧加通配** | `WHERE col_a LIKE '%xxx'` | ❌ 索引失效 |
| **OR 条件** | `WHERE col_a = 'x' OR col_b = 'y'` | ⚠️ 通常全表扫描（可用 union 改写） |
| **列进行运算/函数** | `WHERE YEAR(col_a) = 2026` | ❌ 索引失效 |

### 最佳实践

**1. 合理安排列顺序**

将**区分度高、查询频繁的列放在前面**：

```sql
-- 用户表经常查询：WHERE status = 1 AND city = '北京'
-- status 区分度更高（只有2个值），city 区分度更高（城市多）
-- 索引：INDEX idx_status_city (status, city)  -- 正确
-- 索引：INDEX idx_city_status (city, status)  -- 次优
```

**2. 利用索引排序**

```sql
-- 索引：(name, age)
-- 可以利用索引排序
ORDER BY name;              -- ✅ 整个索引可排序
ORDER BY name, age;         -- ✅ 整个索引可排序
ORDER BY name DESC;         -- ✅ 可以

-- 无法利用索引排序
ORDER BY age;               -- ❌ 跳过 name
ORDER BY age, name;         -- ❌ 跳过 name
```

**3. 使用 EXPLAIN 验证**

```sql
EXPLAIN SELECT * FROM users
WHERE name = '张三' AND age = 25
ORDER BY created_at DESC;
-- 查看 key 列是否显示使用了索引
-- 查看 Extra 是否出现 Using filesort（需要额外排序）
```

> 📎 来源：[MySQL 官方文档 — Multiple-Column Indexes](https://dev.mysql.com/doc/refman/8.0/en/multiple-column-indexes.html)

---

## 题目 4：PostgreSQL 的 VACUUM 机制是什么？为什么需要定期 VACUUM？

**参考答案：**

### PostgreSQL MVCC 与 VACUUM 的背景

PostgreSQL 使用 MVCC（多版本并发控制）实现事务隔离。UPDATE 和 DELETE 时，旧版本行不会立即删除，而是标记为"死亡元组"，由 VACUUM 后续清理。

```sql
-- UPDATE 时，PostgreSQL 实际执行：
-- 1. 插入新行（xmax = 当前事务ID）
-- 2. 标记旧行为死亡元组（并未物理删除）
```

### 为什么要 VACUUM？

**1. 回收死亡元组占用的磁盘空间**

死亡元组如果不清理，会持续占用磁盘和内存，影响查询性能（数据膨胀）。

**2. 更新事务IDfreeze，避免 wraparound**

PostgreSQL 使用 32 位事务ID，约 40 亿个，用完会 wraparound 环绕，导致事务ID混乱。VACUUM 将旧事务ID freeze，释放空间。

**3. 更新 Free Space Map**

让新增数据能复用死亡元组的物理位置，避免表文件持续膨胀。

**4. 更新可见性映射（Visibility Map）**

帮助 PostgreSQL 快速判断哪些页只包含可见元组，扫描时跳过，提高 index-only scan 效率。

### VACUUM 的类型

| 类型 | 说明 |
|------|------|
| **VACUUM（普通）** | 只清理死亡元组，不回收空间，可并发执行 |
| **VACUUM FULL** | 回收所有可用空间，会锁表，生产环境慎用 |
| **Autovacuum** | 后台自动执行，由参数控制触发阈值 |
| **VACUUM ANALYZE** | 清理 + 更新统计信息（表行数、分布），帮助优化器生成更优执行计划 |

### 关键参数配置

```sql
-- autovacuum 默认开启，以下是重要参数
ALTER TABLE mytable SET (
    autovacuum_vacuum_threshold = 50,      -- 死亡元组超过50行触发
    autovacuum_analyze_threshold = 50,     -- 变更行数超过50触发 analyze
    autovacuum_vacuum_scale_factor = 0.1,   -- 表超过10%时触发
    autovacuum_vacuum_cost_delay = 20ms     -- VACUUM 成本延迟（控制IO影响）
);
```

### 监控与诊断

```sql
-- 查看表膨胀情况
SELECT relname, n_dead_tup, n_live_tup, last_vacuum, last_autovacuum
FROM pg_stat_user_tables
WHERE n_dead_tup > 1000
ORDER BY n_dead_tup DESC;

-- 查看表和索引大小
SELECT pg_size_pretty(pg_total_relation_size('mytable'));

-- 手动 VACUUM
VACUUM VERBOSE ANALYZE mytable;
```

### 常见问题与处理

| 问题 | 原因 | 解决方案 |
|------|------|----------|
| 表持续膨胀 | 更新频繁但 VACUUM 未及时执行 | 调低 `autovacuum_vacuum_scale_factor` |
| VACUUM 占用大量IO | 清理量太大 | 调高 `autovacuum_vacuum_cost_delay` |
| 事务ID接近 wraparound | 长期未 VACUUM 的表 | `VACUUM FULL` 或紧急执行 `VACUUM FREEZE` |
| 索引比表还大 | 索引死亡元组未清理 | `REINDEX` 重建索引 |

> 📎 来源：[PostgreSQL Documentation — VACUUM](https://www.postgresql.org/docs/current/sql-vacuum.html)

---

## 题目 5：MySQL 主从复制原理是什么？延迟原因有哪些？如何优化？

**参考答案：**

### 主从复制原理

MySQL 主从复制基于 **binlog** 实现，采用一主多从架构：

```
┌─────────────┐         binlog          ┌─────────────┐
│   Master    │ ──────────────────────►│   Slave     │
│             │   I/O Thread           │             │
│  - 写入数据 │   传输 binlog events   │  - I/O Thread│
│  - 记录    │ ──────────────────────►│  - SQL Thread│
│    binlog  │   Relay Log            │  - 回放 SQL  │
└─────────────┘                        └─────────────┘
```

**三个线程（Master 上 1 个，Slave 上 2 个）：**

| 线程 | 位置 | 职责 |
|------|------|------|
| **Binlog Dump** | Master | 读取 binlog，发送给 Slave I/O Thread |
| **I/O Thread** | Slave | 接收 binlog，写入本地 relay log |
| **SQL Thread** | Slave | 读取 relay log，在本地回放执行 |

**复制方式：**

| 方式 | 说明 | 配置参数 |
|------|------|----------|
| **异步复制** | Master 提交后不等待 Slave 返回 | 默认方式 |
| **半同步复制** | Master 等待至少一个 Slave 确认 | `rpl_semi_sync_master_wait_point = AFTER_COMMIT` |
| **全同步复制** | 所有 Slave 回放完才返回 | 性能差，实际很少用 |
| **延迟复制** | Slave 故意延迟 N 秒 | `MASTER_DELAY = N`（用于恢复误操作） |

### 主从延迟的常见原因

| 原因 | 说明 |
|------|------|
| **网络延迟** | Master 与 Slave 网络带宽不足或延迟高 |
| **Slave 机器性能差** | Slave 配置低，无法及时回放 |
| **大事务** | Master 一个事务修改行数太多，Slave 回放时间长 |
| **Slave 慢查询** | I/O 争用或 SQL Thread 排队 |
| **Binlog 格式** | Row 格式比 Statement 格式数据量大，传输慢 |
| **单线程回放** | MySQL 5.7 前 SQL Thread 单线程，5.7+ 可开启多线程（逻辑时钟） |
| **无主键更新** | UPDATE 全表扫描无索引，复制时更慢 |

### 优化方案

**1. 升级为并行复制（MySQL 5.7+）**

```sql
-- 开启基于 logical clock 的并行复制
SET GLOBAL slave_parallel_type = 'LOGICAL_CLOCK';
SET GLOBAL slave_parallel_workers = 8; -- 建议与 CPU 核心数匹配
```

**2. 拆分大事务**

```sql
-- 反例：一个事务删除1000万行
BEGIN;
DELETE FROM large_table WHERE created_at < '2025-01-01'; -- 1000万行
COMMIT;

-- 正例：分批删除，每批1000行
WHILE 1=1 LOOP
    DELETE FROM large_table WHERE id IN (
        SELECT id FROM large_table WHERE created_at < '2025-01-01' LIMIT 1000
    );
    IF SQL%ROWCOUNT = 0 THEN EXIT; END IF;
END LOOP;
```

**3. 使用 GTID 复制**

```sql
-- 开启 GTID（全局事务ID），方便故障转移
gtid_mode = ON
enforce_gtid_consistency = ON
```

**4. 读写分离 + 延迟感知**

业务层将读请求打到最近从库，或使用 `SELECT UNIX_TIMESTAMP() - UNIX_TIMESTAMP(relay_log_update_time)` 判断延迟。

**5. 升级到 MySQL 8.0**

8.0 的 **MGR（Group Replication）** 和 **InnoDB Cluster** 提供更完善的复制机制。

### 监控延迟

```sql
-- 查看从库延迟
SHOW SLAVE STATUS\G
-- Seconds_Behind_Master：延迟秒数（不准确，受网络/CPU影响）

-- 更精确：对比 GTID 位置
SELECT SOURCE_UUID, LAST_ERROR_MESSAGE FROM performance_schema.replication_connection_status;
```

> 📎 来源：[MySQL 官方文档 — Replication Implementation](https://dev.mysql.com/doc/refman/8.0/en/replication-implementation.html)

---

*本文件由 AI 面试题助手自动生成，每日更新*
*仓库地址：https://github.com/qq286158530/interview-questions*
