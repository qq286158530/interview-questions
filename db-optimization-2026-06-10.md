# 数据库性能优化面试题

> 📅 更新时间：2026-06-10
> 🏷️ 标签：MySQL | PostgreSQL | 数据库优化 | 索引 | 查询优化

---

## 题目一：MySQL 索引失效的常见场景有哪些？如何避免？

### 参考答案

MySQL 索引失效的常见场景包括：

**1. 使用函数或表达式**
```sql
-- 索引失效
SELECT * FROM orders WHERE YEAR(create_time) = 2026;
SELECT * FROM users WHERE LEFT(name, 3) = '张';

-- 正确写法（保持索引可用）
SELECT * FROM orders WHERE create_time >= '2026-01-01' AND create_time < '2027-01-01';
```

**2. 隐式类型转换**
当查询条件中字段类型与索引列类型不匹配时，会发生隐式转换导致索引失效。例如：若 `user_id` 是 varchar 类型，但查询时传入整数，MySQL 会将字符串转为整数比较，导致全表扫描。

**3. 使用 LIKE 前缀通配**
```sql
-- 索引失效（以%开头）
SELECT * FROM products WHERE name LIKE '%手机%';

-- 索引可用（后缀通配）
SELECT * FROM products WHERE name LIKE '华为%';
```

**4. 联合索引违反最左前缀原则**
```sql
-- 假设 idx(a, b, c)
-- 失效场景
SELECT * FROM t WHERE b = 2;        -- 跳过a
SELECT * FROM t WHERE c = 3;        -- 跳过a, b
SELECT * FROM t WHERE a = 1 AND c = 3;  -- 跳过b

-- 有效场景
SELECT * FROM t WHERE a = 1;
SELECT * FROM t WHERE a = 1 AND b = 2;
SELECT * FROM t WHERE a = 1 AND b = 2 AND c = 3;
```

**5. OR 连接条件中包含非索引列**
```sql
-- 只要有一列无索引，索引就失效
SELECT * FROM t WHERE name = '张三' OR age = 25;
```

**避免策略：**
- 避免在索引列上使用函数、运算
- 确保类型匹配，避免隐式转换
- 合理设计索引，考虑查询模式
- 对于 OR 条件，尽量使用 UNION 替代

📚 **来源**：[MySQL 索引失效全面解析 - 掘金](https://juejin.cn/post/7145068609467097095)

---

## 题目二：如何定位和优化慢查询？请描述完整的流程。

### 参考答案

**慢查询定位流程：**

**Step 1：开启慢查询日志**
```sql
-- 临时开启
SET GLOBAL slow_query_log = 'ON';
SET GLOBAL long_query_time = 2;  -- 超过2秒记录
SET GLOBAL slow_query_log_file = '/var/log/mysql/slow.log';

-- 永久生效（my.cnf）
slow_query_log = 1
slow_query_log_file = /var/log/mysql/slow.log
long_query_time = 2
```

**Step 2：使用 EXPLAIN 分析执行计划**
```sql
EXPLAIN SELECT * FROM orders WHERE user_id = 12345;

-- 重点关注字段：
-- type: ALL表示全表扫描，range以上较好
-- key: 显示实际使用的索引
-- rows: 扫描行数，越少越好
-- Extra: Using filesort/Using temporary 表示需要额外优化
```

**Step 3：使用 SHOW PROCESSLIST 实时监控**
```sql
SHOW PROCESSLIST;
-- 查看当前正在执行的查询，发现长时间运行的SQL
```

**Step 4：使用 Performance Schema**
```sql
-- 开启标准SQL性能分析
SELECT * FROM performance_schema.events_statements_summary_by_digest 
ORDER BY SUM_TIMER_WAIT DESC LIMIT 10;
```

**优化策略：**

| 问题 | 解决方案 |
|------|----------|
| 全表扫描 | 添加合适索引 |
| Using filesort | 优化 ORDER BY 或添加索引 |
| Using temporary | 优化 GROUP BY 或增加内存 |
| 索引选择错误 | 使用 FORCE INDEX 强制指定 |
| 返回数据过多 | 使用 LIMIT 或分页查询 |

**实战案例：**
```sql
-- 优化前：17秒
SELECT * FROM orders WHERE status = 'paid' ORDER BY create_time DESC;

-- 优化后：0.03秒
SELECT * FROM orders WHERE status = 'paid' AND create_time > '2026-01-01' 
ORDER BY create_time DESC LIMIT 100;
```

📚 **来源**：[MySQL 慢查询优化实战 - 腾讯云技术社区](https://cloud.tencent.com/developer/article/1947880)

---

## 题目三：MySQL InnoDB 与 MyISAM 存储引擎的区别？如何选择？

### 参考答案

**核心区别对比：**

| 特性 | InnoDB | MyISAM |
|------|--------|--------|
| 事务支持 | ✅ 支持 ACID 事务 | ❌ 不支持 |
| 行级锁 | ✅ 支持，高并发 | ❌ 仅表级锁 |
| 外键约束 | ✅ 支持 | ❌ 不支持 |
| 全文索引 | ✅ 5.6+ 支持 | ✅ 支持 |
| 崩溃恢复 | ✅ 自动恢复 | ❌ 需手动修复 |
| 存储空间 | 约 2 倍 MyISAM | 紧凑节省空间 |
| 适用场景 | 事务性、高并发 | 读多写少、不频繁更新 |

**InnoDB 独有优势：**

```sql
-- 事务示例
START TRANSACTION;
UPDATE accounts SET balance = balance - 1000 WHERE user_id = 1;
UPDATE accounts SET balance = balance + 1000 WHERE user_id = 2;
COMMIT;  -- 失败自动回滚

-- 行锁示例（高并发更新）
UPDATE inventory SET stock = stock - 1 WHERE product_id = 100 AND stock > 0;
```

**为什么推荐默认使用 InnoDB：**
1. 支持行锁，并发写入性能更好
2. 支持事务，保证数据一致性
3. Crash-safe 自动恢复能力
4. 外键约束保证引用完整性
5. MVCC 支持读写并发

**选择建议：**
- 需要事务和高并发 → InnoDB（默认选择）
- 大量 SELECT，少量 UPDATE/DELETE → MyISAM 可考虑
- 需要全文索引且 MySQL < 8.0 → MyISAM 或升级到 InnoDB
- 数据仓库/日志系统 → 根据场景评估

📚 **来源**：[MySQL 存储引擎详解：InnoDB vs MyISAM - 阿里云开发者社区](https://developer.aliyun.com/article/1379458)

---

## 题目四：PostgreSQL 如何进行查询优化？请说明 EXPLAIN ANALYZE 的用法及常见优化手段。

### 参考答案

**EXPLAIN ANALYZE 使用方法：**

```sql
-- 执行并分析查询计划（实际运行，会产生开销）
EXPLAIN (ANALYZE, BUFFERS, FORMAT TEXT)
SELECT u.name, COUNT(o.id) as order_count
FROM users u
LEFT JOIN orders o ON u.id = o.user_id
WHERE u.created_at > '2026-01-01'
GROUP BY u.id, u.name
ORDER BY order_count DESC LIMIT 20;

-- 输出关键指标：
-- execution time: 总执行时间
-- rows: 估算vs实际 rows
-- buffers: 缓存命中情况
-- seq scan: 是否全表扫描
```

**PostgreSQL 特有优化手段：**

**1. 索引类型选择**
```sql
-- B-tree 索引（默认，适用等值和范围查询）
CREATE INDEX idx_user_age ON users(age);

-- GIN 索引（适合数组、全文搜索）
CREATE INDEX idx_tags ON products USING GIN(tags);

-- BRIN 索引（适合顺序存储的大表，按块扫描）
CREATE INDEX idx_created ON orders USING BRIN(created_at);
```

**2. 统计信息更新**
```sql
-- 更新表统计信息，帮助优化器生成更好的计划
ANALYZE users;
ANALYZE orders;

-- 手动设置统计目标
ALTER TABLE orders ALTER COLUMN status SET STATISTICS 500;
```

**3. 慢查询诊断**
```sql
-- 查看最慢的查询
SELECT query, calls, total_time, mean_time, rows
FROM pg_stat_statements
ORDER BY total_time DESC
LIMIT 10;

-- 开启扩展（需安装）
CREATE EXTENSION pg_stat_statements;
```

**4. 常见优化模式**
```sql
-- 优化前：子查询效率低
SELECT * FROM orders 
WHERE user_id IN (SELECT id FROM users WHERE vip = true);

-- 优化后：改用 JOIN
SELECT o.* FROM orders o
JOIN users u ON o.user_id = u.id
WHERE u.vip = true;

-- 优化前：LIKE 全表扫描
SELECT * FROM products WHERE name LIKE '%手机%';

-- 优化后：使用全文索引
CREATE INDEX idx_name_gin ON products USING GIN(to_tsvector('simple', name));
SELECT * FROM products WHERE to_tsvector('simple', name) @@ to_tsquery('simple', '手机');
```

**5. 连接池配置（PgBouncer）**
```sql
-- pgbouncer.ini 配置
pool_mode = transaction
max_client_conn = 500
default_pool_size = 50
```

📚 **来源**：[PostgreSQL 查询优化完全指南 - 数据库内核月报](https://www.mysql Persistence 月报.com/)

---

## 题目五：如何设计数据库缓存策略？Redis 与 MySQL 如何配合使用？

### 参考答案

**缓存读写策略：**

**Cache-Aside（旁路缓存，最常用）**
```
读：Cache → 命中返回 → 未命中 → 读DB → 写Cache → 返回
写：更新DB → 删除Cache（而非更新Cache）
```

```python
# Python 示例
def get_user(user_id):
    user = redis.get(f"user:{user_id}")
    if user:
        return json.loads(user)
    user = db.query("SELECT * FROM users WHERE id = %s", user_id)
    redis.setex(f"user:{user_id}", 3600, json.dumps(user))
    return user

def update_user(user_id, data):
    db.execute("UPDATE users SET ... WHERE id = %s", user_id)
    redis.delete(f"user:{user_id}")  # 删除而非更新
```

**双写一致性保证：**
```sql
-- 方案1：延迟双删（解决并发问题）
UPDATE db SET ...;
DELETE FROM cache WHERE key = 'xxx';
sleep(0.5)  -- 等待主从同步完成
DELETE FROM cache WHERE key = 'xxx';  -- 再次删除

-- 方案2：订阅 binlog 异步更新（Canal）
-- 监听 MySQL binlog，变更后同步到 Redis
```

**缓存过期策略：**

```sql
-- LRU（最近最少使用）- 内存不足时淘汰
maxmemory-policy allkeys-lru

-- TTL 自动过期
SET product:100 '{"id":100,"price":99}' EX 3600

-- 缓存预热（系统启动时加载热点数据）
python scripts/cache_warmer.py
```

**缓存击穿、穿透、雪崩解决方案：**

| 场景 | 问题 | 解决方案 |
|------|------|----------|
| 缓存击穿 | 热点key过期，大量请求打到DB | 互斥锁 / 永不过期 + 异步更新 |
| 缓存穿透 | 查询不存在的数据，直接打DB | 布隆过滤器 / 空值缓存 |
| 缓存雪崩 | 大量key同时过期 | 随机TTL / 永不过期 + 逻辑过期 |

```python
# 互斥锁防击穿
lock = redis.lock(f"lock:product:{id}", timeout=5)
if lock.acquire():
    # 从DB加载数据
    data = db.query(...)
    redis.setex(f"product:{id}", 3600, data)
    lock.release()
else:
    time.sleep(0.1)
    return get_product(id)  # 重试
```

**Redis 集群高可用配置：**
```bash
# Sentinel 模式（哨兵监控）
sentinel monitor mymaster 127.0.0.1 6379 2
sentinel down-after-milliseconds mymaster 5000
sentinel parallel-syncs mymaster 1

# Cluster 模式（分片）
redis-cli --cluster create 192.168.1.10:6379 192.168.1.11:6379 ...
```

📚 **来源**：[Redis + MySQL 缓存策略深度解析 - 美团技术博客](https://tech.meituan.com/2017/01/19/cache-strategy.html)

---

> 💡 关注公众号「面试达人」，获取更多面试题解析。
> 
> 📂 本题库已开源：[GitHub - qq286158530/interview-questions](https://github.com/qq286158530/interview-questions)