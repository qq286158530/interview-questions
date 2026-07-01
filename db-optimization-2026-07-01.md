# 数据库性能优化面试题

> 📅 日期：2026-07-01
> 🏷️ 标签：MySQL | PostgreSQL | 数据库优化

---

## 题目一：MySQL索引优化 — 为什么需要遵循最左前缀原则？

### 题目
在MySQL中，当创建一个联合索引 `(a, b, c)` 时，为什么查询 `WHERE b = 1` 无法使用该索引？什么情况下会导致索引失效？

### 答案

**最左前缀原则（Leftmost Prefix Principle）** 是联合索引的核心规则。

#### 联合索引的存储结构

联合索引按照索引列的顺序依次排序。以 `(name, age, city)` 为例：

```
B+Tree 结构：
        [name=Tom, age=20, city=Beijing]
       /                                \
[name=Tom, age=21, city=Shanghai]  [name=Lucy, age=22, city=Guangzhou]
```

数据首先按 `name` 排序，name 相同时按 `age` 排序，age 相同时按 `city` 排序。

#### 为什么 `WHERE b = 1` 无法使用索引？

索引 `(a, b, c)` 的排列顺序是 **a → b → c**。查询 `WHERE b = 1` 跳过了索引的最左列 `a`，MySQL 无法从 B+Tree 的根节点开始有序查找，因为：

- 没有 `a` 的值，数据在 `a` 维度上是无序的
- MySQL 只能进行全表扫描或索引扫描（index scan），但无法利用 B+Tree 的有序性进行快速定位

#### 会导致索引失效的常见场景

| 场景 | 示例 | 是否使用索引 |
|------|------|-------------|
| 跳过最左列 | `WHERE b = 1` | ❌ |
| 使用函数/运算 | `WHERE YEAR(created_at) = 2026` | ❌ |
| 类型转换 | 索引列类型为 INT，查询用字符串 `WHERE id = '1'` | ❌ |
| LIKE 开头是通配符 | `WHERE name LIKE '%Tom'` | ❌ |
| OR 连接不同列 | `WHERE a = 1 OR c = 3`（无覆盖索引）| ❌ |
| 使用 NOT/NOT IN | `WHERE status NOT IN (1,2)` | ❌ |
| 数据量过小 | 全表 100 行，索引选择率 > 20% | ❌ |

#### 正确做法

```sql
-- ✅ 使用完整最左前缀
WHERE a = 1 AND b = 2 AND c = 3
WHERE a = 1 AND b = 2

-- ✅ 优化器可能使用索引扫描（不是最优）
WHERE a = 1
```

---

## 题目二：如何定位和优化慢查询？

### 题目
生产环境中发现某条SQL查询缓慢（执行时间 > 3秒），请描述完整的排查和优化思路。

### 答案

#### 第一步：定位慢查询

```sql
-- 开启慢查询日志
SET GLOBAL slow_query_log = 'ON';
SET GLOBAL long_query_time = 1;  -- 超过1秒记录
SET GLOBAL slow_query_log_file = '/var/log/mysql/slow.log';

-- 查看当前会话
SHOW VARIABLES LIKE 'slow_query%';

-- 分析慢查询日志（使用 mysqldumpslow）
mysqldumpslow -t 5 /var/log/mysql/slow.log
```

#### 第二步：使用 EXPLAIN 分析执行计划

```sql
EXPLAIN SELECT * FROM orders WHERE user_id = 100 AND status = 'paid';

-- 关键字段解读：
-- type: 访问类型 (const > eq_ref > ref > range > index > ALL)
-- key: 实际使用的索引
-- rows: 预计扫描行数
-- Extra: 额外信息 (Using filesort, Using temporary, Using index)
```

#### 第三步：常见优化手段

**1. 优化索引**
```sql
-- 添加合适的索引
CREATE INDEX idx_user_status ON orders(user_id, status);
```

**2. 优化 SQL 语句**
```sql
-- ❌ 避免 SELECT *
SELECT id, name, email FROM users WHERE id = 1;

-- ❌ 避免在 WHERE 中使用函数
-- ❌ 避免隐式类型转换
```

**3. 减少扫描数据量**
```sql
-- ✅ 使用 LIMIT 限制返回
-- ✅ 分页优化：使用延迟关联
SELECT * FROM orders
INNER JOIN (
    SELECT id FROM orders WHERE user_id = 100 LIMIT 10000, 20
) AS t USING(id);
```

**4. 避免文件排序和临时表**
```sql
-- Extra 列出现 Using filesort / Using temporary 时需优化
-- 解决方式：创建合适的索引覆盖所需列
```

**5. 查看索引使用情况**
```sql
SHOW STATUS LIKE 'Handler_read%';
-- Handler_read_rnd_next 高 = 全表扫描多
```

---

## 题目三：MySQL InnoDB 与 MyISAM 的核心区别？

### 题目
在选择存储引擎时，InnoDB 和 MyISAM 各有什么特点？什么场景下选择 MyISAM 更合适？

### 答案

| 特性 | InnoDB | MyISAM |
|------|--------|--------|
| 事务支持 | ✅ 支持 ACID 事务 | ❌ 不支持 |
| 行锁/表锁 | 行锁（高并发）| 表锁 |
| 外键约束 | ✅ 支持 | ❌ 不支持 |
| 全文索引 | ✅ MySQL 5.6+ 支持 | ✅ 原生支持 |
| 索引结构 | B+Tree + 聚簇索引 | B+Tree（非聚簇）|
| 数据存储 | 表空间 | .myd 文件 |
| 崩溃恢复 | 自动恢复（redo log）| 慢（需要修复）|
| 计数统计 | 全表扫描（count）| 保存到元数据 |
| 并发能力 | 高 | 低 |

#### InnoDB 的优势（默认选择）

```sql
-- 1. 事务安全
START TRANSACTION;
UPDATE accounts SET balance = balance - 100 WHERE user_id = 1;
UPDATE accounts SET balance = balance + 100 WHERE user_id = 2;
COMMIT;  -- 失败自动回滚
```

```sql
-- 2. 聚簇索引（数据与索引同结构，减少IO）
-- 主键索引叶子节点直接存储行数据
-- 适合范围查询和主键查找
```

#### MyISAM 适用场景

- **读多写少**且**不需要事务**的场景（如日志系统、报表系统）
- 需要**全文索引**且数据量不大（MySQL 5.5 及之前版本）
- 追求**极致的 SELECT COUNT(\*)** 性能（MyISAM 将计数存在元数据中）

```sql
-- MyISAM 计数优势示例
SELECT COUNT(*) FROM large_table;  -- MyISAM 直接读元数据
```

#### 现代推荐

> **除非有特殊原因，强烈建议使用 InnoDB**。MySQL 8.0+ 已将 InnoDB 作为默认存储引擎，MyISAM 已被标记为废弃（deprecated）。

---

## 题目四：PostgreSQL 查询优化器如何选择执行计划？

### 题目
PostgreSQL 的查询优化器是如何工作的？为什么同样的 SQL 在不同数据量下执行计划不同？

### 答案

#### PostgreSQL 查询优化器原理

PostgreSQL 使用**基于代价的优化器（CBO - Cost-Based Optimizer）**，而非规则优化器。

```
SQL 语句
    ↓
Parser（解析）→ AST
    ↓
Rewriter（重写）→ 逻辑计划
    ↓
Planner（优化器）→ 物理执行计划 ← 【核心环节】
    ↓
Executor（执行器）
```

#### 代价模型（Cost Model）

优化器会估算不同执行计划的「代价」，选择代价最低的：

```sql
-- 开启执行计划显示代价估算
EXPLAIN (ANALYZE, COSTS, BUFFERS) 
SELECT * FROM users WHERE age > 25;
```

**主要代价因素：**

| 因素 | 说明 |
|------|------|
| 页面读取代价 | 顺序读 vs 随机读（可通过 `seq_page_cost` / `random_page_cost` 配置）|
| 运算符代价 | 排序、聚合、连接等 CPU 开销 |
| 行数估算 | 基于统计信息（statistics）|

#### 为什么不同数据量执行计划不同？

```sql
-- 数据量小（100行）→ 全表扫描更快
-- 数据量大（1000万行）→ 索引扫描更快

-- 原因：优化器根据统计信息估算行数
SELECT reltuples, relpages FROM pg_class WHERE relname = 'users';
-- reltuples: 估算行数
-- relpages: 估算页数
```

#### 统计信息收集

```sql
-- 手动更新统计信息
ANALYZE users;

-- 配置自动统计收集
ALTER TABLE users SET (autovacuum_analyze_threshold = 50);
```

#### 强制使用特定索引（调试用）

```sql
-- 临时强制使用索引
SET enable_seqscan = off;  -- 禁用顺序扫描
SET enable_hashjoin = off; -- 禁用哈希连接

-- 查看更详细的计划
EXPLAIN (ANALYZE, VERBOSE, BUFFERS, FORMAT JSON)
SELECT * FROM users u JOIN orders o ON u.id = o.user_id;
```

---

## 题目五：数据库缓存策略 — 如何设计 Redis + MySQL 的缓存模式？

### 题目
在高并发系统中，如何设计 Redis 缓存来减轻 MySQL 的压力？缓存穿透、缓存击穿、缓存雪崩如何应对？

### 答案

#### 经典缓存模式：Cache-Aside

```
读操作：
  应用 → Redis（命中？→返回）
           ↓ 未命中
         → MySQL（查询 → 写入Redis → 返回）

写操作：
  应用 → MySQL（写入）
           ↓
         → Redis（删除/更新缓存）
```

```python
# Python 示例
def get_user(user_id):
    cache_key = f"user:{user_id}"
    
    # 先查缓存
    user = redis.get(cache_key)
    if user:
        return json.loads(user)
    
    # 缓存未命中，查数据库
    user = db.query("SELECT * FROM users WHERE id = %s", user_id)
    
    # 写入缓存（设置过期时间）
    redis.setex(cache_key, 3600, json.dumps(user))
    
    return user

def update_user(user_id, data):
    # 先更新数据库
    db.execute("UPDATE users SET ... WHERE id = %s", user_id)
    
    # 删除缓存（而非更新，避免并发问题）
    redis.delete(f"user:{user_id}")
```

#### 三大缓存问题及解决方案

**1. 缓存穿透（查询不存在的数据）**

问题：恶意或错误请求查询不存在的数据，每次都穿透到 MySQL。

```
解决方案：
  a. 布隆过滤器（Bloom Filter）
  b. 缓存空值（短过期时间）
```

```python
# 布隆过滤器方案
def get_product(product_id):
    if bloom_filter.might_contain(product_id):
        # 可能存在，查缓存和数据库
        return get_from_db_or_cache(product_id)
    else:
        # 一定不存在，直接返回
        return None

# 缓存空值方案
result = get_from_db(product_id)
if result is None:
    redis.setex(f"product:{product_id}", 60, "NULL")  # 短过期时间
```

**2. 缓存击穿（热点key过期，瞬间大量请求击穿到DB）**

问题：某个热点 key 过期瞬间，大量并发请求同时穿透到数据库。

```
解决方案：
  a. 互斥锁（Mutex）
  b. 永不过期（定期更新）
  c. 逻辑过期（不设置TTL，存过期时间戳）
```

```python
import threading

# 互斥锁方案
lock = redis.lock(f"lock:product:{product_id}", timeout=10)
if lock.acquire():
    try:
        # 双检锁：获取锁后再次确认缓存
        result = redis.get(f"product:{product_id}")
        if not result:
            result = db.query(...)
            redis.setex(f"product:{product_id}", 3600, result)
    finally:
        lock.release()
else:
    # 未获取到锁，短暂等待后重试
    time.sleep(0.1)
    return redis.get(f"product:{product_id}")
```

**3. 缓存雪崩（大量 key 同时过期）**

问题：大量缓存 key 设置了相同的过期时间，同时过期导致数据库压力骤增。

```
解决方案：
  a. 过期时间加随机值
  b. 永不过期 + 后台更新
  c. 多级缓存架构
```

```python
# 过期时间加随机值
TTL = 3600 + random.randint(0, 300)  # 3600-3900秒

# 后台定时更新（永不过期）
def refresh_cache():
    while True:
        keys = redis.keys("product:*")
        for key in keys:
            data = db.query(...)
            redis.set(key, json.dumps(data))  # 不设过期时间
        time.sleep(300)  # 每5分钟刷新
```

#### 缓存更新策略对比

| 策略 | 一致性 | 实现复杂度 | 适用场景 |
|------|--------|----------|---------|
| Cache-Aside | 高 | 中 | 读多写少 |
| Write-Through | 高 | 高 | 数据频繁更新 |
| Write-Behind | 中 | 高 | 异步写入，高吞吐 |
| Read-Through | 中 | 中 | 简化应用逻辑 |

---

## 📚 参考资料

1. **MySQL 官方文档 - 索引**  
   https://dev.mysql.com/doc/refman/8.0/en/mysql-indexes.html

2. **PostgreSQL 官方文档 - 使用 EXPLAIN**  
   https://www.postgresql.org/docs/current/using-explain.html

3. **MySQL 慢查询日志配置与使用**  
   https://dev.mysql.com/doc/refman/8.0/en/slow-query-log.html

4. **Redis 设计与实现 — Redis 缓存问题**  
   https://redis.io/docs/manual/patterns/

5. **InnoDB 存储引擎架构**  
   https://dev.mysql.com/doc/refman/8.0/en/innodb-architecture.html

---

> 📌 本题库由 `interview-questions` 仓库自动生成  
> GitHub: https://github.com/qq286158530/interview-questions
