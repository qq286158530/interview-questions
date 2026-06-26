# 数据库性能优化面试题

> 📅 整理日期：2026-06-26  
> 🏷️ 标签：MySQL | PostgreSQL | 性能优化 | 索引 | 查询优化

---

## 题目 1：MySQL 中索引失效的常见场景有哪些？

### 参考答案

索引失效是面试中的高频考点，主要包括以下场景：

**1. 使用函数或运算**
```sql
-- 失效：WHERE YEAR(create_time) = 2024
SELECT * FROM orders WHERE YEAR(create_time) = 2024;

-- 生效：改写为范围查询
SELECT * FROM orders WHERE create_time >= '2024-01-01' AND create_time < '2025-01-01';
```

**2. 类型转换**
```sql
-- 失效：phone 是 varchar 类型，但传入数字
SELECT * FROM users WHERE phone = 13800138000;

-- 生效：使用字符串
SELECT * FROM users WHERE phone = '13800138000';
```

**3. LIKE 模糊匹配以 % 开头**
```sql
-- 失效：前面有通配符
SELECT * FROM products WHERE name LIKE '%手机';

-- 生效：前缀匹配
SELECT * FROM products WHERE name LIKE '苹果%';
```

**4. 联合索引违反最左前缀原则**
```sql
-- 创建索引 KEY idx(a, b, c)
-- 失效
SELECT * FROM t WHERE b = 1;
-- 失效
SELECT * FROM t WHERE c = 1;
-- 生效
SELECT * FROM t WHERE a = 1 AND c = 1; -- 只用上 a
```

**5. OR 连接条件中有字段不带索引**
```sql
-- 如果 age 没有索引，整个查询都失效
SELECT * FROM users WHERE name = '张三' OR age = 25;
```

**6. 使用 NOT、!=、<> 操作符**
部分情况下导致全表扫描。

**7. ORDER BY 踩坑**
```sql
-- 如果 ORDER BY 的字段不在索引中，或排序方向不一致，无法利用索引
SELECT * FROM t ORDER BY a ASC, b DESC; -- 方向不一致，失效
```

---

## 题目 2：如何定位并优化慢查询？

### 参考答案

**Step 1：开启慢查询日志**
```sql
-- 查看是否开启
SHOW VARIABLES LIKE 'slow_query_log%';

-- 临时开启（重启失效）
SET GLOBAL slow_query_log = 'ON';
SET GLOBAL long_query_time = 2;  -- 超过 2 秒记录
```

**Step 2：使用 EXPLAIN 分析执行计划**
```sql
EXPLAIN SELECT u.name, o.amount
FROM orders o
JOIN users u ON o.user_id = u.id
WHERE o.status = 1;

-- 关键字段：
-- type: 从好到差：system > const > eq_ref > ref > range > index > ALL
-- key: 实际使用的索引
-- rows: 扫描行数，越少越好
-- Extra: Using filesort / Using temporary 表示需要优化
```

**Step 3：使用 SHOW PROFILE**
```sql
SET profiling = 1;
执行你的查询;
SHOW PROFILES;
SHOW PROFILE FOR QUERY 1;  -- 查看各阶段耗时
```

**Step 4：常见优化手段**
| 优化方向 | 具体做法 |
|---------|---------|
| 索引优化 | 添加合适索引，利用覆盖索引避免回表 |
| SQL 改写 | 避免 SELECT *，减少 JOIN 层数 |
| 分库分表 | 大表拆分为历史表 + 热表 |
| 读写分离 | 主从复制，将读请求打到从库 |
| LIMIT 优化 | 使用延迟关联：先索引覆盖查 ID，再关联 |

---

## 题目 3：MySQL InnoDB 与 MyISAM 存储引擎的核心区别是什么？

### 参考答案

| 特性 | InnoDB | MyISAM |
|------|--------|--------|
| **事务支持** | 支持 ACID 事务 | 不支持 |
| **锁粒度** | 行级锁 + 间隙锁 | 表级锁 |
| **并发能力** | 高 | 低（写操作会锁表） |
| **外键** | 支持 | 不支持 |
| **崩溃恢复** | 自动崩溃恢复（redo log） | 需手动修复（myisamchk） |
| **全文索引** | 5.6+ 支持 | 原生支持 |
| **存储结构** | 聚簇索引（主键索引和数据在一起） | 非聚簇索引（索引文件独立） |
| **count(\*)** | 全表扫描 | 保存行数（快） |
| **推荐场景** | 核心业务、高并发、需事务 | 读多写少、不需要事务 |

**面试追问点：**
- 为什么InnoDB建议主键自增？→ 减少页分裂，提升插入效率
- 为什么InnoDB适合做主从复制？→ 支持行级复制，binlog支持多种格式

---

## 题目 4：PostgreSQL 中如何分析并优化一条慢 SQL？

### 参考答案

**1. 查看慢查询**
```sql
-- 开启 pg_stat_statements（需先安装扩展）
CREATE EXTENSION pg_stat_statements;

-- 查看最慢的 10 条查询
SELECT query, calls, total_exec_time, mean_exec_time, rows
FROM pg_stat_statements
ORDER BY total_exec_time DESC
LIMIT 10;
```

**2. 使用 EXPLAIN ANALYZE**
```sql
EXPLAIN (ANALYZE, BUFFERS, FORMAT TEXT)
SELECT u.name, COUNT(o.id) as order_count
FROM users u
LEFT JOIN orders o ON u.id = o.user_id
WHERE u.created_at > '2024-01-01'
GROUP BY u.name;
```

**3. 关键指标解读**
- **Seq Scan**：全表扫描，尽量少
- **Bitmap Heap Scan**：索引扫描后的回表
- **Nested Loop / Hash Join / Merge Join**：选择合适的JOIN算法
- **Buffers: shared hit**：命中缓存的块数，比例越高越好

**4. 常用优化手段**
```sql
-- 创建合适索引（BRIN 索引适合有序数据）
CREATE INDEX idx_orders_created ON orders USING BRIN (created_at);

-- 表达式索引（避免函数导致索引失效）
CREATE INDEX idx_users_year ON users ((DATE_PART('year', created_at)));

-- 物化视图加速聚合查询
CREATE MATERIALIZED VIEW monthly_sales AS
SELECT DATE_TRUNC('month', created_at) as month, SUM(amount) as total
FROM orders GROUP BY 1;

REFRESH MATERIALIZED VIEW CONCURRENTLY monthly_sales;
```

---

## 题目 5：如何设计一个高效的数据库缓存策略？

### 参考答案

**缓存策略分层**

```
┌─────────────┐     ┌─────────────┐     ┌─────────────┐
│   Client    │ ──▶ │   Redis     │ ──▶ │  Database   │
│  (本地缓存)  │     │  (分布式缓存) │     │  (MySQL/PG)  │
└─────────────┘     └─────────────┘     └─────────────┘
```

**1. 缓存读写模式：Cache-Aside（最常用）**
```python
# 读
def get_user(user_id):
    user = redis.get(f"user:{user_id}")
    if user:
        return json.loads(user)
    user = db.query("SELECT * FROM users WHERE id = %s", user_id)
    redis.setex(f"user:{user_id}", 3600, json.dumps(user))
    return user

# 写
def update_user(user_id, data):
    db.execute("UPDATE users SET ... WHERE id = %s", user_id)
    redis.delete(f"user:{user_id}")  # 删除缓存，而非更新
```

**2. 缓存淘汰策略选择**
| 策略 | 适用场景 |
|------|---------|
| LRU | 数据访问频率差异大 |
| LFU | 热点数据明显 |
| TTL | 数据时效性要求高 |

**3. 缓存问题应对**

- **缓存穿透**：布隆过滤器 or 存空值（TTL要短）
- **缓存击穿**：互斥锁 or 热点数据永不过期
- **缓存雪崩**：随机TTL + 熔断降级

```python
# 互斥锁防击穿
def get_user_lock(user_id):
    key = f"user:{user_id}"
    lock_key = f"lock:{key}"
    
    val = redis.get(key)
    if val:
        return json.loads(val)
    
    # 尝试加锁
    if redis.set(lock_key, "1", nx=True, ex=10):
        user = db.query(...)
        redis.setex(key, 3600, json.dumps(user))
        redis.delete(lock_key)
        return user
    else:
        time.sleep(0.1)
        return get_user_lock(user_id)  # 重试
```

**4. 缓存一致性方案**
- **延迟双删**：先删缓存 → 更新数据库 → 延迟再删缓存
- **订阅 binlog**：用 Canal 监听数据库变更，同步更新 Redis

---

> 📚 **推荐阅读**
> - 《高性能MySQL》— Barbara Detrick et al.
> - PostgreSQL 官方文档：https://www.postgresql.org/docs/current/using-explain.html
> - MySQL 官方文档：https://dev.mysql.com/doc/refman/8.0/en/optimization.html