# 数据库性能优化面试题

> 📅 更新时间：2026-08-08
> 🗄️ 涵盖：MySQL / PostgreSQL 索引优化、查询优化、存储引擎、缓存策略

---

## 题目一：MySQL 中 B+Tree 索引的高度通常为 3~4 层，为什么还能支撑千万级数据量的高速查询？

### 参考答案

B+Tree 索引的高度指的是从根节点到叶子节点的路径长度。每一次磁盘 I/O 就是一次树层的"翻页"查找。

**为什么层数这么少？**

- InnoDB 默认 `PAGE_SIZE = 16KB`
- 假设主键为 `BIGINT`（8 字节），每个非叶子节点可存储约 **1170 个指针**（16KB / (8 + 6) ≈ 1170，6 是指针开销）
- 根节点可指向约 1170 个第二层节点，每个第二层节点再指向 1170 个第三层节点……
- **3 层 B+Tree 的最大指针容量**：
  - 第一层（根）: 1 个节点
  - 第二层: 1170 个节点
  - 第三层（叶子层）: 1170 × 1170 ≈ **136.9 万**个叶子页
  - 每个叶子页约存 16KB / (主键8字节 + 行数据约15~200字节) ≈ **100~200 行**
  - 总计可索引 **1.5 亿 ~ 2.7 亿**条记录

**千万级数据为何还快？**

- 只需要 **3 次磁盘 I/O**（根 → 中间层 → 叶子层）就能定位到目标记录
- 相比全表扫描（可能需要几十万次 I/O），性能提升千倍以上
- 叶子节点之间通过双向链表相连，范围查询（`BETWEEN`、`LIKE`）只需一次定位后顺序遍历

**面试加分点**：如果数据分布稀疏（索引字段选择性强），实际扫描行数很少；如果回表代价高，可以通过**覆盖索引**避免回表。

---

## 题目二：有一条 SQL 执行很慢，如何使用 EXPLAIN 分析并定位问题？请详细说明 EXPLAIN 各字段的含义。

### 参考答案

```sql
EXPLAIN SELECT * FROM orders WHERE customer_id = 100 AND status = 'paid' ORDER BY created_at DESC LIMIT 10;
```

### 关键字段解读

| 字段 | 含义 | 正常值 / 问题信号 |
|------|------|-------------------|
| **type** | 访问类型，描述 MySQL 如何查找行 | `const/eq_ref/ref/range` 为佳；`ALL` 表示全表扫描 🚨 |
| **key** | 实际使用的索引 | 若为 `NULL` 说明没走索引 🚨 |
| **rows** | MySQL 估算需要扫描的行数 | 数值越大性能越差；上万就值得优化 |
| **Extra** | 额外信息 | 包含 `Using filesort`（需额外排序）🚨、`Using temporary`（建临时表）🚨、`Using index condition`（索引下推）|

### 常见问题及解决方案

**问题 1：`type = ALL`（全表扫描）**
```sql
-- 解决：为 WHERE 条件列创建索引
ALTER TABLE orders ADD INDEX idx_customer_status (customer_id, status, created_at);
```

**问题 2：`Extra = Using filesort`（内存/磁盘排序）**
```sql
-- 解决：将排序列加入联合索引尾部，使其本身有序
ALTER TABLE orders ADD INDEX idx_customer_status_date (customer_id, status, created_at DESC);
```

**问题 3：回表过多**
```sql
-- 解决：建立覆盖索引，所有需要的列都包含在索引中，无需回表
ALTER TABLE orders ADD INDEX idx_cover (customer_id, status, created_at, id, amount);
-- 查询改为覆盖索引查询：
SELECT id, amount, created_at FROM orders WHERE customer_id = 100 AND status = 'paid' ORDER BY created_at DESC LIMIT 10;
```

---

## 题目三：PostgreSQL 与 MySQL（InnoDB）在 MVCC 机制上有何不同？这对性能优化有什么实际影响？

### 参考答案

### 核心区别

| 特性 | MySQL (InnoDB) | PostgreSQL |
|------|---------------|------------|
| **实现方式** | 回滚段（Undo Log）| 闲杂事务视图（每行 `xmin/xmax` 版本链）|
| **隔离级别实现** | 基于 Undo 的 ReadView | 基于元组版本 + 快照 |
| **更新行为** | 写操作对记录加排他锁；旧版本写入 Undo | 写操作创建新版本元组，旧版本保留；采用 **Append-Only** |
| **垃圾回收** | 由 `purge` 线程异步清理 | 由 `VACUUM` 进程清理dead tuple |

### 对性能的实际影响

**1. 写放大问题**
- PostgreSQL 的 Append-Only 特性意味着频繁 UPDATE 会导致表膨胀（bloat），需要定期 `VACUUM`，否则查询会扫描大量无用版本链
- 优化方法：合理配置 `autovacuum`，或对高频更新表调低 `autovacuum_vacuum_scale_factor`

**2. 读性能差异**
- InnoDB 在多数读场景下直接读最新数据（除非显式开启事务隔离），不需要像 PG 那样在版本链中查找可见版本
- 但在 `REPEATABLE READ` 隔离级别下，InnoDB 的长事务会导致 Undo 链过长，增加 undo page 占用

**3. 索引失效场景**
- PostgreSQL 中 HOT（Heap-Only Tuple）更新可以避免更新索引（如果更新不影响索引列），减少索引维护开销
- MySQL 的 InnoDB 更新索引列时必须同步更新所有相关索引

**面试加分**：如果面试官追问，可补充 PostgreSQL 14+ 的 `VACUUM` 改进，以及 MySQL 8.0 的 `Instant ADD COLUMN` 优化。

---

## 题目四：如何设计一个高效的数据库缓存策略？Redis + MySQL 场景下，如何解决缓存穿透、缓存击穿、缓存雪崩三大问题？

### 参考答案

### 三大问题及解决方案

#### 1. 缓存穿透（查询不存在的数据）

**问题**：大量请求查询 DB 和 Cache 都不存在的 key，每次都打到数据库。

**解决方案：**

```python
# 方案一：布隆过滤器（Bloom Filter）
# 将所有存在的 key 存入 Bloom Filter，请求进来先查 Bloom Filter
# 存在 → 查 Redis → 查 DB；不存在 → 直接返回空

# 方案二：缓存空值（短期）
if not found_in_db:
    redis.setex(f"empty:user:{user_id}", 300, "NULL")  # 5分钟过期
```

#### 2. 缓存击穿（热点 key 过期瞬间）

**问题**：某个热点 key 过期瞬间，大量并发请求全部打到 DB。

**解决方案：**

```python
# 方案一：互斥锁（Redis SETNX）
lock = redis.setnx(f"lock:product:{product_id}", "1")
if lock:
    data = db.query(...)
    redis.setex(f"product:{product_id}", 3600, json.dumps(data))
    redis.delete(f"lock:product:{product_id}")
else:
    time.sleep(0.05)  # 短暂等待后重试
    return redis.get(f"product:{product_id}")

# 方案二：永不过期 + 异步重建（逻辑过期）
# value 中存 "数据 + 逻辑过期时间"，命中时检查是否过期
# 若过期则开启独立线程重建缓存，主线程返回旧数据
```

#### 3. 缓存雪崩（大量 key 同时过期）

**问题**：大量 key 设置了相同的过期时间，同时过期导致瞬间大量请求打到 DB。

**解决方案：**

```python
# 方案一：过期时间加随机偏移
expire_time = base_expire + random.randint(0, 300)  # 5分钟随机偏移

# 方案二：Redis 持久化 + 预热
# RDB / AOF 保障故障恢复，启动时主动加载热点数据

# 方案三：高可用架构
# 主从 + Sentinel 保障 Cache 可用性，避免单点故障引发雪崩
```

### 多级缓存架构建议

```
浏览器 → CDN → Nginx(本地缓存) → Redis集群 → MySQL
```

- **热点数据**：Redis Cluster + 合理分片
- **本地缓存**：进程内 LRU Cache（如 Guava/Caffeine）减少 Redis 请求
- **预加载**：系统启动或低峰期主动加载热点数据

---

## 题目五：PostgreSQL 的 EXPLAIN ANALYZE 与 MySQL 的 EXPLAIN 有什么区别？如何用 EXPLAIN ANALYZE 优化一个慢查询？

### 参考答案

### MySQL vs PostgreSQL 的 EXPLAIN 区别

| 维度 | MySQL | PostgreSQL |
|------|-------|------------|
| **EXPLAIN** | 仅显示估算的执行计划，不执行 | 仅显示估算计划，不执行 |
| **EXPLAIN ANALYZE** | 5.6 及以上支持（实际执行并返回真实数据）| 原生支持，执行并显示真实耗时 |
| **输出信息** | 估算行数、成本 | 估算 + 实际行数、实际耗时、循环次数 |
| **Buffer 统计** | 无专门字段 | `Buffers: shared hit=X read=Y` 显示块读写 |
| **代价模型** | 仅相对成本 | `cost=X..Y` 显示启动成本和总成本 |

### PostgreSQL EXPLAIN ANALYZE 优化实战

```sql
-- 原始慢查询
EXPLAIN (ANALYZE, BUFFERS, FORMAT TEXT)
SELECT o.id, o.amount, c.name
FROM orders o
JOIN customers c ON o.customer_id = c.id
WHERE o.status = 'pending'
  AND o.created_at > NOW() - INTERVAL '7 days'
ORDER BY o.created_at DESC
LIMIT 100;
```

**输出示例解读：**

```
Limit  (cost=0.56..854.23 rows=100 width=42) (actual time=0.042..12.345 rows=100 loops=1)
  ->  Nested Loop  (cost=0.56..854.23 rows=100 width=42) (actual time=0.039..12.300 rows=100 loops=1)
        Buffers: shared hit=234 read=56
        ->  Index Scan using idx_orders_created_at on orders o
              (cost=0.42..720.10 rows=85 width=24) (actual time=0.020..8.200 rows=100 loops=1)
              Index Cond: (created_at > (now() - '7 days'::interval))
              Filter: ((status)::text = 'pending'::text)
              Buffers: shared hit=180 read=12
        ->  Index Scan using idx_customers_id on customers c
              (cost=0.14..0.16 rows=1 width=18) (actual time=0.003..0.004 rows=1 loops=100)
              Index Cond: (id = o.customer_id)
              Buffers: shared hit=54 read=44
Planning time: 1.234 ms
Execution time: 12.567 ms
```

**关键优化点识别：**

1. **`Buffers: read=56`** 高 → 大量数据从磁盘读取，考虑增加 `shared_buffers` 或优化索引
2. **`Nested Loop` 中 `loops=100`** → 每次外层行都触发一次内层索引扫描，考虑建立**覆盖索引**
3. **`Filter: status='pending'`** → 过滤在 Index Scan 后执行，无法利用索引条件

**优化方案：**

```sql
-- 1. 创建复合覆盖索引，索引列包含 WHERE + ORDER BY + SELECT 所需列
CREATE INDEX idx_orders_cover ON orders (created_at DESC, status)
INCLUDE (id, customer_id, amount);

-- 2. 如果 status 选择性低，可考虑对高频状态单独建索引
-- 3. 调整 work_mem 减少 Sort 时的磁盘交换
SET work_mem = '64MB';
```

---

## 📚 参考来源

1. **MySQL 索引原理 - B+Tree 详解**  
   https://dev.mysql.com/doc/refman/8.0/en/innodb-storage-engine.html

2. **MySQL EXPLAIN 执行计划分析**  
   https://dev.mysql.com/doc/refman/8.0/en/explain.html

3. **PostgreSQL MVCC 机制详解**  
   https://www.postgresql.org/docs/current/mvcc.html

4. **Redis 缓存三大问题详解**  
   https://redis.io/docs/manual/patterns/#rate-limiting

5. **PostgreSQL EXPLAIN ANALYZE 实战**  
   https://www.postgresql.org/docs/current/using-explain.html

---

*本文件由 AI 自动生成并推送至 GitHub 仓库*
