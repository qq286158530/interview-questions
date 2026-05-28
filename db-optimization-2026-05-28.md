# 数据库性能优化面试题

> 本文档整理了5道高质量的MySQL/PostgreSQL数据库性能优化面试题，涵盖索引优化、查询优化、存储引擎、缓存策略等核心知识点。  
> 来源说明：题目基于业界经典面试题库整理，结合公开技术文章与面试经验总结。

---

## 题目一：MySQL索引失效的常见场景有哪些？

### 参考答案

在实际开发中，索引失效会导致查询性能急剧下降。以下是**最常见的索引失效场景**：

### 1. 使用函数或运算
```sql
-- 索引失效 ❌
SELECT * FROM orders WHERE YEAR(create_time) = 2024;
SELECT * FROM users WHERE age + 1 > 30;

-- 正确写法 ✅
SELECT * FROM orders WHERE create_time >= '2024-01-01' AND create_time < '2025-01-01';
```

### 2. 使用 LIKE 前缀通配符
```sql
-- 索引失效 ❌（%在最前面）
SELECT * FROM products WHERE name LIKE '%手机%';

-- 索引生效 ✅（%在最后面）
SELECT * FROM products WHERE name LIKE '华为%';
```

### 3. 类型转换
```sql
-- phone定义为varchar，但传入数字，索引失效 ❌
SELECT * FROM users WHERE phone = 13800138000;

-- 正确写法 ✅
SELECT * FROM users WHERE phone = '13800138000';
```

### 4. OR连接条件
```sql
-- 任一条件列没有索引，整条语句索引失效 ❌
SELECT * FROM users WHERE name = '张三' OR age = 25;

-- 建议：改用IN或UNION拆解
SELECT * FROM users WHERE name = '张三'
UNION
SELECT * FROM users WHERE age = 25;
```

### 5. 隐式类型转换
字符型与数字型比较时，MySQL会自动将字符转为数字，导致索引失效。

### 6. 使用 NOT IN、<>、IS NULL
```sql
-- 可能导致全表扫描 ❌
SELECT * FROM orders WHERE status IS NULL;
SELECT * FROM users WHERE age NOT IN (20, 30);
```

### 7. 联合索引违反最左前缀原则
```sql
-- 建立联合索引 (a, b, c)
-- 以下哪些能命中索引？
WHERE a = 1           -- ✅ 命中
WHERE a = 1 AND b = 2 -- ✅ 命中
WHERE a = 1 AND c = 3 -- ✅ 命中（只用到a）
WHERE b = 2           -- ❌ 不命中
WHERE c = 3           -- ❌ 不命中
WHERE b = 2 AND c = 3 -- ❌ 不命中
```

---

## 题目二：如何定位并优化慢查询？

### 参考答案

### Step 1：开启慢查询日志
```sql
-- 查看是否开启
SHOW VARIABLES LIKE 'slow_query_log%';
SHOW VARIABLES LIKE 'long_query_time%';

-- 开启（临时生效）
SET GLOBAL slow_query_log = 'ON';
SET GLOBAL long_query_time = 1; -- 超过1秒记录
```

### Step 2：分析慢查询日志
使用 `mysqldumpslow` 工具：
```bash
mysqldumpslow -t 5 /var/log/mysql/slow.log   # 取最慢的5条
```

### Step 3：使用 EXPLAIN 分析执行计划
```sql
EXPLAIN SELECT * FROM orders WHERE user_id = 10086;
```
关键字段：
- **type**：最好达到 `ref`/`range`，避免 `ALL`（全表扫描）
- **key**：实际使用的索引
- **rows**：扫描行数，越少越好
- **Extra**：Using filesort / Using temporary 表示需要优化

### Step 4：使用 PROFILING 分析开销
```sql
SET profiling = 1;
SELECT ...;
SHOW PROFILES;
SHOW PROFILE FOR QUERY 1;
```

### Step 5：针对性优化
| 问题 | 优化手段 |
|------|---------|
| 全表扫描 | 添加合适索引 |
| Using filesort | 添加ORDER BY字段索引 |
| Using temporary | 减少SELECT字段数量 |
| 连接效率低 | 调整JOIN顺序，小表驱动大表 |

### Step 6：业务层优化
- 分页优化（延迟关联 / 游标分页）
- 避免深分页
- 读写分离
- 适当使用缓存

---

## 题目三：MySQL InnoDB与MyISAM存储引擎的区别是什么？

### 参考答案

| 特性 | InnoDB | MyISAM |
|------|--------|--------|
| **事务支持** | 支持ACID事务 | 不支持事务 |
| **行锁粒度** | 行锁（高并发） | 表锁 |
| **外键约束** | 支持 | 不支持 |
| **全文索引** | 5.6+支持，之前不支持 | 支持 |
| **存储结构** | 聚簇索引（数据和索引在一起） | 非聚簇索引（索引与数据分离） |
| **Crash安全** | 支持自动恢复 | 不支持 |
| **COUNT(*)性能** | 相对较慢（需要扫描全表） | 快（行数存在元数据） |
| **适用场景** | 核心业务、高并发、需事务 | 读多写少、不需要事务的场景 |

### 核心区别解读

**1. 聚簇索引 vs 非聚簇索引**
InnoDB的B+Tree叶子节点直接存储数据行（聚簇），MyISAM叶子节点存储的是数据地址（非聚簇）。这意味着InnoDB读取数据少一次磁盘IO。

**2. 事务与锁机制**
InnoDB通过MVCC支持并发读写，而MyISAM在任何操作期间都会锁整张表，高并发下性能急剧下降。

**3. Crash Safe**
InnoDB使用Redo Log保证崩溃后数据不丢失，MyISAM在突然断电后可能丢失数据或导致索引损坏。

**面试加分点**：InnoDB的聚簇索引设计使得主键查询非常快，但二级索引查询需要回表两次查B+Tree；而MyISAM的所有索引都是非聚簇的，查询都要多一次磁盘寻址。

---

## 题目四：PostgreSQL中有哪些性能优化手段？

### 参考答案

### 1. 索引优化
```sql
-- 常用索引类型
CREATE INDEX idx_user ON orders(user_id);                    -- B-tree（默认）
CREATE INDEX idx_geom ON locations USING GIST(geom);        -- GiST（地理数据）
CREATE INDEX idx_name ON users USING gin(to_tsvector('zhcfg', name)); -- GIN（全文搜索）
```

### 2. 查询优化技巧
```sql
-- 避免SELECT *，只取需要的字段
SELECT user_id, order_status FROM orders WHERE user_id = 1;

-- 使用EXPLAIN ANALYZE分析执行计划
EXPLAIN (ANALYZE, BUFFERS, FORMAT TEXT)
SELECT * FROM orders WHERE create_time > '2024-01-01';

-- 批量插入优化
COPY orders FROM '/tmp/orders.csv' WITH (FORMAT csv);
```

### 3. 分区表（Table Partitioning）
```sql
-- 按时间分区
CREATE TABLE orders (
    id BIGSERIAL,
    create_time TIMESTAMP NOT NULL,
    amount NUMERIC(10,2)
) PARTITION BY RANGE (create_time);

CREATE TABLE orders_2024 PARTITION OF orders
    FOR VALUES FROM ('2024-01-01') TO ('2025-01-01');
```
分区后查询可以只扫描对应分区，大幅减少扫描量。

### 4. 配置参数调优
```ini
# postgresql.conf
shared_buffers = 256MB          # 共享缓冲区，建议为服务器内存的25%
effective_cache_size = 768MB     # 有效缓存，服务器内存的75%
work_mem = 64MB                  # 排序/哈希操作内存
maintenance_work_mem = 128MB     # 维护操作内存（VACUUM、CREATE INDEX等）
```

### 5. 使用连接池（PgBouncer）
```ini
; pgbouncer.ini
pool_mode = transaction
max_client_conn = 200
default_pool_size = 20
```

### 6. 监控慢查询
```sql
-- 查看当前最慢的查询
SELECT pid, now() - query_start AS duration, query
FROM pg_stat_activity
WHERE state != 'idle' AND now() - query_start > interval '1 minute'
ORDER BY duration DESC;
```

---

## 题目五：如何设计一套数据库缓存策略？

### 参考答案

### 缓存架构设计

```
用户请求 → Redis缓存 → (未命中) → MySQL数据库
```

### 1. 缓存Key设计
```python
# 按业务维度设计key，避免冲突
# 用户信息缓存
user:info:{user_id}          -> JSON/Hash
# 订单列表缓存（带分页）
user:orders:{user_id}:page:{page}:size:{size}
# 热点商品缓存
product:hot:{product_id}
```

### 2. 缓存过期策略（LRU + TTL）
```python
# 为每个缓存设置TTL，分层过期避免雪崩
CACHE_TTL = {
    'user_info': 3600,      # 1小时
    'product_detail': 1800, # 30分钟
    'hot_product': 300,     # 5分钟
}
```

### 3. 缓存击穿解决方案
| 方案 | 原理 | 适用场景 |
|------|------|---------|
| **互斥锁**（Mutex） | 只允许一个请求查库写缓存 | 数据一致性要求高 |
| **逻辑过期** | 数据永不过期，用版本号控制更新 |数据一致性要求一般 |
| **布隆过滤器** | 提前过滤不存在的key | 防止缓存穿透 |

```python
# 互斥锁实现示例（Redis SETNX）
def get_user_info(user_id):
    key = f"user:info:{user_id}"
    info = redis.get(key)
    if info:
        return json.loads(info)
    
    # 获取锁
    lock_key = f"lock:{key}"
    if redis.set(lock_key, "1", nx=True, ex=10):
        # 查数据库
        info = db.query("SELECT * FROM users WHERE id = %s", user_id)
        redis.setex(key, 3600, json.dumps(info))
        redis.delete(lock_key)
        return info
    else:
        time.sleep(0.1)
        return get_user_info(user_id)  # 等待后重试
```

### 4. 缓存与数据库一致性
- **Cache Aside**（最常用）：读时先缓存再库，写时先库再删缓存
- **Read Through**：读请求由缓存服务负责加载
- **Write Through**：写操作同时更新缓存和数据库

### 5. 缓存容量规划
- 预估热点数据量：80%请求落在20%数据上
- 预留30%容量，防止内存碎片
- 监控Redis内存使用率，超过80%需扩容或清理

---

## 参考来源

1. [MySQL索引失效场景全解](https://zhuanlan.zhihu.com/p/144782912)
2. [慢查询优化方法论](https://www.mysql.com/cn/documentation/)
3. [InnoDB与MyISAM存储引擎对比](https://dev.mysql.com/doc/refman/8.0/en/innodb-introduction.html)
4. [PostgreSQL性能优化指南](https://www.postgresql.org/docs/current/performance-tips.html)
5. [Redis缓存策略设计](https://redis.io/docs/manualpatterns/)

---

*整理日期：2026-05-28*
*🐼 by 小憨宝面试助手*