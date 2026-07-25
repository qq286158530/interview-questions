# 数据库性能优化面试题 · 2026-07-25

> 涵盖 MySQL 与 PostgreSQL 核心知识点：索引设计、查询调优、存储引擎、缓存策略、配置优化

---

## 题目一：MySQL 中索引失效的常见场景有哪些？如何避免？

### 参考答案

索引失效（Index Not Used）是面试中的高频问题，常见场景如下：

#### 1. 使用函数或运算
```sql
-- 索引失效 ❌
SELECT * FROM orders WHERE YEAR(create_time) = 2026;
SELECT * FROM users WHERE age + 1 > 30;

-- 正确写法 ✅
SELECT * FROM orders WHERE create_time >= '2026-01-01' AND create_time < '2027-01-01';
```

#### 2. 类型隐式转换
```sql
-- phone 是 varchar，传入数字 → 隐式转换，索引失效 ❌
SELECT * FROM users WHERE phone = 13800138000;

-- 正确写法 ✅（使用字符串）
SELECT * FROM users WHERE phone = '13800138000';
```

#### 3. LIKE 前缀通配符
```sql
-- %在前面，索引失效 ❌
SELECT * FROM products WHERE name LIKE '%手机%';

-- 只在后面，索引生效 ✅
SELECT * FROM products WHERE name LIKE '苹果%';
```

#### 4. OR 条件中包含非索引列
```sql
-- 如果 status 没有索引，整个查询索引失效 ❌
SELECT * FROM orders WHERE user_id = 1 OR status = 'paid';

-- 改用 UNION ✅
SELECT * FROM orders WHERE user_id = 1
UNION ALL
SELECT * FROM orders WHERE status = 'paid' AND user_id != 1;
```

#### 5. 联合索引不遵循最左前缀原则
```sql
-- 联合索引 (a, b, c)
-- 失效场景 ❌
SELECT * FROM t WHERE b = 2;
SELECT * FROM t WHERE c = 3;

-- 生效场景 ✅
SELECT * FROM t WHERE a = 1;
SELECT * FROM t WHERE a = 1 AND b = 2;
```

#### 6. 使用 NOT IN、<>、IS NOT NULL
```sql
-- 索引失效 ❌（InnoDB 对 NOT NULL 优化较差）
SELECT * FROM users WHERE email IS NOT NULL;
SELECT * FROM orders WHERE status != 'cancelled';
```

#### 避免策略
- 使用 `EXPLAIN` 分析执行计划，确认索引是否被使用
- 尽量使用覆盖索引（Covering Index），减少回表
- 对高频查询建立合适的联合索引，并遵循最左前缀原则
- 避免在 WHERE 条件中对字段做运算或函数处理

---

## 题目二：MySQL InnoDB 与 MyISAM 存储引擎的区别？各自适用场景是什么？

### 参考答案

| 特性 | InnoDB | MyISAM |
|------|--------|--------|
| **事务支持** | 支持 ACID 事务 | 不支持事务 |
| **锁粒度** | 行级锁（高并发） | 表级锁 |
| **外键约束** | 支持外键 | 不支持外键 |
| **崩溃恢复** | 支持自动恢复（redo log） | 崩溃恢复能力弱 |
| **全文索引** | MySQL 5.6+ 支持 | 原生支持全文索引 |
| **存储结构** | 聚簇索引（数据在B+树叶子节点） | 非聚簇索引（索引与数据分离） |
| **COUNT(\*)** | 全表扫描（带 WHERE 时） | 保存行数，快速返回 |
| **适用场景** | 高并发、事务安全 | 读多写少、日志表 |

#### 面试加分点
```
InnoDB 选择聚簇索引的优点：
- 数据访问是单次 I/O（索引树 + 数据在一起）
- 范围查询快（B+树叶子节点连续存储）

InnoDB 的缺点：
- 辅助索引查询需回表（先查辅助索引，再查聚簇索引）
- 每个 InnoDB 表都有一个隐藏的主键列（6字节），如果表没有主键会选择一个唯一键或生成内部主键
```

#### 适用场景总结
- **InnoDB**：互联网业务、订单系统、支付系统、金融类应用（几乎所有 OLTP 场景）
- **MyISAM**：只读报表、日志分析、批量数据导入（已逐渐被 InnoDB 替代）

---

## 题目三：如何优化慢查询？请描述你的排查思路和优化方法。

### 参考答案

#### 排查思路（五步法）

**第一步：开启慢查询日志**
```sql
-- 查看慢查询配置
SHOW VARIABLES LIKE 'slow_query%';
SHOW VARIABLES LIKE 'long_query_time';

-- 开启慢查询日志
SET GLOBAL slow_query_log = 'ON';
SET GLOBAL long_query_time = 1;  -- 超过1秒记录
SET GLOBAL slow_query_log_file = '/var/log/mysql/slow.log';
```

**第二步：使用 EXPLAIN 分析执行计划**
```sql
EXPLAIN SELECT u.name, o.total
FROM orders o
JOIN users u ON o.user_id = u.id
WHERE o.create_time > '2026-01-01';

-- 关键字段解读：
-- type: const > eq_ref > ref > range > index > ALL（至少要到 ref 级别）
-- key: 实际使用的索引
-- rows: 扫描行数（越少越好）
-- Extra: Using filesort / Using temporary → 需要优化
```

**第三步：检查索引是否合理**
```sql
-- 查看表的所有索引
SHOW INDEX FROM orders;

-- 使用 optimizer_trace 分析复杂查询
SET SESSION optimizer_trace = 'enabled=ON';
SELECT ...;
SELECT * FROM INFORMATION_SCHEMA.OPTIMIZER_TRACE;
```

**第四步：使用 PROFILING**
```sql
SET profiling = 1;
SELECT ...;
SHOW PROFILES;           -- 查看所有查询耗时
SHOW PROFILE FOR QUERY 1; -- 查看具体阶段耗时
```

**第五步：定位 TOP SQL**
```sql
-- MySQL 8.0+ 可使用 performance_schema
SELECT DIGEST_TEXT AS query,
       COUNT_STAR AS exec_count,
       SUM_TIMER_WAIT/1000000000000 AS total_latency_s
FROM performance_schema.events_statements_summary_by_digest
ORDER BY SUM_TIMER_WAIT DESC
LIMIT 10;
```

#### 优化方法

| 优化手段 | 说明 |
|---------|------|
| 索引优化 | 覆盖索引减少回表、联合索引最左前缀、删除冗余索引 |
| SQL 重写 | 改写子查询为 JOIN、分解 IN (大量值)、避免 SELECT * |
| 分页优化 | 延迟关联 / 游标分页替代 OFFSET |
| 表结构优化 | 适当冗余字段避免 JOIN、拆表（冷热分离） |
| 架构优化 | 读写分离、主从延迟处理、引入缓存层 |

#### 分页优化示例
```sql
-- 低效：OFFSET 大时极慢 ❌
SELECT * FROM orders LIMIT 100000, 20;

-- 高效：延迟关联 ✅
SELECT o.* FROM orders o
INNER JOIN (
    SELECT id FROM orders
    ORDER BY id
    LIMIT 100000, 20
) t ON o.id = t.id;

-- 最优：游标分页（基于上一页最大ID）✅
SELECT * FROM orders
WHERE id > 100000
ORDER BY id
LIMIT 20;
```

---

## 题目四：PostgreSQL 中如何进行性能调优？涉及哪些关键参数和工具？

### 参考答案

#### 关键配置参数（postgresql.conf）

```ini
# ==== 内存相关 ====
shared_buffers = 25% of RAM          # 共享缓冲区（建议 OS 内存的 25%）
work_mem = 64MB                       # 单个排序/哈希操作内存
maintenance_work_mem = 512MB          # VACUUM、CREATE INDEX 等维护操作内存

# ==== 写入相关 ====
wal_buffers = 16MB                    # WAL 日志缓冲区
checkpoint_completion_target = 0.9    # 检查点分布（减少 I/O 尖峰）
max_wal_size = 2GB                     # WAL 最大保留量

# ==== 并发相关 ====
max_connections = 200                 # 最大连接数（连接池优于直接增加）
effective_cache_size = 75% of RAM     # 优化器估计可用缓存（索引+数据）

# ==== 查询规划 ====
random_page_cost = 1.1                # SSD 下设为 1.1（机械盘默认 4.0）
effective_io_concurrency = 200        # SSD 并发 I/O 能力
```

#### 常用调优工具

**1. pg_stat_statements（生产必备）**
```sql
-- 开启扩展
CREATE EXTENSION pg_stat_statements;

-- 查询最慢 SQL
SELECT query,
       calls,
       total_exec_time / 1000 AS total_ms,
       mean_exec_time AS mean_ms,
       rows / calls AS avg_rows
FROM pg_stat_statements
ORDER BY total_exec_time DESC
LIMIT 10;
```

**2. EXPLAIN ANALYZE**
```sql
EXPLAIN (ANALYZE, BUFFERS, FORMAT TEXT)
SELECT u.name, COUNT(o.id) as order_count
FROM users u
LEFT JOIN orders o ON u.id = o.user_id
WHERE u.created_at > '2026-01-01'
GROUP BY u.id;
-- BUFFERS: 显示共享块命中/读取情况，帮助判断缓存命中率
```

**3. pgAdmin / pg_stat_monitor 可视化工具**
- 实时监控 QPS、连接数、缓存命中率
- TOP SQL 分析

**4. VACUUM 和 ANALYZE**
```sql
-- 手动清理死亡元组，维护统计信息
VACUUM ANALYZE orders;

-- 查看表膨胀情况
SELECT relname,
       n_dead_tup,
       n_live_tup,
       last_vacuum,
       last_autovacuum
FROM pg_stat_user_tables
WHERE n_dead_tup > 10000
ORDER BY n_dead_tup DESC;
```

#### PostgreSQL vs MySQL 性能特点
```
PostgreSQL 优势：
- 更智能的成本模型（CBO 基于统计信息）
- 表达式索引 + 部分索引（灵活性更高）
- JSONB 完整支持（HStore 扩展）
- 窗口函数 / CTE（WITH RECURSIVE）性能更强
- 物理复制流复制延迟更低

MySQL 优势：
- 写入吞吐量（Redo Log 组提交优化）
- 简单场景下运维更轻量
- 生态（MyRocks、TiDB 兼容）
```

---

## 题目五：如何设计一个高效的数据库缓存策略？缓存与数据库一致性如何保证？

### 参考答案

#### 缓存策略设计

**缓存层级**
```
请求 → L1缓存（Nginx本地/Lua共享字典）→ L2缓存（Redis）→ 数据库
```

**缓存模式**

| 模式 | 原理 | 适用场景 | 缺点 |
|------|------|---------|------|
| Cache Aside | 先读缓存，缓存miss再查DB，更新时先删缓存 | 读多写少 | 删除缓存失败会导致不一致 |
| Read Through | 缓存负责加载数据，应用只和缓存交互 | 读多 | 实现复杂 |
| Write Through | 写数据库时同步写缓存 | 写入频繁 | 写入延迟增加 |
| Write Behind | 异步批量写缓存，再刷数据库 | 写入极高 | 数据丢失风险 |

#### 缓存与数据库一致性问题

**问题根源：并发读写导致的数据不一致**

```
时序问题示例（Cache Aside 下的数据不一致）：
T1: 客户端更新数据库 age=20
T2: 客户端删除缓存
T3: 另一个客户端读取 → 缓存miss → 查DB（age=20旧值）→ 写入缓存
结果：缓存中是旧值 age=20，而数据库已是 age=20（碰巧一致）
但如果T1和T3之间有值变更，就会不一致
```

**保证一致性的方案**

**方案1：延迟双删（推荐）**
```python
def update_user(user_id, age):
    # 1. 先更新数据库
    db.update(user_id, age)
    # 2. 删除缓存
    redis.delete(f"user:{user_id}")
    # 3. 延迟再删（异步，再删除一次）
    import threading
    threading.Timer(0.5, redis.delete, args=[f"user:{user_id}"]).start()
```

**方案2：设置过期时间（最终一致）**
```python
# 大多数场景用 TTL 兜底，不追求强一致
TTL = 5分钟
redis.setex(f"user:{user_id}", TTL, user_data)
```

**方案3：订阅 MySQL binlog（旁路缓存）**
```
MySQL binlog → Canal/Kafka → 解析 → 删缓存
适合大规模系统，保证最终一致且对业务代码无侵入
```

**缓存雪崩、穿透、击穿应对**

| 问题 | 原因 | 解决方案 |
|------|------|---------|
| 缓存雪崩 | 大量 key 同时过期 | 随机 TTL、构建多级缓存 |
| 缓存穿透 | 查询不存在的数据 | 布隆过滤器 / 缓存空值 |
| 缓存击穿 | 热点 key 过期瞬间大量请求 | 互斥锁 / 热点永不过期 |

**示例：分布式互斥锁防止缓存击穿**
```python
import redis

def get_user(user_id):
    cache_key = f"user:{user_id}"
    # 1. 先读缓存
    data = redis.get(cache_key)
    if data:
        return json.loads(data)
    
    # 2. 获取锁
    lock = redis.lock(f"lock:{cache_key}", timeout=10)
    if lock.acquire(blocking=False):
        try:
            # 3. 双重检查
            data = redis.get(cache_key)
            if data:
                return json.loads(data)
            # 4. 查数据库
            data = db.query("SELECT * FROM users WHERE id = ?", user_id)
            redis.setex(cache_key, 300, json.dumps(data))
            return data
        finally:
            lock.release()
    else:
        # 等待其他线程加载
        time.sleep(0.1)
        return get_user(user_id)
```

---

> 📚 **参考来源**
> - MySQL 慢查询优化：https://dev.mysql.com/doc/refman/8.0/en/slow-query-log.html
> - InnoDB 索引原理：https://dev.mysql.com/doc/refman/8.0/en/innodb-storage-engine.html
> - PostgreSQL 性能调优：https://www.postgresql.org/docs/current/runtime-config-query.html
> - PostgreSQL pg_stat_statements：https://www.postgresql.org/docs/current/pgstatstatements.html
> - Redis 缓存策略：https://redis.io/docs/manual patterns/cache/
> - 数据库热点问题：https://github.com/tenngyu/DB-Interview-Questions

*整理 by 小憨宝 · 2026-07-25*
