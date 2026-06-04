# 数据库性能优化面试题

> 来源：综合整理自网络经典面试题 + 技术社区高赞讨论
> 日期：2026-06-04

---

## 题目 1：MySQL 中索引失效的常见场景有哪些？

### 参考答案

索引失效的常见场景包括：

1. **使用 `LIKE` 以通配符开头**
   ```sql
   SELECT * FROM users WHERE name LIKE '%wang';
   ```
   `'%wang'` 会导致全表扫描，因为前缀通配符阻止了 B+ 树索引的有序性。

2. **使用函数或运算**
   ```sql
   SELECT * FROM orders WHERE YEAR(create_time) = 2026;
   SELECT * FROM products WHERE price + 10 > 100;
   ```
   对索引列使用函数、运算（`+ - * /`）会导致索引失效。

3. **类型转换**
   ```sql
   SELECT * FROM users WHERE phone = 13800138000;  -- phone 是 varchar
   ```
   隐式类型转换导致索引失效。

4. **使用 `OR` 连接多个条件**
   ```sql
   SELECT * FROM users WHERE name = 'Tom' OR age = 20;
   ```
   若 `age` 列无索引，MySQL 会放弃使用索引做全表扫描。

5. **不符合最左前缀原则**
   ```sql
   -- 索引为 (name, age, city)
   SELECT * FROM users WHERE age = 20;        -- 失效
   SELECT * FROM users WHERE city = 'Beijing'; -- 失效
   ```

6. **使用 `NOT IN` / `!=`**
   部分场景下优化器选择不走索引。

7. **数据量过小时**，优化器认为全表扫描更快。

---

## 题目 2：如何定位并优化慢查询？

### 参考答案

### Step 1：开启慢查询日志
```ini
# my.cnf
slow_query_log = 1
slow_query_log_file = /var/log/mysql/slow.log
long_query_time = 1
```

### Step 2：分析慢查询日志
使用 `mysqldumpslow` 汇总：
```bash
mysqldumpslow -t 5 /var/log/mysql/slow.log
```

### Step 3：`EXPLAIN` 分析执行计划
```sql
EXPLAIN SELECT ...
EXPLAIN ANALYZE ...  -- MySQL 8.0+
```
重点关注：
- `type`：最好达到 `ref/range`，避免 `ALL`（全表扫描）
- `Extra`：`Using filesort`、`Using temporary` 是警告信号
- `rows`：扫描行数是否合理

### Step 4：针对性优化
| 问题 | 解决方案 |
|------|---------|
| `Using filesort` | 创建合适索引，消除排序 |
| `Using temporary` | 优化 `GROUP BY`、`ORDER BY` |
| 全表扫描 | 增建索引或拆表 |
| `Using index condition` | 考虑覆盖索引 |

### Step 5：使用 `SHOW PROFILE`
```sql
SET profiling = 1;
SELECT ...;
SHOW PROFILE;
```

---

## 题目 3：InnoDB 与 MyISAM 的核心区别是什么？如何选择？

### 参考答案

| 特性 | InnoDB | MyISAM |
|------|--------|--------|
| **事务支持** | ✅ 支持 ACID 事务 | ❌ 不支持 |
| **行锁** | ✅ 支持行锁，并发高 | ❌ 仅表锁 |
| **外键** | ✅ 支持 | ❌ 不支持 |
| **崩溃恢复** | ✅ 支持 crash-safe | ❌ 需手动修复 |
| **全文索引** | ✅ 5.6+ 支持 | ✅ 原生支持 |
| **count(*) 性能** | 较慢（需扫描） | 快（维护了计数器） |
| **存储结构** | `.ibd`（独立表空间） | `.MYD` + `.MYI` |

### 选择建议

- **InnoDB**：生产环境首选，处理高并发、有事务需求、数据可靠性要求高的场景
- **MyISAM**：适用于只读/低并发场景（如报表、日志表）、需要全文检索的老系统

> MySQL 8.0+ 默认存储引擎已是 InnoDB。

---

## 题目 4：PostgreSQL 中如何优化大表的全表扫描与分页查询？

### 参考答案

### 大表全表扫描优化

1. **评估是否需要索引**
   ```sql
   -- 分析查询模式后创建索引
   CREATE INDEX CONCURRENTLY idx_orders_created ON orders(create_time);
   ```

2. **使用 `EXPLAIN (ANALYZE, BUFFERS)` 分析**
   ```sql
   EXPLAIN (ANALYZE, BUFFERS, FORMAT TEXT) SELECT * FROM orders;
   ```

3. **并行扫描**
   ```sql
   SET max_parallel_workers_per_gather = 4;
   -- PostgreSQL 自动使用并行扫描大表
   ```

4. **分区表（Table Partitioning）**
   ```sql
   CREATE TABLE orders (
       id bigserial,
       created_at date
   ) PARTITION BY RANGE (created_at);

   CREATE TABLE orders_2026 PARTITION OF orders
       FOR VALUES FROM ('2026-01-01') TO ('2027-01-01');
   ```
   查询时自动裁剪分区，大幅减少扫描量。

### 分页查询深翻页优化

**传统 OFFSET 的问题：**
```sql
SELECT * FROM orders ORDER BY id LIMIT 1000000 OFFSET 100000;
-- 偏移量越大越慢，需要扫描并丢弃大量数据
```

**优化方案 1：游标分页（Keyset Pagination）**
```sql
-- 第1页
SELECT * FROM orders ORDER BY id DESC LIMIT 20;
-- 下一页，用上一页最后一条的 id 作为起点
SELECT * FROM orders WHERE id < :last_id ORDER BY id DESC LIMIT 20;
```

**优化方案 2：使用表达式索引 + 过滤**
```sql
CREATE INDEX idx_orders_id ON orders (id DESC);
-- 利用索引顺序，避免大 offset
```

---

## 题目 5：数据库缓存策略有哪些？如何设计多级缓存架构？

### 参考答案

### 常见缓存策略

| 策略 | 描述 | 适用场景 |
|------|------|---------|
| **Cache-Aside** | 应用先查缓存，miss 时查 DB 并写缓存 | 读多写少 |
| **Write-Through** | 写操作同时更新缓存和 DB | 数据一致性要求高 |
| **Write-Behind** | 写操作先更新缓存，异步批量写 DB | 写性能要求高 |
| **Refresh-Ahead** | 预刷新即将过期的缓存 | 热数据固定场景 |

### 多级缓存架构设计

```
用户请求
   ↓
┌─────────────────┐
│  浏览器缓存      │  (HTTP Cache-Control / ETag)
└─────────────────┘
   ↓ miss
┌─────────────────┐
│  CDN 缓存        │  (静态资源)
└─────────────────┘
   ↓ miss
┌─────────────────┐
│  Redis 缓存       │  (热点数据，TTL 短)
└─────────────────┘
   ↓ miss
┌─────────────────┐
│  本地缓存        │  (Guava Cache / Caffeine)
└─────────────────┘
   ↓ miss
┌─────────────────┐
│  MySQL/PG DB     │
└─────────────────┘
```

### Redis 缓存实战要点

**1. 缓存穿透（请求绕过缓存直接打爆 DB）**
- 布隆过滤器（Bloom Filter）拦截不存在的数据
- 缓存空值（短期 TTL）

**2. 缓存击穿（热点 key 失效瞬间大量请求）**
- 互斥锁（Mutex）保证只有一个线程回源
- 永不过期 + 异步更新

**3. 缓存雪崩（大量 key 同时过期）**
- 随机 TTL，避免同时失效
- 高可用架构（Redis Cluster）

**4. 缓存与 DB 一致性**
```sql
-- 延迟双删策略
DELETE FROM cache WHERE key = 'product:123';
UPDATE products SET stock = stock - 1 WHERE id = 123;
-- 短暂延迟后再次删除（异步删除脏数据）
```
推荐最终一致性强于实时一致性，使用消息队列解耦。

---

> 📚 更多面试题持续更新中，欢迎 Star ⭐：[github.com/qq286158530/interview-questions](https://github.com/qq286158530/interview-questions)