# 数据库性能优化面试题集

> 📅 更新时间：2026-06-15  
> 🎯 涵盖：MySQL / PostgreSQL 索引优化、查询优化、存储引擎、缓存策略等核心知识点

---

## 题目 1：MySQL 中 InnoDB 存储引擎的索引结构是什么？它是如何实现高效查找的？

### 参考答案

**InnoDB 使用 B+ 树作为索引结构**，而非 B 树或哈希索引，这一点是面试中的高频考点。

**B+ 树的特点：**

- 是一种平衡多叉树，所有数据都存储在叶子节点，叶子节点之间通过双向链表连接
- 非叶子节点只存储索引键值和子节点指针，不存储实际数据
- 树高通常为 3~4 层，可支撑千万级数据的单次查询（只需 3~4 次磁盘 I/O）

**InnoDB 的两类索引：**

| 类型 | 说明 |
|------|------|
| **聚簇索引 (Clustered Index)** | 主键索引，叶子节点存储完整行数据（索引即数据） |
| **辅助索引 (Secondary Index)** | 非主键索引，叶子节点存储主键值，查询时需回表 |

**查找过程示例：**

```
SELECT * FROM users WHERE name = '张三';
```

1. 先在 `name` 的辅助索引树中找到对应叶子节点，获取主键值（如 `id=10086`）
2. 再回到主键聚簇索引树，根据主键找到完整行数据（回表）
3. 如果 SELECT 只查主键和 name，则无需回表（索引覆盖）

**为什么不用哈希索引？**
哈希索引适合等值查询（O(1)），但无法支持范围查询（`WHERE age > 20`）、排序和最左前缀匹配等场景。

**性能优化建议：**
- 主键不宜过长（建议使用自增主键或 UUID 的短形式），否则所有辅助索引都会变大
- 避免在频繁查询的字段上建太多单字段索引，结合复合索引的最左前缀原则优化

---

📚 **参考来源**
- [MySQL 官方文档 - InnoDB Indexes](https://dev.mysql.com/doc/refman/8.4/en/innodb-indexes.html)
- [MySQL 官方文档 - Clustered and Secondary Indexes](https://dev.mysql.com/doc/refman/8.4/en/innodb-index-types.html)

---

## 题目 2：如何排查和解决 MySQL 查询慢的问题？请写出主要思路和关键 SQL。

### 参考答案

**Step 1：开启慢查询日志，定位慢 SQL**

```sql
-- 查看慢查询是否开启
SHOW VARIABLES LIKE 'slow_query_log';

-- 设置慢查询阈值（如 1 秒）
SET GLOBAL slow_query_log_time = 1;

-- 查看最近 5 条慢查询记录
SELECT * FROM mysql.slow_log ORDER BY start_time DESC LIMIT 5;
```

**Step 2：使用 EXPLAIN 分析执行计划**

```sql
EXPLAIN SELECT * FROM orders WHERE status = 1 AND create_time > '2026-01-01';
```

重点关注字段：

| 字段 | 含义 | 判断标准 |
|------|------|----------|
| `type` | 连接类型 | `ALL`（全表扫描）需优化，`range` 及以上较好 |
| `key` | 实际使用的索引 | 若为 NULL 说明没用上索引 |
| `rows` | 扫描行数估算 | 越大越需优化 |
| `Extra` | 附加信息 | 出现 `Using filesort`/`Using temporary` 需注意 |

**Step 3：检查索引是否生效**

```sql
-- 查看表的所有索引
SHOW INDEX FROM orders;

-- 使用 FORCE INDEX 强制使用某索引（诊断用）
SELECT * FROM orders FORCE INDEX (idx_status) WHERE status = 1;
```

**Step 4：分析业务和 SQL 写法**

常见导致慢查询的原因：

- **SELECT ***：取了很多不需要的字段，增加网络传输和回表开销
- **隐式类型转换**：如 `WHERE phone = 13800138000`（phone 是 varchar），导致索引失效
- **模糊查询以 % 开头**：如 `WHERE name LIKE '%张'`，无法使用索引
- **OR 连接条件**：两边字段类型或编码不一致会导致索引失效
- **深分页问题**：`LIMIT 10000, 10` 需要先扫描 10010 行，推荐使用**游标分页**：

```sql
-- 低效（深分页）
SELECT * FROM orders ORDER BY id LIMIT 10000, 10;

-- 高效（游标分页）
SELECT * FROM orders WHERE id > #{last_id} ORDER BY id LIMIT 10;
```

**Step 5：使用 SHOW PROCESSLIST 观察实时连接**

```sql
SHOW PROCESSLIST;
-- 查看正在执行的 SQL，找出长时间运行的查询后 KILL 掉
```

---

📚 **参考来源**
- [MySQL 官方文档 - EXPLAIN Output Format](https://dev.mysql.com/doc/refman/8.4/en/explain-output.html)
- [MySQL 官方文档 - Slow Query Log](https://dev.mysql.com/doc/refman/8.4/en/slow-query-log.html)

---

## 题目 3：PostgreSQL 中如何利用 EXPLAIN ANALYZE 分析查询性能？举一个实际案例。

### 参考答案

`EXPLAIN ANALYZE` 是 PostgreSQL 中最重要的性能分析工具，它会实际执行 SQL（可加 `ANALYZE OFF` 只预估）并返回详细的执行信息。

**基础用法：**

```sql
EXPLAIN (ANALYZE, BUFFERS, FORMAT TEXT)
SELECT o.id, o.total_amount, u.name
FROM orders o
JOIN users u ON o.user_id = u.id
WHERE o.status = 1 AND o.create_time > '2026-01-01';
```

**关键输出字段解析：**

| 字段 | 含义 |
|------|------|
| `actual time` | 实际执行时间（毫秒）|
| `rows` | 实际返回行数 |
| `loops` | 该节点被执行次数 |
| `Buffers: shared hit` | 从缓存读取的块数 |
| `Buffers: read` | 从磁盘读取的块数（越少越好）|

**实战案例：为什么这条 SQL 慢？**

```sql
EXPLAIN (ANALYZE)
SELECT * FROM products
WHERE category = 'electronics'
  AND price BETWEEN 100 AND 500
ORDER BY stock DESC;
```

输出示例：

```
Sort  (cost=15230.50..15400.20 rows=67880 width=124)
      (actual time=245.330..248.120 rows=1520 loops=1)
  Sort Key: stock DESC
  Sort Method: quicksort  Memory: 1234kB
  ->  Seq Scan on products  (cost=0.00..8900.00 rows=67880 width=124)
        (actual time=0.045..230.110 rows=1520 loops=1)
        Filter: ((category = 'electronics') AND (price >= 100) AND (price <= 500))
        Rows Removed by Filter: 684360
```

**问题分析：**
- `Seq Scan on products` 说明走了全表扫描，没有利用索引
- `Rows Removed by Filter: 684360` 说明过滤掉了 68 万行无效数据
- `Sort` 节点对 1520 行做了排序，开销不大

**优化方案：创建复合索引**

```sql
CREATE INDEX idx_products_category_price_stock
ON products(category, price, stock DESC);
```

创建后再次 EXPLAIN ANALYZE，如果 `Seq Scan` 变为 `Index Scan` 或 `Index Only Scan`，且 `Buffers: read` 大幅下降，说明优化有效。

**BUFFERS 选项的重要性：**
- `Buffers: shared hit` 表示从 PostgreSQL 共享缓冲区命中（内存），性能好
- `Buffers: read` 表示从磁盘读取，性能差
- 如果 read 块数很多，考虑增加 `shared_buffers` 配置或优化 SQL 减少扫描量

---

📚 **参考来源**
- [PostgreSQL 官方文档 - Using EXPLAIN](https://www.postgresql.org/docs/current/using-explain.html)
- [PostgreSQL 官方文档 - Planning and Diagnosis](https://www.postgresql.org/docs/current/performance-tips.html)

---

## 题目 4：Redis 缓存与数据库如何配合使用？什么是缓存穿透、缓存击穿、缓存雪崩？如何解决？

### 参考答案

这是**缓存策略**模块最常考的综合性问题，结合了架构设计和实际问题的解决能力。

**三种常见缓存问题：**

---

### 🕳️ 缓存穿透（Cache Penetration）

**问题：** 查询一个根本不存在的数据（数据既不在缓存也不在 DB），请求直接打到数据库。

**危害：** 大量恶意或异常请求击穿缓存，持续冲击数据库。

**解决方案：**

| 方案 | 做法 |
|------|------|
| **布隆过滤器 (Bloom Filter)** | 将所有合法数据的 key 存入布隆过滤器，查询前先判断是否存在 |
| **缓存空值** | 将查询结果为空的数据也缓存起来（value = null），设置较短过期时间 |
| **参数合法性校验** | 对 ID、phone 等字段做格式校验，过滤明显非法的请求 |

---

### ⚡ 缓存击穿（Cache Breakdown）

**问题：** 某个热点 key 突然过期，导致大量并发请求同时去查询数据库。

**场景：** 秒杀商品、爆款新闻等高并发热点 key。

**解决方案：**

| 方案 | 做法 |
|------|------|
| **互斥锁 (Mutex)** | 使用 Redis SETNX / Redisson 实现分布式锁，只允许一个请求去查 DB |
| **热点数据永不过期** | 通过逻辑过期（如 value 中存储 `data + expire_time`），后台异步更新 |
| **多级缓存** | 本地缓存（Caffeine/Guava） + Redis + DB，多层挡请求 |

---

### ❄️ 缓存雪崩（Cache Avalanche）

**问题：** 大面积缓存同时过期，或 Redis 集群宕机，导致数据库瞬间压力暴增。

**解决方案：**

| 方案 | 做法 |
|------|------|
| **过期时间随机化** | 给缓存过期时间加上随机值（如 `baseTTL + random(0, 300s)`），避免同时过期 |
| **Redis 高可用架构** | 使用 Redis Sentinel（主从 + 自动故障转移）或 Redis Cluster 避免全量宕机 |
| **数据库高可用** | 做读写分离、分库分表，提升数据库层的抗压能力 |
| **服务熔丝限流** | 使用 Hystrix / Sentinel 对数据库查询做限流，避免数据库过载 |

---

**完整的缓存读写模式（Cache-Aside）：**

```python
def get_user(user_id):
    # 1. 先查缓存
    user = redis.get(f"user:{user_id}")
    if user:
        return json.loads(user)
    
    # 2. 缓存未命中，查数据库
    user = db.query("SELECT * FROM users WHERE id = %s", user_id)
    
    # 3. 写入缓存（设置过期时间，防止雪崩加随机值）
    if user:
        redis.setex(f"user:{user_id}", 3600 + random.randint(0, 300), json.dumps(user))
    
    return user
```

---

📚 **参考来源**
- [Redis 官方文档 - Redis Architecture](https://redis.io/docs/about/)
- [Google Cloud - Redis Cache Patterns](https://cloud.google.com/blog/products/databases/redis-caching-best-practices)
- [Martin Fowler - Caching Patterns](https://martinfowler.com/articles/patterns-of-distributed-systems/)

---

## 题目 5：MySQL 的 MVCC 机制是什么？它是如何解决幻读问题的？

### 参考答案

**MVCC（Multi-Version Concurrency Control，多版本并发控制）** 是 MySQL InnoDB 实现高性能并发读写的核心技术，被追问时能拉开差距。

**核心概念：**

- 每行数据有两个隐藏字段：`DB_TRX_ID`（最近一次修改的事务ID）和 `DB_ROLL_PTR`（指向 undo log 的指针）
- InnoDB 为每个事务维护一个** Read View（读视图）**，记录当前活跃事务 ID 列表
- 不同事务看到的数据版本不同，实现**快照读**，无需加锁

**Read Committed (RC) vs Repeatable Read (RR)：**

| 隔离级别 | 快照读行为 |
|----------|-----------|
| **RC（已提交读）** | 每次 SELECT 都生成新的 Read View，可能看到其他事务已提交的修改 |
| **RR（可重复读）** | 事务开始时生成 Read View，整个事务内读到的数据版本一致 |

**幻读（Phantom Read）是什么？**

> 同一个事务内，两次相同条件的 SELECT 结果集行数不一致（因为其他事务插入了新行）。

**InnoDB 如何解决幻读？**

1. **MVCC + 快照读**：普通 SELECT 是快照读，读的是事务开始时的数据快照，事务内的两次查询结果一致，不会看到新插入的行

2. **Next-Key Lock（临键锁）**：对于当前读（`SELECT ... FOR UPDATE`、`INSERT`、`UPDATE`、`DELETE`），InnoDB 使用 Next-Key Lock 锁住索引区间（左闭右开），防止其他事务在区间内插入新记录

```sql
-- 事务 A：锁定 id 在 (10, 20] 区间
SELECT * FROM orders WHERE id > 10 AND id <= 20 FOR UPDATE;
-- 事务 B：尝试插入 id=15 的记录，会被阻塞直到事务 A 提交
```

**Next-Key Lock 的结构：**

- `Record Lock`：记录锁，锁住具体的索引记录
- `Gap Lock`：间隙锁，锁住索引之间的空隙，防止插入
- 两者结合 = **Next-Key Lock**（临键锁）

**注意：** RR 级别下 Next-Key Lock 生效；RC 级别下只使用 Record Lock，无幻读问题（因为每次快照都重新生成）。

---

📚 **参考来源**
- [MySQL 官方文档 - InnoDB Multi-Versioning](https://dev.mysql.com/doc/refman/8.4/en/innodb-multi-versioning.html)
- [MySQL 官方文档 - InnoDB Locking](https://dev.mysql.com/doc/refman/8.4/en/innodb-locking.html)
- [MySQL 官方文档 - Consistent Non-Locking Reads](https://dev.mysql.com/doc/refman/8.4/en/innodb-consistent-read.html)

---

> 📌 **面试技巧提示**：题目 1~2 侧重 MySQL，题目 3 考 PostgreSQL，题目 4 是缓存与数据库联合作战，题目 5 深入原理。准备时可以先理解原理，再用 `EXPLAIN` / `EXPLAIN ANALYZE` 在本地跑一遍，效果更好。祝面试顺利 🚀