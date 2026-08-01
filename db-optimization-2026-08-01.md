# 数据库性能优化面试题

> 📅 日期：2026-08-01
> 📚 来源：MySQL 8.0 Reference Manual / PostgreSQL 18 Documentation / 各大技术社区

---

## 题目一：MySQL 索引优化——B+ 树索引原理与最佳实践

### 📌 题目

请解释 MySQL InnoDB 存储引擎中 B+ 树索引的工作原理，为什么它比哈希索引和 B 树索引更适合作为数据库的主索引？在实际开发中，创建复合索引时应该如何安排列的顺序？请结合具体场景说明。

### ✅ 答案

#### 1. B+ 树索引工作原理

InnoDB 使用 B+ 树作为默认索引结构，B+ 树是一种多路平衡查找树，具有以下特点：

**结构特点：**
- 所有数据记录都存储在叶子节点（Leaf Nodes）中，叶子节点之间通过双向链表连接
- 非叶子节点只存储索引键值和子节点指针，不存储实际数据
- 树的每个节点通常为 16KB（由 `innodb_page_size` 决定）
- 树高通常为 3~4 层，可以索引千万级数据

**查询过程：**
```
例如：SELECT * FROM users WHERE id = 10086
1. 从根节点开始，根据键值比较找到对应的子节点指针
2. 逐层向下遍历，直到找到叶子节点
3. 在叶子节点的双向链表中进行顺序查找
4. 找到目标记录后，通过主键回表查询完整数据
```

#### 2. 为什么 B+ 树比哈希索引更适合作为主索引？

| 特性 | B+ 树索引 | 哈希索引 |
|------|-----------|----------|
| **范围查询** | ✅ 支持，天然有序 | ❌ 不支持，只能精确匹配 |
| **排序** | ✅ 支持，B+ 树叶节点有序 | ❌ 不支持 |
| **最左前缀匹配** | ✅ 支持 | ❌ 不支持 |
| **等值查询速度** | O(log n) | O(1)，更快 |
| **数据量影响** | 稳定，B+ 树层高固定 | 可能有哈希冲突 |

**InnoDB 会自动为表的主键创建主键索引（Clustered Index）**，如果表没有主键，则选择第一个 NOT NULL 的唯一索引，如果都没有，InnoDB 会生成一个隐藏的 6 字节 `ROWID` 作为主键。

#### 3. 复合索引列顺序最佳实践

**核心原则：区分度高的列放在前面**

**案例说明：**
```sql
-- 假设有一个订单表
CREATE TABLE orders (
    id BIGINT PRIMARY KEY,
    user_id BIGINT NOT NULL,
    status TINYINT NOT NULL,  -- 1:待支付 2:已支付 3:已完成 4:已取消
    created_at DATETIME NOT NULL,
    INDEX idx_user_status (user_id, status)  -- 复合索引
);
```

**场景分析：**
- **高频查询 1：** `WHERE user_id = 100 AND status = 2`（查询某用户的已支付订单）
  - ✅ 复合索引完全覆盖，查询效率最高
  
- **高频查询 2：** `WHERE user_id = 100`（查询某用户所有订单）
  - ✅ 可以使用索引的最左前缀，快速定位该用户的所有订单
  
- **高频查询 3：** `WHERE status = 2`（查询所有已支付订单）
  - ❌ 无法使用复合索引（跳过了 user_id），需要全表扫描

**排序和分页优化：**
```sql
-- 如果经常按时间排序查询某个用户订单
INDEX idx_user_created (user_id, created_at)
-- 可以支持：WHERE user_id = 100 ORDER BY created_at DESC

-- 如果经常按状态和时间排序
INDEX idx_user_status_created (user_id, status, created_at)
```

**避免索引失效的做法：**
```sql
-- ❌ 在索引列上使用函数，导致索引失效
SELECT * FROM orders WHERE YEAR(created_at) = 2026;

-- ✅ 改写为范围查询，使用索引
SELECT * FROM orders WHERE created_at >= '2026-01-01' AND created_at < '2027-01-01';
```

**参考来源：**
- [MySQL 8.0 Reference Manual - InnoDB Indexes](https://dev.mysql.com/doc/refman/8.0/en/innodb-indexes.html)
- [MySQL 8.0 Reference Manual - CREATE INDEX](https://dev.mysql.com/doc/refman/8.0/en/create-index.html)

---

## 题目二：MySQL InnoDB 存储引擎与事务隔离级别

### 📌 题目

MySQL InnoDB 的默认隔离级别是什么？请详细解释 MySQL 的 4 种事务隔离级别及其各自能解决的并发问题（脏读、不可重复读、幻读），并说明 MVCC 机制是如何工作的。

### ✅ 答案

#### 1. InnoDB 默认隔离级别

**可重复读（REPEATABLE READ）** 是 InnoDB 的默认隔离级别。

#### 2. 四种隔离级别与并发问题

| 隔离级别 | 脏读（Dirty Read） | 不可重复读（Non-repeatable Read） | 幻读（Phantom Read） |
|---------|-------------------|-----------------------------------|---------------------|
| **读未提交（READ UNCOMMITTED）** | ✅ 可能发生 | ✅ 可能发生 | ✅ 可能发生 |
| **读已提交（READ COMMITTED）** | ❌ 不可能 | ✅ 可能发生 | ✅ 可能发生 |
| **可重复读（REPEATABLE READ）** | ❌ 不可能 | ❌ 不可能 | ❌ 不可能（InnoDB 通过 Next-Key Lock 解决） |
| **串行化（SERIALIZABLE）** | ❌ 不可能 | ❌ 不可能 | ❌ 不可能 |

**三种并发问题定义：**

1. **脏读：** 读取到其他事务未提交的数据
2. **不可重复读：** 同一个事务中，两次读取同一行数据，结果不同（因为其他事务修改并提交了）
3. **幻读：** 同一个事务中，两次执行相同查询，第二次看到了其他事务插入的新行

**InnoDB 如何解决幻读：**

InnoDB 在可重复读隔离级别下，使用 **Next-Key Lock（记录锁 + 间隙锁）** 来锁定索引区间，防止幻读：

```sql
-- 例如：SELECT * FROM orders WHERE status = 2 FOR UPDATE;
-- InnoDB 会锁定 status = 2 的所有记录，以及这些记录之间的"间隙"
-- 阻止其他事务在间隙中插入新记录
```

#### 3. MVCC（多版本并发控制）机制

**MVCC 的核心思想：** 每个事务在启动时看到数据库的一个"快照"（Snapshot），读取的是快照中的数据，而不是最新的数据。

**InnoDB 的 MVCC 实现：**

每行数据都有两个隐藏列：
- `DB_TRX_ID`：最近修改该行的事务 ID
- `DB_ROLL_PTR`：指向 undo log 中旧版本的指针

**读已提交（READ COMMITTED）下的 MVCC：**
```sql
-- 事务 A 启动，读取订单状态
BEGIN;
SELECT status FROM orders WHERE id = 1;  -- 此时 status = 1

-- 事务 B 修改该订单状态为 2 并提交
BEGIN;
UPDATE orders SET status = 2 WHERE id = 1;
COMMIT;

-- 事务 A 再次读取（READ COMMITTED 每次都获取最新快照）
SELECT status FROM orders WHERE id = 1;  -- 此时 status = 2，发生了不可重复读！
```

**可重复读（REPEATABLE READ）下的 MVCC：**
```sql
-- 事务 A 启动，快照时间点为 T1
BEGIN;

-- 事务 B 修改该订单状态为 2 并提交
BEGIN;
UPDATE orders SET status = 2 WHERE id = 1;
COMMIT;

-- 事务 A 再次读取，仍然读取快照中的旧数据
SELECT status FROM orders WHERE id = 1;  -- status = 1，未发生变化
COMMIT;
```

**ReadView 机制：**
- 事务启动时创建 ReadView，包含：当前活跃事务列表 `m_ids`、最小活跃事务 `min_trx_id`、最大活跃事务 `max_trx_id`
- 读取数据时，根据数据的 `DB_TRX_ID` 判断：
  - 如果 `DB_TRX_ID < min_trx_id`：数据在快照之前已提交，可见
  - 如果 `DB_TRX_ID` 在 `m_ids` 中：数据由未提交事务修改，不可见（需通过 undo log 查找旧版本）
  - 如果 `DB_TRX_ID > max_trx_id`：数据由未来事务修改，不可见

**MVCC + Next-Key Lock = 完整的并发控制：**
- MVCC 解决了读写并发问题（读不加锁）
- Next-Key Lock 解决了写写并发问题

**参考来源：**
- [MySQL 8.0 Reference Manual - InnoDB Locking](https://dev.mysql.com/doc/refman/8.0/en/innodb-locking.html)
- [MySQL 8.0 Reference Manual - Transaction Isolation Levels](https://dev.mysql.com/doc/refman/8.0/en/innodb-transaction-isolation-levels.html)

---

## 题目三：PostgreSQL EXPLAIN 执行计划分析

### 📌 题目

在 PostgreSQL 中，如何使用 EXPLAIN 和 EXPLAIN ANALYZE 分析查询性能？请解释以下执行计划中各节点的含义，并说明如何识别性能瓶颈（以 Seq Scan、Index Scan、Bitmap Heap Scan、Nested Loop 为例）。

### ✅ 答案

#### 1. EXPLAIN vs EXPLAIN ANALYZE

```sql
-- EXPLAIN：只显示执行计划，不实际执行
EXPLAIN SELECT * FROM orders WHERE user_id = 100;

-- EXPLAIN ANALYZE：执行查询并显示实际运行时间和代价
EXPLAIN ANALYZE SELECT * FROM orders WHERE user_id = 100;
```

#### 2. 执行计划节点解析

**示例表结构：**
```sql
CREATE TABLE orders (
    id BIGINT PRIMARY KEY,
    user_id BIGINT NOT NULL,
    status INT NOT NULL,
    total_amount DECIMAL(10,2) NOT NULL
);
CREATE INDEX idx_orders_user_id ON orders(user_id);
```

**示例执行计划：**
```sql
EXPLAIN ANALYZE
SELECT * FROM orders WHERE user_id = 100;
```

**结果示例：**
```
Bitmap Heap Scan on orders  (cost=4.30..16.50 rows=5 width=45) (actual time=0.031..0.038 rows=5 loops=1)
  Recheck Cond: (user_id = 100)
  ->  Bitmap Index Scan on idx_orders_user_id  (cost=0.00..4.30 rows=5 width=0) (actual time=0.018..0.018 rows=5 loops=1)
        Index Cond: (user_id = 100)
Planning Time: 0.542 ms
Execution Time: 0.087 ms
```

#### 3. 各节点含义

| 节点类型 | 含义 | 适用场景 |
|---------|------|---------|
| **Seq Scan（顺序扫描）** | 全表扫描，读取所有数据页 | 无索引、查询数据量大（>20%）、小表 |
| **Index Scan（索引扫描）** | 先查索引找到数据位置，再访问数据行 | 索引选择性好、返回数据量较少 |
| **Bitmap Heap Scan（位图堆扫描）** | 先用索引找到所有匹配行的位图，再批量获取数据 | 复合查询、返回数据量中等 |
| **Nested Loop（嵌套循环）** | 外层表驱动内层表，类似双重循环 | 小表驱动大表、内层有索引 |
| **Hash Join（哈希连接）** | 将小表哈希后放入内存，再遍历大表探测 | 等值连接、大表间连接 |
| **Merge Join（归并连接）** | 先对两表排序，再按顺序合并 | 已排序或需要排序的等值连接 |

**cost 参数含义（以 `cost=4.30..16.50` 为例）：**
- `4.30`：启动代价（startup cost），返回第一行之前需要的代价
- `16.50`：总代价（total cost），返回所有行的估计代价
- 代价单位是磁盘 page 读取的倍数（默认 `seq_page_cost = 1.0`）

**actual time 参数含义（以 `actual time=0.031..0.038` 为例）：**
- `0.031`：该节点开始到返回第一行的时间（毫秒）
- `0.038`：该节点返回所有行的时间（毫秒）

#### 4. 性能瓶颈识别

**瓶颈 1：大量数据的 Seq Scan**
```sql
-- 问题：查询用户状态时发生全表扫描
EXPLAIN ANALYZE SELECT * FROM orders WHERE status = 3;

-- 优化方案：为 status 列创建索引
CREATE INDEX idx_orders_status ON orders(status);

-- 如果 status 值分布极不均匀（大部分是已完成），考虑部分索引
CREATE INDEX idx_orders_status_pending ON orders(status) WHERE status = 1;
```

**瓶颈 2：Nested Loop 中被驱动表无索引**
```sql
-- 问题：Nested Loop 中内层表全表扫描
EXPLAIN ANALYZE
SELECT * FROM orders o, users u 
WHERE o.user_id = u.id AND u.city = '北京';

-- 优化方案：为 users.city 创建索引
CREATE INDEX idx_users_city ON users(city);

-- 或改写 SQL，使用 Hash Join
SET enable_nestedloop = off;  -- 临时禁用 Nested Loop
```

**瓶颈 3：Bitmap Heap Scan 返回大量数据**
```sql
-- 问题：Bitmap 索引扫描返回了太多行，效率不如 Index Scan
-- 识别：rows 数量远大于实际需要的数量

-- 优化方案：分析表统计信息，确保索引选择性强
ANALYZE orders;
-- 或创建更精确的复合索引
CREATE INDEX idx_orders_user_status ON orders(user_id, status);
```

**瓶颈 4：排序操作 Using filesort**
```sql
-- 问题：无法使用索引排序，需要额外的文件排序
EXPLAIN ANALYZE SELECT * FROM orders ORDER BY created_at DESC LIMIT 10;

-- 优化方案：创建覆盖索引
CREATE INDEX idx_orders_created ON orders(created_at DESC);

-- 如果已有复合索引：
CREATE INDEX idx_orders_user_created ON orders(user_id, created_at DESC);
```

**查看慢查询日志配置：**
```sql
-- 查看当前配置
SHOW log_min_duration_statement;

-- 设置慢查询阈值为 1 秒
ALTER SYSTEM SET log_min_duration_statement = '1s';

-- 重载配置
SELECT pg_reload_conf();
```

**参考来源：**
- [PostgreSQL 18 Documentation - Using EXPLAIN](https://www.postgresql.org/docs/current/using-explain.html)
- [PostgreSQL 18 Documentation - Performance Tips](https://www.postgresql.org/docs/current/performance-tips.html)

---

## 题目四：SQL 查询优化——JOIN vs 子查询 / 分页优化

### 📌 题目

在 SQL 查询中，什么时候应该使用 JOIN 替代子查询？请分析两者的执行计划差异。同时，请解释在大数据量分页查询中常见的性能问题及解决方案（以 MySQL 的 LIMIT offset, size 为例）。

### ✅ 答案

#### 1. JOIN vs 子查询

**子查询示例：**
```sql
SELECT * FROM orders 
WHERE user_id IN (
    SELECT id FROM users WHERE city = '北京'
);
```

**JOIN 改写：**
```sql
SELECT o.* FROM orders o
INNER JOIN users u ON o.user_id = u.id
WHERE u.city = '北京';
```

**执行计划差异分析：**

| 维度 | 子查询 | JOIN |
|------|-------|------|
| **执行次数** | 子查询可能执行多次（取决于优化器） | 通常只执行一次 |
| **临时表** | 某些情况下需要临时表存储子查询结果 | 数据在内存中联合 |
| **可读性** | 更直观，表达清晰的业务逻辑 | 需要理解表关系 |
| **NULL 值处理** | `IN` 对 NULL 值敏感（`NOT IN` 可能有陷阱） | `JOIN` 自动过滤 NULL |

**MySQL 优化器的处理：**

MySQL 5.6+ 对子查询有较大优化，通常会将不相关子查询（子查询不依赖外层查询）提升为派生表（Derived Table）进行优化：

```sql
-- MySQL 优化器可能将上述子查询改写为：
SELECT o.* FROM orders o
INNER JOIN (SELECT id FROM users WHERE city = '北京') u ON o.user_id = u.id;
```

**必须使用子查询的场景：**
```sql
-- 需要聚合后再筛选（子查询更自然）
SELECT user_id, COUNT(*) as order_count
FROM orders
GROUP BY user_id
HAVING COUNT(*) > 10;

-- 相关子查询（子查询依赖外层查询）
SELECT o.*, 
    (SELECT COUNT(*) FROM orders WHERE user_id = o.user_id) as user_order_count
FROM orders o;
```

#### 2. 大数据量分页性能问题

**问题分析：**

```sql
-- 常见分页查询（深分页问题）
SELECT * FROM orders ORDER BY id LIMIT 1000000, 10;
```

**执行过程：**
1. MySQL 先按索引排序（Using filesort 或索引排序）
2. 然后顺序扫描前 1,000,010 条记录，丢弃前 1,000,000 条
3. 返回最后 10 条

当 offset 很大时，前面的数据扫描变成了"无用功"，随着页数增加，性能急剧下降。

**解决方案一：游标分页（推荐）**

```sql
-- ❌ 传统 OFFSET 分页：每次查询都重新排序
SELECT * FROM orders ORDER BY id LIMIT 1000000, 10;

-- ✅ 游标分页：基于上一页最后一条记录的 ID
-- 上一页最后一条：id = 1000000
SELECT * FROM orders 
WHERE id > 1000000 
ORDER BY id 
LIMIT 10;
```

**对比：**
- OFFSET 分页：随着 offset 增大，时间复杂度接近 O(n)
- 游标分页：时间复杂度恒定 O(log n + 10)

**解决方案二：延迟关联**

```sql
-- 先通过索引获取主键，再关联获取完整数据
SELECT o.* FROM orders o
INNER JOIN (
    SELECT id FROM orders 
    ORDER BY id 
    LIMIT 1000000, 10
) t ON o.id = t.id;
```

**解决方案三：记录总数缓存**

```sql
-- 前端不显示总页数，只显示"下一页"
-- 或定期缓存总数，减少 COUNT(*) 开销

-- 异步获取总数
SELECT COUNT(*) as total FROM orders;  -- 定期执行，缓存结果
```

**解决方案四：倒序分页优化**

```sql
-- 如果经常需要访问"最新数据"
CREATE INDEX idx_orders_id_desc ON orders(id DESC);

-- 查询第 N 页（倒序）
SELECT * FROM orders 
ORDER BY id DESC 
LIMIT 10 OFFSET 100;

-- 改写为游标形式
SELECT * FROM orders 
WHERE id < 100  -- 上一页最小 id
ORDER BY id DESC 
LIMIT 10;
```

**PostgreSQL 的优化：OFFSET 使用 keyset pagination**

```sql
-- PostgreSQL 同样推荐游标分页
-- 代替：OFFSET 1000000

-- 使用 keyset：
WHERE id < 1000000 ORDER BY id DESC LIMIT 10
```

**参考来源：**
- [MySQL 8.0 Reference Manual - Optimizing Queries with EXPLAIN](https://dev.mysql.com/doc/refman/8.0/en/optimizing-queries-with-explain.html)
- [PostgreSQL Wiki - Pagination](https://wiki.postgresql.org/wiki/Limitithmetic)

---

## 题目五：数据库缓存策略与连接池优化

### 📌 题目

请描述一个完整的数据库缓存策略，包括：数据库内置缓存（Query Cache、Buffer Pool）、应用层缓存（Redis）的使用场景，以及何时应该使用缓存而非直接查询数据库。同时请说明数据库连接池的工作原理及常见配置参数对性能的影响。

### ✅ 答案

#### 1. 数据库缓存体系

```
┌─────────────────────────────────────────────────────┐
│                    应用层                            │
│  ┌─────────┐   ┌─────────┐   ┌─────────┐           │
│  │  Redis  │   │ Memcached │  │ 本地缓存 │           │
│  └────┬────┘   └────┬────┘   └────┬────┘           │
└───────┼─────────────┼─────────────┼──────────────────┘
        │             │             │
        ▼             ▼             ▼
┌─────────────────────────────────────────────────────┐
│                   数据库层                            │
│  ┌─────────────┐  ┌─────────────┐  ┌────────────┐ │
│  │ Query Cache │  │ Buffer Pool │  │ Log Buffer │ │
│  │  (MySQL)    │  │   (InnoDB)   │  │  (WAL)     │ │
│  └─────────────┘  └─────────────┘  └────────────┘ │
└─────────────────────────────────────────────────────┘
```

#### 2. MySQL Query Cache（已移除，了解历史）

**说明：** MySQL 8.0 已移除 Query Cache，了解其设计思想即可。

- **原理：** 缓存完整 SELECT 查询结果及其结果集
- **命中条件：** 查询语句完全一致（包括空格、大小写）
- **失效时机：** 任何对相关表的操作（INSERT/UPDATE/DELETE）都会使该表所有缓存失效
- **问题：** 在高并发写入场景下，缓存频繁失效，命中率极低

**替代方案：** 使用应用层缓存（Redis）

#### 3. InnoDB Buffer Pool（核心缓存）

**Buffer Pool 是 InnoDB 最重要的内存区域**，用于缓存表数据和索引。

```sql
-- 查看 Buffer Pool 大小
SHOW VARIABLES LIKE 'innodb_buffer_pool_size';

-- 设置 Buffer Pool 大小（通常设置为物理内存的 60-80%）
SET GLOBAL innodb_buffer_pool_size = 8589934592;  -- 8GB
```

**Buffer Pool 管理机制：**
```
最近最少使用（LRU）淘汰策略：

[ New Sublist (young) ] ←──── 热点数据（最近访问）
         │
         │  访问时移动
         ▼
[ Old Sublist (old) ]
         │
         │ 淘汰时移除
         ▼
热点数据保护：默认前 37% (3/8) 为 New，不会被轻易淘汰
```

**Buffer Pool 相关参数优化：**
```sql
-- 将 Buffer Pool 划分为多个实例，减少锁竞争
innodb_buffer_pool_instances = 4  -- 多核 CPU 时建议设置

-- 允许在线调整 Buffer Pool 大小（MySQL 5.7+）
SET GLOBAL innodb_buffer_pool_size = 12884901888;  -- 12GB

-- 预热：服务器重启后加载热点数据到内存
-- 方式1：保存 Buffer Pool 快照
SELECT innodb_buffer_pool_dump_now = 1;  -- 导出
SELECT innodb_buffer_pool_load_now = 1;  -- 导入

-- 方式2：启动时自动加载
innodb_buffer_pool_load_at_startup = ON
```

#### 4. Redis 应用层缓存

**使用场景：**

| 场景 | 是否使用缓存 | 原因 |
|------|-------------|------|
| **频繁读取，少量写入的数据** | ✅ 是 | 缓存命中率高，减少数据库压力 |
| **实时性要求不高的数据** | ✅ 是 | 如商品详情页、配置数据 |
| **计算密集型查询** | ✅ 是 | 如排行榜、统计数据 |
| **强一致性数据** | ❌ 否 | 如库存、余额、扣减类操作 |
| **高频写入的计数器** | ❌ 否 | 缓存一致性问题复杂 |

**缓存模式：**

```sql
-- 模式1：Cache-Aside（旁路缓存，最常用）
-- 读：先查 Redis，未命中则查数据库，再写入 Redis
-- 写：先更新数据库，再删除 Redis（而非更新）


-- 模式2：Read-Through（读穿透）
-- 应用只访问缓存层，缓存层自动加载数据库


-- 模式3：Write-Through（写穿透）
-- 数据同时写入缓存和数据库
```

**缓存问题：**

1. **缓存穿透：** 查询不存在的数据穿透到数据库
   - 解决：布隆过滤器 / 缓存空值（设置短 TTL）

2. **缓存击穿：** 热点 key 过期瞬间，大量请求打到数据库
   - 解决：互斥锁 / 永不过期（异步更新）

3. **缓存雪崩：** 大量 key 同时过期或 Redis 宕机
   - 解决：过期时间加随机值 / Redis 高可用部署

#### 5. 数据库连接池

**为什么需要连接池？**

- 建立 TCP 连接 + 认证：通常需要 10~100ms
- 数据库连接是有限资源（MySQL 默认 max_connections = 151）
- 频繁创建销毁连接会造成巨大开销

**连接池工作原理：**
```
应用启动 → 连接池初始化 N 个连接 → 请求到来获取连接 → 使用完毕归还连接

连接复用：请求1 使用连接 → 归还 → 请求2 获取同一连接
```

**核心配置参数：**

| 参数 | 说明 | 建议值 |
|------|------|--------|
| **最大连接数** | 连接池能持有的最大连接数 | CPU 核心数 × 2 + 磁盘数 |
| **最小空闲连接** | 空闲时保持的最小连接数 | 业务峰值的 20-30% |
| **连接超时** | 获取连接的最大等待时间 | 10~30 秒 |
| **空闲超时** | 空闲连接的最大存活时间 | 10~30 分钟 |
| **心跳检测** | 定期检测连接是否有效 | 30~60 秒 |
| **连接复用检测** | 归还连接前检查连接状态 | 配置 testOnReturn |

**常见连接池：**
- **HikariCP（Java）：** 性能最优，Spring Boot 2.x 默认
- **Druid（阿里巴巴）：** 监控能力强，适合 DBA 监控
- **PGBouncer（PostgreSQL）：** 服务端连接池，减少连接数

**MySQL 连接数配置：**
```sql
-- 查看当前连接数
SHOW STATUS LIKE 'Threads_connected';
SHOW VARIABLES LIKE 'max_connections';

-- 设置最大连接数
SET GLOBAL max_connections = 500;

-- 建议配置（my.cnf）
max_connections = 500
wait_timeout = 600          -- 空闲连接超时（秒）
interactive_timeout = 600  -- 交互式连接超时
```

**参考来源：**
- [MySQL 8.0 Reference Manual - InnoDB Buffer Pool](https://dev.mysql.com/doc/refman/8.0/en/innodb-buffer-pool.html)
- [PostgreSQL 18 Documentation - Managing Kernel Resources](https://www.postgresql.org/docs/current/kernel-resources.html)

---

## 📚 推荐阅读

1. **MySQL 8.0 Reference Manual - Optimization**
   https://dev.mysql.com/doc/refman/8.0/en/optimization.html

2. **PostgreSQL 18 Documentation - Performance Tips**
   https://www.postgresql.org/docs/current/performance-tips.html

3. **Percona Blog - MySQL Performance Optimization**
   https://www.percona.com/blog/

4. **《高性能 MySQL》第 3 版** - Baron Schwartz 等著

---

> 🔖 本面试题库由自动化脚本每日更新 | GitHub: qq286158530/interview-questions
