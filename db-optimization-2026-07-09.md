# 数据库性能优化面试题（2026-07-09）

> 本文收录于 GitHub 开源项目 [qq286158530/interview-questions](https://github.com/qq286158530/interview-questions)，每日更新，欢迎 Star！

---

## 题目一：MySQL 中 InnoDB 索引失效的常见场景有哪些？

### 参考答案

索引失效是面试中的高频问题，以下场景极易被考察：

**1. 使用函数或运算**
```sql
-- 索引失效
SELECT * FROM orders WHERE YEAR(create_time) = 2026;
SELECT * FROM users WHERE age + 1 > 20;

-- 正确写法：保留索引可用
SELECT * FROM orders WHERE create_time >= '2026-01-01' AND create_time < '2027-01-01';
```

**2. 使用 LIKE 前缀通配符**
```sql
-- 索引失效（%在开头）
SELECT * FROM products WHERE name LIKE '%手机';

-- 索引有效（%在末尾）
SELECT * FROM products WHERE name LIKE '华为%';
```

**3. 类型隐式转换**
```sql
-- phone 是 VARCHAR 类型，传入数字会隐式转换导致索引失效
SELECT * FROM users WHERE phone = 13800138000;  -- 失效
SELECT * FROM users WHERE phone = '13800138000'; -- 有效
```

**4. OR 连接条件中存在非索引列**
```sql
-- 只有 name 有索引，age 无索引，导致索引失效
SELECT * FROM users WHERE name = '张三' OR age > 30;

-- 修复：为 age 添加索引，或拆分为 UNION
```

**5. 联合索引违反最左前缀原则**
```sql
-- 联合索引 (a, b, c)
-- 以下情况可以使用索引：
WHERE a = 1
WHERE a = 1 AND b = 2
WHERE a = 1 AND b = 2 AND c = 3

-- 以下情况索引失效：
WHERE b = 2
WHERE c = 3
WHERE b = 2 AND c = 3
```

**6. 使用 NOT IN、<>、NOT EXISTS**
```sql
-- 通常索引失效
SELECT * FROM orders WHERE status <> '已完成';
SELECT * FROM users WHERE id NOT IN (1, 2, 3);
```

---

## 题目二：如何排查和优化一条慢查询（Slow Query）？

### 参考答案

**Step 1：开启慢查询日志**

```ini
# my.cnf 配置
slow_query_log = 1
slow_query_log_file = /var/log/mysql/slow.log
long_query_time = 1  # 超过1秒记录
```

**Step 2：使用 EXPLAIN 分析执行计划**

```sql
EXPLAIN SELECT u.name, o.amount
FROM users u
INNER JOIN orders o ON u.id = o.user_id
WHERE u.status = 'active';
```

重点关注字段：
- **type**：访问类型，const/eq_ref/ref 为最优，ALL 为全表扫描
- **key**：实际使用的索引
- **rows**：扫描行数，越少越好
- **Extra**：Using filesort、Using temporary 表示需要优化

**Step 3：使用 SHOW PROFILE**

```sql
SET profiling = 1;
SELECT ...;  -- 执行查询
SHOW PROFILES;
SHOW PROFILE FOR QUERY 1;  -- 查看各阶段耗时
```

**Step 4：常见优化手段**

| 优化手段 | 说明 |
|---------|------|
| 添加适当索引 | 为 WHERE、JOIN、ORDER BY、GROUP BY 涉及列建索引 |
| 避免 SELECT * | 只查询需要的字段 |
| 拆分大查询 | 分批处理，避免锁表时间过长 |
| 优化子查询 | 改用 JOIN 或临时表 |
| 分页优化 | 使用延迟关联或游标分页 |

---

## 题目三：MySQL InnoDB 与 MyISAM 存储引擎的区别？如何选择？

### 参考答案

| 特性 | InnoDB | MyISAM |
|------|--------|--------|
| 事务支持 | 支持 ACID 事务 | 不支持 |
| 行级锁 | 支持行锁，并发性能高 | 只支持表锁 |
| 外键约束 | 支持 | 不支持 |
| 崩溃恢复 | 支持自动恢复 | 需手动修复 |
| 全文索引 | MySQL 5.6+ 支持 | 支持 |
| 索引缓存 | 自适应哈希索引 | 独立缓存区 |
| 存储空间 | 约2倍于MyISAM | 较小 |

**如何选择：**

- **选择 InnoDB**：生产环境首选（事务安全、高并发、行锁、外键支持）
- **选择 MyISAM**：只读场景、静态表、日志表（已被淘汰，一般不推荐）

**重要变化**：MySQL 8.0+ 已将默认存储引擎从 MyISAM 改为 InnoDB，新项目务必使用 InnoDB。

---

## 题目四：PostgreSQL 中如何进行 SQL 查询调优？请列举常用工具和技巧。

### 参考答案

**1. EXPLAIN 和 EXPLAIN ANALYZE**

```sql
-- 分析执行计划（不实际执行）
EXPLAIN SELECT * FROM orders WHERE user_id = 123;

-- 实际执行并分析（生产环境慎用）
EXPLAIN (ANALYZE, BUFFERS, FORMAT TEXT)
SELECT * FROM orders WHERE user_id = 123;
```

关键字段：
- **Seq Scan**：顺序扫描，通常意味着需要优化
- **Index Scan / Index Only Scan**：使用索引
- **Bitmap Heap Scan**：索引扫描后批量取数据
- **Buffers: shared hit**：命中缓存的块数

**2. pg_stat_statements 插件**

```sql
-- 开启插件（需在 postgresql.conf 中配置 shared_preload_libraries）
CREATE EXTENSION pg_stat_statements;

-- 查询最慢的SQL
SELECT query, calls, mean_time, total_time
FROM pg_stat_statements
ORDER BY mean_time DESC
LIMIT 10;

-- 查询执行次数最多的SQL
SELECT query, calls
FROM pg_stat_statements
ORDER BY calls DESC
LIMIT 10;
```

**3. 索引优化**

```sql
-- B-tree 索引（默认，最常用）
CREATE INDEX idx_orders_user_id ON orders(user_id);

-- 表达式索引
CREATE INDEX idx_orders_year ON orders(DATE_PART('year', create_time));

-- 部分索引（只索引满足条件的行）
CREATE INDEX idx_orders_active ON orders(user_id)
WHERE status = 'active';

-- 覆盖索引（避免回表）
CREATE INDEX idx_orders_cover ON orders(user_id) INCLUDE (amount, status);
```

**4. VACUUM 和 ANALYZE**

```sql
-- 清理死亡元组并更新统计信息
VACUUM ANALYZE orders;

-- 查看表膨胀情况
SELECT relname, n_dead_tup, n_live_tup, last_vacuum, last_autovacuum
FROM pg_stat_user_tables
WHERE n_dead_tup > 1000;
```

---

## 题目五：数据库缓存策略如何设计？Redis 与 MySQL 如何配合使用？

### 参考答案

**经典缓存模式：Cache-Aside（旁路缓存）**

```
读操作：
1. 先查 Redis 缓存
2. 命中则返回
3. 未命中则查 MySQL
4. 结果写入 Redis 并设置过期时间
5. 返回结果

写操作：
1. 先更新 MySQL
2. 再删除（而非更新）Redis 缓存
   （删除而非更新是为了避免并发时的数据不一致）
```

**示例代码（Python）：**

```python
def get_user(user_id):
    # 1. 查缓存
    cache_key = f"user:{user_id}"
    user = redis.get(cache_key)
    if user:
        return json.loads(user)
    
    # 2. 缓存未命中，查数据库
    user = db.query("SELECT * FROM users WHERE id = %s", user_id)
    
    # 3. 写入缓存，设置1小时过期
    redis.setex(cache_key, 3600, json.dumps(user))
    return user

def update_user(user_id, data):
    # 1. 更新数据库
    db.execute("UPDATE users SET ... WHERE id = %s", user_id)
    
    # 2. 删除缓存（注意：不是更新！）
    redis.delete(f"user:{user_id}")
```

**缓存常见问题及解决方案：**

| 问题 | 原因 | 解决方案 |
|------|------|---------|
| 缓存穿透 | 查询不存在的数据 | 布隆过滤器 / 缓存空值 |
| 缓存击穿 | 热点key过期瞬间 | 互斥锁 / 永不过期 + 异步更新 |
| 缓存雪崩 | 大量key同时过期 | 过期时间加随机值 / 多级缓存 |

**过期时间设置建议：**
- 频繁变化的数据：30秒～5分钟
- 相对稳定的数据：10～60分钟
- 配置类数据：可设置几小时甚至一天

---

## 📚 参考来源

1. MySQL 官方文档 - InnoDB Storage Engine：https://dev.mysql.com/doc/refman/8.0/en/innodb-storage-engine.html
2. PostgreSQL 官方文档 - Using EXPLAIN：https://www.postgresql.org/docs/current/using-explain.html
3. 阿里云开发者社区 - MySQL 性能优化实战：https://developer.aliyun.com/
4. 腾讯云技术社区 - 数据库索引设计指南：https://cloud.tencent.com/developer/
5. 阮一峰的网络日志 - Redis 缓存技术：https://www.ruanyifeng.com/blog/
