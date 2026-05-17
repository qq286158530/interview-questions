# 数据库性能优化面试题

> 更新日期：2026-05-17
> 涵盖：MySQL / PostgreSQL 索引优化、查询优化、存储引擎、缓存策略

---

## 题目 1：MySQL 中 InnoDB 的索引组织结构是什么？为什么主键应使用自增整型？

**答案：**

InnoDB 采用 **B+ 树** 作为索引结构，所有索引都是 B+ 树。表数据（聚簇索引）存储在主键索引的叶子节点中，每个叶子节点包含完整的行数据（Row）。

```
主键索引 B+ 树（聚簇索引）
├── 叶子节点 1 → [PK=1, row data]
├── 叶子节点 2 → [PK=2, row data]
└── 叶子节点 3 → [PK=3, row data]
```

**为什么主键应使用自增整型？**

1. **减少页分裂**：自增主键新插入的数据永远追加到索引树最右侧叶子节点末尾，不需要挪动已有数据，最大限度减少 **Page Split**（页分裂）。
2. **查询效率高**：整型比较比字符串比较快，CPU 指令少。
3. **占用空间小**：INT4 字节 vs UUID/字符串更小，索引体积更小，内存能容纳更多索引节点，缓存命中率更高。
4. **顺序写入**：顺序的主键使数据物理存储也是顺序的，磁盘顺序 I/O 效率远高于随机 I/O。

---

## 题目 2：什么情况下索引会失效？请列举至少 5 种场景并解释原因。

**答案：**

### 1. 使用函数或运算
```sql
-- 索引在 YEAR(create_time) 上失效
SELECT * FROM orders WHERE YEAR(create_time) = 2025;
```
原因：索引列参与了计算，数据库无法利用 B+ 树的有序性。

### 2. 类型转换
```sql
-- order_id 为 VARCHAR，传入整型
SELECT * FROM orders WHERE order_id = 12345;
```
原因：MySQL 会对字符串列做隐式类型转换，相当于在索引列上执行了函数。

### 3. LIKE 前缀通配符
```sql
SELECT * FROM products WHERE name LIKE '%手机';
```
原因：`*%`* 开头导致无法二分查找，索引有序性被破坏。

### 4. 范围条件右边列
```sql
-- 索引为 (status, created_at)
SELECT * FROM orders WHERE status > 3 AND created_at > '2025-01-01';
```
原因：status 范围查询之后，created_at 列的有序性被打断，无法使用索引。

### 5. 使用 OR 连接非索引列
```sql
SELECT * FROM users WHERE name = '张三' OR age = 25;
```
原因：age 列无索引时，OR 条件导致全表扫描。

### 6. 使用 NOT / != / IS NOT NULL
```sql
SELECT * FROM orders WHERE status IS NOT NULL;
SELECT * FROM orders WHERE status != 'paid';
```
原因：大部分存储引擎无法利用 B+ 树的有序性反转来快速定位。

### 7. 错误的字符集
```sql
-- 表字符集 utf8mb4，索引列校对字符集 utf8_general_ci
WHERE name = '你好' -- 可能不走索引
```

---

## 题目 3：如何优化慢查询？请描述完整的分析和优化流程。

**答案：**

### Step 1：定位慢查询
```sql
-- 开启慢查询日志（MySQL）
SET GLOBAL slow_query_log = 'ON';
SET GLOBAL long_query_time = 1;  -- 超过1秒记录
SET GLOBAL slow_query_log_file = '/var/log/mysql/slow.log';

-- 查看当前配置
SHOW VARIABLES LIKE 'slow_query%';
SHOW VARIABLES LIKE 'long_query_time';

-- PostgreSQL: 启用 pg_stat_statements
CREATE EXTENSION pg_stat_statements;
SELECT query, calls, mean_time, total_time 
FROM pg_stat_statements 
ORDER BY mean_time DESC LIMIT 10;
```

### Step 2：使用 EXPLAIN / EXPLAIN ANALYZE 分析执行计划
```sql
-- MySQL
EXPLAIN SELECT * FROM orders WHERE user_id = 100;
EXPLAIN ANALYZE SELECT * FROM orders WHERE user_id = 100;

-- PostgreSQL
EXPLAIN (ANALYZE, BUFFERS, FORMAT TEXT) SELECT * FROM orders WHERE user_id = 100;
```

重点关注：
- **type**：ref / range / ALL（ALL = 全表扫描）
- **key**：实际使用的索引
- **rows**：扫描行数估算
- **Extra**：Using filesort / Using temporary / Using index（覆盖索引）

### Step 3：根据执行计划优化
| 问题 | 优化手段 |
|------|----------|
| 全表扫描 | 造当添加索引 |
| Using filesort | 添加 ORDER BY 列的索引，或减少排序数据量 |
| Using temporary | 优化 JOIN 顺序，或拆分为简单查询 |
| 扫描行数过多 | 拆步查询，加 WHERE 条件缩小范围 |

### Step 4：验证优化效果
- 使用 `SHOW PROFILES`（MySQL 5.7）或 `SET profiling=1` 查看查询耗时。
- 多次执行确认稳定，避免「第一次慢」导致的误判（冷数据）。

---

## 题目 4：MySQL InnoDB 与 PostgreSQL 的 MVCC 实现有什么区别？

**答案：**

### MySQL InnoDB 的 MVCC
- 基于 **Undo Log** 实现。
- 每行数据有 2 个隐藏列：`DB_TRX_ID`（最近修改的事务ID）和 `DB_ROLL_PTR`（指向 Undo Log 的指针）。
- 读取数据时，根据事务的 `Read View`，通过 `DB_ROLL_PTR` 遍历 Undo Log 链，找到对当前事务可见的版本。
- **缺点**：长事务会导致 Undo Log 膨胀，大量数据版本链变长，读性能下降。

### PostgreSQL 的 MVCC
- 基于 **事务号（XID）** 和 **多版本快照（Snapshot）** 实现。
- 每行数据有 `xmin`（插入事务ID）和 `xmax`（删除事务ID）两个字段。
- 读取时判断：`xmin > Snapshot.xmax` 或 `xmin in Snapshot.xip_list` → 该行对当前事务不可见。
- **优点**：写不阻塞读，读不阻塞写（通过 HEAP 表的 TOAST 压缩和无覆盖写实现）。
- **缺点**：UPDATE 会产生新的行版本（Row Version），旧版本垃圾回收依赖 VACUUM。

### 核心对比

| 特性 | InnoDB | PostgreSQL |
|------|--------|------------|
| 多版本存储 | Undo Log（回滚段） | 行版本直接存在 HEAP |
| 垃圾回收 | Purge 线程 | VACUUM 进程 |
| 长事务影响 | Undo Log 膨胀 | 快照膨胀，膨胀 |
| 事务隔离级别 | 默认 REPEATABLE READ | 默认 READ COMMITTED |

---

## 题目 5：如何设计数据库缓存策略来提升读性能？请描述缓存分层和缓存失效处理。

**答案：**

### 缓存分层架构

```
应用层 → L1 本地缓存（Caffeine/Guava）→ L2 分布式缓存（Redis）→ 数据库
```

### 分层策略

| 层级 | 工具 | 适用场景 | TTL |
|------|------|----------|-----|
| L1 本地缓存 | Caffeine / Guava Cache | 访问极其频繁、数据几乎不变 | 几十秒～几分钟 |
| L2 分布式缓存 | Redis / Memcached | 跨节点共享、可动态更新 | 几分钟～几小时 |
| 数据库 | Query Cache（已废弃）/ Buffer Pool | 最终数据源 | 常驻内存 |

### 缓存读写策略

#### Cache-Aside（旁路缓存，最常用）
```
读：先读缓存，命中则返回；未命中则查数据库，写入缓存，返回数据。
写：先写数据库，再删除（而非更新）缓存。
```
> 删缓存而非更新：避免并发时缓存值与数据库不一致。

#### Write-Through
```
写：同时写缓存和数据库，任意一方失败则回滚。
读：缓存未命中则穿透到数据库。
```

#### Write-Behind
```
写：先写缓存，异步批量写数据库。
优点：写入性能极高；缺点：数据可能丢失。
```

### 缓存失效（击穿/穿透/雪崩）处理

| 问题 | 描述 | 解决方案 |
|------|------|----------|
| **缓存击穿** | 热 Key 过期，瞬间大量请求击穿到 DB | 互斥锁 / 永不过期 + 异步重建 |
| **缓存穿透** | 查询不存在的数据，大量请求打到 DB | 布隆过滤器 / 空值缓存 |
| **缓存雪崩** | 大量 Key 同时过期 | 过期时间加随机值 / 多级缓存 / Redis 集群 |
| **缓存热点** | 极端热点数据打爆单节点 | Redis Cluster + 读写分离 + 热点 key 分散 |

### Redis 缓存过期策略
- **volatile-lru**：从已设置过期时间的 key 中淘汰最近最少使用
- **allkeys-lru**：所有 key 中淘汰最近最少使用（最常用）
- **volatile-ttl**：淘汰即将过期的 key
- **noeviction**：不淘汰，满则返回 OOM（线上慎用）

---

## 参考来源

1. [MySQL 8.0 Reference Manual - InnoDB Indexes](https://dev.mysql.com/doc/refman/8.0/en/innodb-index-types.html)
2. [MySQL Performance: Using EXPLAIN](https://dev.mysql.com/doc/refman/8.0/en/using-explain.html)
3. [PostgreSQL Documentation - MVCC](https://www.postgresql.org/docs/current/mvcc.html)
4. [High Performance MySQL, 3rd Edition - B+ Tree Indexes](https://www.oreilly.com/library/view/high-performance-mysql/9781449314286/)
5. [Redis Documentation - Eviction Policies](https://redis.io/docs/reference/eviction/)