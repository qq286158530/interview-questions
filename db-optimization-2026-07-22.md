# 数据库性能优化面试题

> 📅 更新时间：2026-07-22
> 📚 涵盖范围：MySQL / PostgreSQL 索引优化、查询优化、存储引擎、缓存策略

---

## 题目一：MySQL 中索引失效的常见场景有哪些？如何避免？

### 参考答案

索引失效是面试中的高频考点，以下是常见的导致索引失效的场景：

**1. 使用函数或运算**
```sql
-- 索引失效 ❌
SELECT * FROM orders WHERE YEAR(create_time) = 2026;

-- 正确写法 ✅
SELECT * FROM orders WHERE create_time >= '2026-01-01' AND create_time < '2027-01-01';
```

**2. 类型转换**
```sql
-- phone 是 varchar 类型，传入数字会导致隐式类型转换，索引失效 ❌
SELECT * FROM users WHERE phone = 13800138000;

-- 正确写法 ✅
SELECT * FROM users WHERE phone = '13800138000';
```

**3. 前缀模糊查询**
```sql
-- 最左前缀原则：LIKE 开头使用通配符会导致索引失效 ❌
SELECT * FROM products WHERE name LIKE '%手机%';

-- 正常索引可用 ✅
SELECT * FROM products WHERE name LIKE '华为%';
```

**4. 联合索引未遵循最左匹配原则**
```sql
-- 假设 index(a, b, c) 联合索引
-- 失效情况 ❌
SELECT * FROM t WHERE b = 1;

-- 有效情况 ✅
SELECT * FROM t WHERE a = 1;
SELECT * FROM t WHERE a = 1 AND b = 2;
```

**5. WHERE 条件中使用 NOT、!=、<>**
```sql
-- 索引失效 ❌
SELECT * FROM users WHERE status != 1;
```

**6. 索引列参与计算**
```sql
-- 索引失效 ❌
SELECT * FROM orders WHERE price + 10 > 100;

-- 正确写法 ✅
SELECT * FROM orders WHERE price > 90;
```

**7. OR 连接条件中包含非索引列**
```sql
-- 索引失效 ❌
SELECT * FROM users WHERE name = '张三' OR age = 25;

-- 正确做法：为 age 也建立索引，或拆分为两条查询 ✅
```

**避免策略：**
- 使用 EXPLAIN 分析查询计划
- 尽量使用覆盖索引（Covering Index）
- 避免在 WHERE 条件中对索引列做运算
- 合理设计联合索引，遵循最左匹配原则

---

## 题目二：Explain 执行计划中各个字段的含义是什么？如何根据它优化SQL？

### 参考答案

`EXPLAIN` 是分析 SQL 性能的核心工具，返回字段含义如下：

| 字段 | 含义 |
|------|------|
| `id` | 查询中 SELECT 的序列号，id 越大越先执行 |
| `select_type` | 查询类型（SIMPLE、PRIMARY、SUBQUERY、DERIVED 等） |
| `table` | 操作的表名 |
| `type` | **连接类型**，从优到差：system > const > eq_ref > ref > range > index > ALL |
| `possible_keys` | 可能使用的索引 |
| `key` | 实际使用的索引 |
| `key_len` | 索引使用的字节数，越小越好 |
| `ref` | 与索引比较的列 |
| `rows` | 预估需要扫描的行数，越少越好 |
| `Extra` | 额外信息（Using index、Using filesort、Using temporary 等） |

### 关键优化点

**1. type 字段优化**
- `ALL`（全表扫描）：最差，需要优化
- `range`：范围扫描，可接受
- `ref` / `eq_ref`：索引查找，良好
- `const` / `system`：常量查找，最优

```sql
-- 优化 ALL（慢查询）
EXPLAIN SELECT * FROM orders WHERE status = 1;
-- 优化方案：status 上建立索引
ALTER TABLE orders ADD INDEX idx_status(status);
```

**2. Extra 字段警惕值**
- `Using filesort`：文件排序，效率低，需要优化
  ```sql
  -- 产生原因：ORDER BY 字段无索引或排序规则不一致
  -- 优化：为 ORDER BY 字段建立索引
  ALTER TABLE orders ADD INDEX idx_create_time(create_time);
  ```
- `Using temporary`：使用临时表，需要优化
  ```sql
  -- 产生原因：DISTINCT、GROUP BY、UNION 等操作
  -- 优化：合理建立索引，减少临时表使用
  ```
- `Using index`：覆盖索引，性能好

**3. key_len 计算示例**
```
若联合索引 idx(name, age, phone)
name varchar(20)  utf8mb4 -> 20*4=80+2(长度)=82
age int           -> 4
key_len = 86 表示使用了前两个字段
```

---

## 题目三：MySQL InnoDB 与 MyISAM 存储引擎的区别？如何选择？

### 参考答案

| 特性 | InnoDB | MyISAM |
|------|--------|--------|
| 事务支持 | ✅ 支持 ACID 事务 | ❌ 不支持 |
| 行锁 | ✅ 行级锁，高并发 | ❌ 表级锁 |
| 外键约束 | ✅ 支持 | ❌ 不支持 |
| 崩溃恢复 | ✅ 支持自动恢复（redo log） | ❌ 需手动修复 |
| 全文索引 | ✅ MySQL 5.6+ 支持 | ✅ 原生支持 |
| 索引 | ✅ Clustered Index | ✅ B+Tree |
| 存储空间 | 同等数据量下略大 | 相对较小 |
| 适用场景 | 事务型、高并发、数据可靠性要求高 | 读多写少、不需要事务、日志型 |

**InnoDB 核心优势：**
```sql
-- InnoDB 支持事务回滚
START TRANSACTION;
UPDATE accounts SET balance = balance - 100 WHERE user_id = 1;
UPDATE accounts SET balance = balance + 100 WHERE user_id = 2;
COMMIT; -- 或 ROLLBACK;
```

**崩溃恢复原理：**
- Write Ahead Logging（WAL）：事务提交前先写 redo log
- 脏页刷新机制：将修改写入 redo log，数据库重启时重做

**选择建议：**
- ✅ 选择 InnoDB：生产环境、订单系统、用户系统、任何需要数据可靠性的场景
- MyISAM 适用：只读报表、日志分析表、全文搜索（已逐渐被 InnoDB FTS 替代）

**查看与切换：**
```sql
-- 查看引擎
SHOW TABLE STATUS FROM db_name WHERE Name = 't1';

-- 切换引擎（表迁移）
ALTER TABLE t1 ENGINE = InnoDB;
```

---

## 题目四：如何设计一个高效的数据库缓存策略？Redis 与 MySQL 如何配合使用？

### 参考答案

### 缓存策略设计原则

**1. 缓存读写模式**

**Cache-Aside（旁路缓存，最常用）：**
```python
# 读操作
def get_user(user_id):
    cache_key = f"user:{user_id}"
    user = redis.get(cache_key)
    if user:
        return json.loads(user)
    user = mysql.query("SELECT * FROM users WHERE id = ?", user_id)
    redis.setex(cache_key, 3600, json.dumps(user))  # TTL 1小时
    return user

# 写操作
def update_user(user_id, data):
    mysql.execute("UPDATE users SET ... WHERE id = ?", user_id)
    redis.delete(f"user:{user_id}")  # 删除缓存，让下次读取重建
```

**2. 缓存过期策略**
- **TTL 设置**：根据数据更新频率设置过期时间（60s - 24h）
- **LFU/LRU**：当 Redis 内存满时，按最少使用/最近使用淘汰
- **主动过期**：数据更新时主动删除缓存（Cache-Aside 模式）

**3. 缓存问题及解决方案**

**缓存穿透（查询不存在的数据）：**
```python
# 解决方案：布隆过滤器 or 缓存空值
def get_product(product_id):
    if not bloom_filter.exists(product_id):
        return None  # 直接返回
    # ... 正常查询逻辑
```
```python
# 缓存空值（短期 TTL）
if product is None:
    redis.setex(f"product:{product_id}", 60, "NULL")  # 空值缓存60秒
```

**缓存击穿（热点 key 过期，瞬间大量请求到 DB）：**
```python
# 解决方案：互斥锁 + 永不过期
def get_user_with_lock(user_id):
    cache_key = f"user:{user_id}"
    # 尝试获取锁
    lock = redis.lock(f"lock:{cache_key}", timeout=10)
    if lock.acquire():
        try:
            # 双检锁
            user = redis.get(cache_key)
            if not user:
                user = mysql.query(...)
                redis.setex(cache_key, 3600, json.dumps(user))
            return user
        finally:
            lock.release()
    else:
        time.sleep(0.1)
        return redis.get(cache_key)
```

**缓存雪崩（大量 key 同时过期）：**
```python
# 解决方案：过期时间加随机值
ttl = 3600 + random.randint(0, 300)  # 1小时 + 0~5分钟随机
redis.setex(cache_key, ttl, value)
```

**4. 多级缓存架构**
```
浏览器缓存 → CDN → Nginx 缓存 → 应用本地缓存（Caffeine/Guava） → Redis → MySQL
```

**5. 缓存与数据库一致性**
```python
# 延迟双删（解决读写并发问题）
def update_user(user_id, data):
    mysql.execute("UPDATE users SET ...", user_id)
    redis.delete(f"user:{user_id}")  # 先删除缓存
    time.sleep(0.1)  # 等待读请求完成
    redis.delete(f"user:{user_id}")  # 再次删除
```

---

## 题目五：PostgreSQL 与 MySQL 在性能优化方面有哪些区别？各自有什么独特优化手段？

### 参考答案

### 核心区别

| 维度 | MySQL | PostgreSQL |
|------|-------|------------|
| 架构 | 单进程多线程 | 多进程架构（fork per connection） |
| 存储引擎 | 插件式（InnoDB/MyISAM） | 单引擎（堆表 + MVCC） |
| 索引类型 | B+Tree、Hash、全文、R-Tree | B-Tree、Hash、GiST、SP-GiST、GIN、BRIN |
| 并发控制 | MVCC（InnoDB）| MVCC（原生 MVCC，支持 Snapshot Isolation） |
| 查询优化器 | 规则+代价（较简单）| 基于代价的优化器（更智能）|
| 分区表 | 支持（有限）| 支持（原生 Partitioning）|

### PostgreSQL 独特优化手段

**1. EXPLAIN ANALYZE（执行计划分析）**
```sql
-- PostgreSQL 支持执行并分析
EXPLAIN (ANALYZE, BUFFERS, FORMAT TEXT)
SELECT * FROM orders WHERE status = 1;
-- BUFFERS 显示缓存命中情况
-- ANALYZE 实际执行并返回真实时间
```

**2. 索引类型选择**
```sql
-- 全文搜索
CREATE INDEX idx_content_gin ON articles USING GIN(to_tsvector('english', content));

-- 范围查询 / 物联网时序数据
CREATE INDEX idx_created_brin ON logs USING BRIN(created_at);

-- 多维搜索（地理坐标等）
CREATE INDEX idx_location_gist ON locations USING GIST(geom);
```

**3. 表达式索引**
```sql
-- 针对函数查询建立索引
CREATE INDEX idx_lower_email ON users(LOWER(email));

-- 查询时直接命中索引
SELECT * FROM users WHERE LOWER(email) = 'john@example.com';
```

**4. 部分索引（Partial Index）**
```sql
-- 只为活跃用户建立索引，减小索引体积
CREATE INDEX idx_active_users ON users(last_login) WHERE status = 'active';

-- 查询自动使用
SELECT * FROM users WHERE status = 'active' AND last_login > '2026-01-01';
```

**5. 覆盖索引（Index Only Scan）**
```sql
-- PostgreSQL 支持 Index Only Scan，减少回表
CREATE INDEX idx_user_cover ON users(id) INCLUDE (name, email, phone);
-- 索引包含所有查询字段，可直接返回
```

**6. 连接类型优化**
```sql
-- PostgreSQL 支持多种连接方式
-- Nested Loop：适合小表 join
-- Hash Join：适合大表等值连接
-- Merge Join：适合已排序数据

-- 查看实际使用的连接方式
EXPLAIN SELECT * FROM orders o JOIN users u ON o.user_id = u.id;
```

### MySQL 独特优化手段

**1. 覆盖索引优化**
```sql
-- 索引覆盖所有查询字段
CREATE INDEX idx_user_cover ON users(name, email, phone);
SELECT name, email FROM users WHERE name = '张三';
```

**2. 联合索引最左前缀**
```sql
-- 合理设计联合索引
CREATE INDEX idx_order ON orders(user_id, status, create_time);
-- 查询 1: WHERE user_id = 1 → 使用索引
-- 查询 2: WHERE user_id = 1 AND status = 1 → 使用索引
-- 查询 3: WHERE user_id = 1 AND status = 1 AND create_time > '2026-01-01' → 使用全部索引
```

**3. MRR（Multi-Range Read）优化**
```sql
-- 开启 MRR，减少随机 IO
SET optimizer_switch = 'mrr=on,mrr_cost_based=on';
```

**4. ICP（Index Condition Pushdown）优化**
```sql
-- 下推索引条件到存储引擎层
SET optimizer_switch = 'index_condition_pushdown=on';
-- 对联合索引中间字段的查询可直接在索引层过滤
SELECT * FROM orders WHERE user_id = 1 AND status > 5;
-- status > 5 可以在索引层直接过滤
```

### 共同优化最佳实践

```sql
-- 1. 定期分析表统计信息
MySQL: ANALYZE TABLE orders;
PostgreSQL: ANALYZE orders;

-- 2. 查看慢查询日志
MySQL: slow_query_log = 1; long_query_time = 1;
PostgreSQL: log_min_duration_statement = 1000;  -- 1秒

-- 3. 连接池配置
MySQL: max_connections 根据内存调优，InnoDB buffer pool size
PostgreSQL: max_connections + shared_buffers + effective_cache_size

-- 4. 读写分离
MySQL: 主从复制 + 读写分离中间件（Atlas/ProxySQL）
PostgreSQL: 主从流复制 + pgpool-II / Supabase
```

---

> 📌 **面试小贴士**：数据库优化要结合实际场景，面试时可以提到"先定位瓶颈（Explain/Analyze），再针对性优化（索引→SQL→架构）"的思路。
