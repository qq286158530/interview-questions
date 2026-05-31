# 数据库性能优化面试题（2026-05-31）

> 本系列仓库：[qq286158530/interview-questions](https://github.com/qq286158530/interview-questions）  
> 来源说明：题目综合自业界常见面试场景，由资深数据库工程师整理。

---

## 题目一：MySQL 中 InnoDB 与 MyISAM 存储引擎的核心区别是什么？如何选择？

**答案：**

### 核心区别

| 特性 | InnoDB | MyISAM |
|------|--------|--------|
| **事务支持** | 支持 ACID 事务（commit/rollback） | 不支持事务 |
| **行级锁** | 支持行级锁，并发性能好 | 只支持表级锁 |
| **外键约束** | 支持外键 | 不支持外键 |
| **崩溃恢复** | 支持自动崩溃恢复（crash recovery） | 不支持，崩溃后易损坏 |
| **全文索引** | MySQL 5.6+ 才支持 | 原生支持全文索引 |
| **存储结构** | 聚簇索引（Clustered Index），数据与主键索引一起存储 | 非聚簇索引，数据独立存储 |
| **COUNT(*)** | 无where条件时需全表扫描 | 有专门计数器，快速返回 |
| **单表文件** | .ibd（独立表空间）+ .frm | .MYD + .MYI + .frm |

### InnoDB 的优势场景
- 有事务需求的业务系统（银行、电商订单等）
- 高并发读写场景（行级锁减少锁竞争）
- 需要数据崩溃恢复能力的生产环境
- 依赖外键做数据一致性约束

### MyISAM 的适用场景
- 以读为主、很少更新的场景（如静态配置表）
- 需要全文搜索的老旧系统（MySQL < 5.6）
- 只需要 COUNT(*) 大量统计的场景
- 近乎静态的数据仓库

### 面试追问
> 为什么不推荐使用 MyISAM？
- MyISAM 表级锁导致并发写入性能差
- 崩溃后无法自动恢复，数据可能丢失
- MySQL 8.0 已移除 MyISAM 存储引擎，官方不再维护

---

## 题目二：如何定位并优化慢查询（Slow Query）？请描述完整流程。

**答案：**

### Step 1：开启慢查询日志

```sql
-- 查看慢查询是否开启
SHOW VARIABLES LIKE 'slow_query_log';

-- 开启慢查询日志（永久生效需写入my.cnf）
SET GLOBAL slow_query_log = 'ON';
SET GLOBAL long_query_time = 1; -- 超过1秒记录
SET GLOBAL slow_query_log_file = '/var/log/mysql/slow.log';

-- 或在my.cnf中配置：
-- slow_query_log = 1
-- slow_query_log_file = /var/log/mysql/slow.log
-- long_query_time = 1
```

### Step 2：使用 EXPLAIN 分析执行计划

```sql
EXPLAIN SELECT ...
EXPLAIN ANALYZE SELECT ...  -- MySQL 8.0+，会实际执行并显示真实代价
```

重点关注字段：
- **type**：ALL（全表扫描）最差，eq_ref/ref/range 较好
- **key**：实际使用的索引
- **rows**：预估扫描行数，越少越好
- **Extra**：Using filesort / Using temporary 说明需要优化

### Step 3：使用 SHOW PROFILE（MySQL 5.7-）

```sql
SET profiling = 1;
-- 执行目标查询
SHOW PROFILE ALL FOR QUERY {query_id};
```

### Step 4：常见优化手段

#### 4.1 索引优化
```sql
-- 为WHERE条件字段、JOIN字段、ORDER BY字段添加索引
ALTER TABLE orders ADD INDEX idx_user_id (user_id);
ALTER TABLE orders ADD INDEX idx_created_status (created_at, status);

-- 复合索引遵循最左前缀原则
-- 查询 WHERE a=1 AND b=2 -> 索引 (a, b) 有效
-- 查询 WHERE b=2         -> 索引 (a, b) 无效（未使用左前缀）
```

#### 4.2 避免全表扫描
```sql
-- ❌ 避免
SELECT * FROM orders WHERE status != 'paid';

-- ✅ 改写为
SELECT * FROM orders WHERE status = 'paid' UNION ALL SELECT * FROM orders WHERE status != 'paid';

-- 或使用覆盖索引
SELECT id, status FROM orders WHERE status = 'paid';
```

#### 4.3 避免文件排序和临时表
```sql
-- ❌ Using filesort
SELECT * FROM orders ORDER BY created_at DESC LIMIT 10;

-- ✅ 优化：利用索引的有序性
ALTER TABLE orders ADD INDEX idx_created_at (created_at DESC);

-- ✅ 验证
EXPLAIN SELECT * FROM orders ORDER BY created_at DESC LIMIT 10;
-- Extra 应为 NULL，key 应为 idx_created_at
```

#### 4.4 分页优化
```sql
-- ❌ 深分页问题（偏移量大时性能差）
SELECT * FROM orders LIMIT 1000000, 10;

-- ✅ 优化：基于主键游标
SELECT * FROM orders WHERE id > #{last_id} ORDER BY id LIMIT 10;

-- ✅ 优化：延迟关联
SELECT t.* FROM (
    SELECT id FROM orders ORDER BY id LIMIT 1000000, 10
) AS r JOIN orders t ON t.id = r.id;
```

#### 4.5 减少 JOIN 操作
```sql
-- ❌ 大表JOIN
SELECT * FROM orders o JOIN users u ON o.user_id = u.id
JOIN products p ON o.product_id = p.id;

-- ✅ 拆分查询或使用应用层JOIN
```

### Step 5：线上巡检工具

- **pt-query-digest**（Percona Toolkit）：分析慢查询日志，自动归类同类慢查询
- **Performance Schema**：MySQL 5.6+，记录所有语句执行信息
- **information_schema.tables**：定期检查表碎片

```sql
-- 检查表碎片
SELECT TABLE_NAME, (DATA_LENGTH + INDEX_LENGTH) / 1024 / 1024 AS 'MB',
       DATA_FREE / 1024 / 1024 AS '碎片MB'
FROM information_schema.tables
WHERE TABLE_SCHEMA = 'your_db'
  AND DATA_FREE > 10 * 1024 * 1024; -- 碎片 > 10MB

-- 清理碎片
OPTIMIZE TABLE your_table;
```

---

## 题目三：MySQL 聚簇索引（Clustered Index）的原理是什么？它如何影响查询性能？

**答案：**

### 原理

聚簇索引并不是一种单独的索引类型，而是一种**数据存储方式**。

InnoDB 的聚簇索引特点：
1. **表数据按照主键顺序存储**，每张表只能有一个聚簇索引
2. 主键索引叶子节点存储的是**完整的行数据**（而不是主键值）
3. 二级索引（Secondary Index）叶子节点存储的是**主键值**，回表时需要再次查找主键索引

### 结构图示

```
聚簇索引（主键索引）结构：
┌─────────────┬──────────────────────────────────┐
│ 主键值      │ 整行数据（叶子节点存储完整行）      │
├─────────────┼──────────────────────────────────┤
│ 1           │ id=1, name='Alice', age=25, ...  │
│ 3           │ id=3, name='Bob', age=30, ...     │
│ 7           │ id=7, name='Carol', age=28, ...  │
└─────────────┴──────────────────────────────────┘

二级索引（user_name）结构：
┌─────────────┬────────┐
│ name        │ 主键值  │
├─────────────┼────────┤
│ 'Alice'     │ 1      │  ──→ 回表查主键索引获取完整数据
│ 'Bob'       │ 3      │
│ 'Carol'     │ 7      │
└─────────────┴────────┘
```

### 性能影响

#### 优势
- **主键查询极快**：可以直接在索引叶子节点获取完整数据，只需一次索引查找
- **范围查询快**：数据按主键顺序物理存储，范围扫描时顺序I/O

#### 劣势（面试重点）
- **二级索引查询需回表**：先查二级索引找到主键，再查主键索引获取数据（随机I/O）
- **插入操作依赖主键顺序**：如果主键非自增，插入可能触发页分裂（Page Split），导致随机写入
- **主键过长影响所有二级索引**：每个二级索引都存储主键值，主键过大会浪费大量存储空间

### 最佳实践

1. **主键使用自增ID或趋势递增数值**
   - 避免页分裂，保证顺序写入
   - 推荐使用 `BIGINT UNSIGNED AUTO_INCREMENT`

2. **主键不宜过长**
   ```sql
   -- ❌ UUID作为主键（字符串，无序，插入性能差）
   PRIMARY KEY (`id`)  -- id 是 VARCHAR(36)

   -- ✅ 雪花ID或自增整数
   PRIMARY KEY (`id`)  -- id 是 BIGINT
   ```

3. **覆盖索引优化**：将需要查询的字段包含在二级索引中，避免回表
   ```sql
   -- 只需查 name 和 age，不需要回表
   CREATE INDEX idx_name_age ON users (name, age);

   SELECT name, age FROM users WHERE name = 'Alice';
   -- Extra: Using index（覆盖索引，无需回表）
   ```

---

## 题目四：如何设计一个高效的数据库缓存策略？Redis 与 MySQL 如何配合使用？

**答案：**

### 缓存策略核心问题

1. **缓存什么？** — 读多写少、一致性要求不高的数据
2. **如何缓存？** — Cache Aside / Read Through / Write Through
3. **如何保证双写一致性？** — 缓存与数据库同步问题
4. **如何处理缓存雪崩/穿透/击穿？** — 三个经典问题

### 一、缓存模式

#### Cache Aside（最常用）

```python
# 读
def get_user(user_id):
    cache_key = f"user:{user_id}"
    user = redis.get(cache_key)
    if user:
        return json.loads(user)
    user = db.query("SELECT * FROM users WHERE id = %s", user_id)
    if user:
        redis.setex(cache_key, 3600, json.dumps(user))  # TTL=1小时
    return user

# 写
def update_user(user_id, data):
    db.execute("UPDATE users SET ... WHERE id = %s", user_id)
    redis.delete(f"user:{user_id}")  # 删除缓存，让下次读重建
```

#### Write Through / Write Behind
- Write Through：写数据库时同步写缓存（强一致，但写性能差）
- Write Behind：写数据库异步写缓存（性能高，但有数据丢失风险）

### 二、双写一致性解决方案

#### 方案1：延迟双删（Cache Aside 补充）

```python
# 写操作
def update_user(user_id, data):
    db.execute("UPDATE users SET ... WHERE id = %s", user_id)
    redis.delete(f"user:{user_id}")  # 1. 先删缓存

    import time
    time.sleep(0.1)  # 2. 延迟N秒（等待DB写完成后）
    
    redis.delete(f"user:{user_id}")  # 3. 再次删除（删除脏数据）
```

#### 方案2：基于消息队列的最终一致性

```
1. 业务系统写入DB
2. 发送消息到MQ
3. 消费服务读取DB最新状态，更新缓存
```

#### 方案3：订阅 MySQL binlog（最优解）

使用 **Canal**（阿里的MySQL binlog增量订阅组件）或 **Debezium**：
- 模拟MySQL从库，解析binlog
- 数据变更自动同步到Redis
- 延迟毫秒级，无需业务代码改造

### 三、三个经典问题及解决方案

#### 1. 缓存雪崩（Cache Avalanche）
**问题**：大量缓存同时过期 / Redis宕机，导致数据库瞬时压力过大。

**解决方案：**
```python
# 过期时间加随机值
ttl = 3600 + random.randint(0, 300)

# 多级缓存：本地缓存 + Redis + DB
# Redis宕机时降级到本地缓存

# Redis持久化 + 主从：RDB/AOF保证恢复
```

#### 2. 缓存穿透（Cache Penetration）
**问题**：查询不存在的数据，每次都穿透到DB。

**解决方案：**
```python
# 布隆过滤器（Bloom Filter）
# 将所有存在的key加入布隆过滤器
# 查之前先判断是否可能存在，不存在直接返回

# 缓存空值（TTL短一些）
if user is None:
    redis.setex(cache_key, 60, "NULL")  # 空值缓存60秒
```

#### 3. 缓存击穿（Cache Breakdown）
**问题**：热点key过期瞬间，大量并发请求击穿到DB。

**解决方案：**
```python
# 互斥锁（Redis SETNX）
lock = redis.setnx("lock:user:1", "1")
if lock:
    user = db.query(...)
    redis.setex("user:1", 3600, json.dumps(user))
    redis.delete("lock:user:1")
else:
    time.sleep(0.1)
    return get_user(user_id)  # 重试

# 无锁方案：热点key永不过期 + 异步重建
# 或使用Redis的 Lua脚本保证原子性
```

### 四、缓存设计规范

| 维度 | 规范 |
|------|------|
| **Key设计** | `{业务}:{表名}:{id}:{字段}`，如 `shop:user:123:profile` |
| **Value大小** | 单个 value 控制在 10KB 以内 |
| **过期策略** | 读接近写的数据 TTL 短，历史数据 TTL 长 |
| **容量规划** | 预留 Redis 内存的 70%，留足 Buffer |
| **监控指标** | hit rate、内存使用、命令延迟 |

---

## 题目五：PostgreSQL 的 MVCC 机制是什么？它如何实现事务隔离？这与 MySQL 有何不同？

**答案：**

### MVCC 基本概念

MVCC（Multi-Version Concurrency Control，多版本并发控制）是数据库管理系统的一种并发控制机制。

核心思想：**每个事务读取的是数据库在某个时间点的一致性快照（Snapshot），而不是等待锁释放。**

写入不会阻塞读取，读取也不会阻塞写入，大大提高了并发性能。

### PostgreSQL 的 MVCC 实现

PostgreSQL 使用 **Append-Only** 存储，数据更新时不覆盖旧值，而是追加新版本。

#### 关键字段（每行版本 / Tuple）

```
┌────────────────────────────────────────────────┐
│ xmin  │ xmax  │ t_ctid │ 字段值  │ cmin │ cmax  │
├────────────────────────────────────────────────┤
│ 100   │ -     │ (0,1)  │ Alice  │ 0    │ 0     │
└────────────────────────────────────────────────┘
```
- **xmin**：创建此行版本的事务ID（Transaction ID）
- **xmax**：删除/更新此行版本的事务ID（如果被删除或更新）
- **t_ctid**：指向最新版本行位置的指针（用于链接版本链）
- **cmin/cmax**：同一事务内的命令序列号（Command ID）

#### 事务可见性判断规则

PostgreSQL 通过 `visibility.c` 中的规则判断某行版本对当前事务是否可见：

```
规则（简化）：
1. 如果 xmin 对应的事务状态是 ABORTED → 不可见
2. 如果 xmin 对应的事务状态是 IN_PROGRESS 且当前事务不是该事务本身 → 不可见
3. 如果 xmin == 当前事务ID → 可见（当前事务自己插入的）
4. 如果 xmax 有值且对应事务已提交 → 该行版本已删除，不可继续使用
5. 如果行版本的xmax == 当前事务ID → 该删除对当前事务不可见
```

#### VACUUM 机制

旧版本积累后需要清理，PostgreSQL 通过 **VACUUM** 进程：
- 标记已删除/已更新的行为可用空间（但不立即物理删除）
- 更新自由空间映射表（FSM）
- 防止事务ID回卷（Transaction ID Wraparound）

```sql
-- 手动执行VACUUM
VACUUM orders;

-- 自动VACUUM（默认开启）
-- 相关参数：autovacuum_vacuum_threshold = 50
--           autovacuum_vacuum_scale_factor = 0.1
```

### 事务隔离级别（PostgreSQL）

PostgreSQL 支持的隔离级别（严格程度递增）：
1. **Read Uncommitted** — 实际行为同 Read Committed
2. **Read Committed**（默认）— 每次语句执行时生成新快照
3. **Repeatable Read** — 事务开始时生成快照，使用 `Postgres期末会出错的经典问题：幻读`
4. **Serializable** — 完全串行化，最严格，开销最大

```sql
-- 设置隔离级别
SET TRANSACTION ISOLATION LEVEL SERIALIZABLE;
BEGIN;
SELECT ...;
COMMIT;
```

### MySQL 与 PostgreSQL MVCC 对比

| 维度 | PostgreSQL | MySQL (InnoDB) |
|------|------------|-----------------|
| **实现方式** | Append-Only（旧版本保留，新版本追加） | Undo Log（回滚段） |
| **可见性判断** | 基于事务ID（xmin/xmax） | 基于事务ID + Undo Rollback Pointer |
| **版本存储位置** | 表数据本身存储多版本 | 在Undo表空间中存储旧版本，InnoDB行数据只存最新 |
| **垃圾回收** | VACUUM 进程 | InnoDB通过purge线程清理 |
| **读写并发** | 读写互不阻塞，并发高 | 读写互不阻塞（通过MVCC实现快照读） |
| **默认隔离级别** | Read Committed | REPEATABLE READ |
| **Serializable实现** | 真串行化（SERIALIZABLE） | 实际基于SS2PL（两阶段锁），不是真正的MVCC串行化 |

### 面试追问

> PostgreSQL 的 VACUUM 有什么参数可以优化？

```sql
-- 相关配置参数（postgresql.conf）
autovacuum = on                    -- 开启自动VACUUM
autovacuum_vacuum_threshold = 50   -- 行数阈值
autovacuum_vacuum_scale_factor = 0.1  -- 表行数的百分比
autovacuum_max_workers = 4          -- 并行VACUUM进程数

-- 大表建议手动执行VACUUM（避免自动VACUUM与业务抢资源）
VACUUM ANALYZE orders;

-- 监控VACUUM状态
SELECT relname, n_dead_tup, n_live_tup, last_vacuum
FROM pg_stat_user_tables
WHERE n_dead_tup > 10000;
```

---

> 📌 更多面试题欢迎 Star 本仓库：[https://github.com/qq286158530/interview-questions](https://github.com/qq286158530/interview-questions)