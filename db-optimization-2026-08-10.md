# 数据库性能优化面试题

> 📅 整理日期：2026-08-10
> 📚 涵盖：MySQL / PostgreSQL 索引优化、查询优化、存储引擎、缓存策略、架构设计

---

## 题目 1：B+ 树索引在 MySQL 中是如何工作的？为什么比二叉树更适合？

### 参考答案

**B+ 树是一种多叉平衡查找树，是 MySQL InnoDB 存储引擎默认索引结构。**

**为什么适合数据库：**

1. **高扇出（Fan-out）**：每个节点可存储多个键值，树高通常只有 3~4 层，可以减少磁盘 IO 次数。一个非叶子节点可以容纳约 100~200 个子指针，100 万行数据最多只需 3~4 次磁盘寻道。
2. **磁盘预读友好**：利用磁盘页（通常 16KB）作为节点大小，一次 IO 可以读取整层节点，数据局部性好。
3. **所有数据都在叶子节点**（叶子节点之间通过双向链表连接），范围查询（如 `BETWEEN`、`ORDER BY`）只需遍历叶子链表，无需回溯。
4. **平衡性**：所有叶子节点深度相同，保证查询时间稳定 O(log n)。

**对比二叉树（红黑树）：** 二叉树每个节点只有两个子节点，树高远超 B+ 树，磁盘 IO 次数多 10~100 倍，极端情况下退化为链表。

**InnoDB 中 B+ 树的实际结构：**
- 根节点和中间节点只存储索引列和子页指针
- 叶子节点存储完整的索引列值 + 主键 ID（聚集索引）或索引列值 + 主键 ID（辅助索引）
- 辅助索引查询需要**回表**：先在辅助索引找到主键，再通过主键去聚集索引查找完整行

> 📎 来源：https://dev.mysql.com/doc/refman/8.0/en/innodb-index-types.html

---

## 题目 2：如何诊断和优化一条慢查询（slow query）？

### 参考答案

**步骤一：开启慢查询日志，定位慢 SQL**

```sql
-- 查看慢查询是否开启
SHOW VARIABLES LIKE 'slow_query%';
SHOW VARIABLES LIKE 'long_query_time';

-- 开启慢查询日志（临时）
SET GLOBAL slow_query_log = 'ON';
SET GLOBAL long_query_time = 1; -- 超过1秒记录

-- 查看日志文件路径
SHOW VARIABLES LIKE 'slow_query_log_file';
```

**步骤二：使用 EXPLAIN 分析执行计划**

```sql
EXPLAIN SELECT ...;
EXPLAIN ANALYZE SELECT ...;  -- MySQL 8.0+，会实际执行并返回真实成本
```

重点关注：
- `type`：反映访问类型，从好到差依次 `system > const > eq_ref > ref > range > index > ALL`（`ALL` 为全表扫描，必须优化）
- `key`：实际使用的索引
- `rows`：扫描行数，越少越好
- `Extra`：`Using filesort`、`Using temporary` 是需要优化的信号

**步骤三：针对性优化**

| 问题 | 解决方案 |
|------|----------|
| 全表扫描（type=ALL） | 添加合适索引，利用覆盖索引 |
| Using filesort | 优化 ORDER BY，为排序列建索引 |
| Using temporary | 优化 GROUP BY，尽量在索引中完成分组 |
| 索引失效 | 避免在索引列上使用函数、类型转换、左边 LIKE、OR |

**步骤四：使用 SHOW PROFILE / 性能 Schema**

```sql
SET profiling = 1;
SHOW PROFILES;
SHOW PROFILE FOR QUERY 1;  -- 查看各阶段耗时
```

**步骤五：查看表结构和索引情况**

```sql
SHOW TABLE STATUS FROM db LIKE 'tbl';
SHOW INDEX FROM tbl;
```

> 📎 来源：https://dev.mysql.com/doc/refman/8.0/en/using-explain.html

---

## 题目 3：MySQL InnoDB 和 MyISAM 存储引擎的核心区别是什么？如何选择？

### 参考答案

| 特性 | InnoDB | MyISAM |
|------|--------|--------|
| **事务支持** | 支持 ACID 事务，有 redo log / undo log | 不支持事务 |
| **锁粒度** | 行级锁 + 间隙锁，并发好 | 表级锁 |
| **外键** | 支持 | 不支持 |
| **崩溃恢复** | 自动通过 redo log 恢复 | 只能通过 myisamchk 修复，效率低 |
| **索引结构** | B+ 树（聚集索引）| B+ 树（非聚集索引）|
| **count(*) 性能** | 全表扫描，较慢 | 有内部计数器，很快 |
| **全文索引** | 5.6+ 支持 | 支持 |
| **存储空间** | 较大（支持行压缩）| 较小 |

**聚集索引 vs 非聚集索引：**
- **InnoDB**（聚集索引）：数据文件按主键顺序物理存储，辅助索引叶子节点存主键值
- **MyISAM**（非聚集索引）：数据文件和索引文件分离，索引叶子节点存数据文件指针

**如何选择：**

- ✅ 用 InnoDB：生产环境、需要事务、高并发、数据可靠性要求高
- ✅ 用 MyISAM：只读场景、静态表、全文搜索需求（历史遗留场景）

> 📎 来源：https://dev.mysql.com/doc/refman/8.0/en/innodb-storage-engine.html

---

## 题目 4：PostgreSQL 中如何分析和优化慢查询？和 MySQL 有什么区别？

### 参考答案

**第一步：开启 `pg_stat_statements` 插件（相当于 MySQL 的 performance schema）**

```sql
-- 在 postgresql.conf 中添加
-- shared_preload_libraries = 'pg_stat_statements'
-- pg_stat_statements.track = top

-- 开启插件
CREATE EXTENSION IF NOT EXISTS pg_stat_statements;

-- 查看最慢的 SQL
SELECT query, calls, total_exec_time, avg_exec_time, rows
FROM pg_stat_statements
ORDER BY total_exec_time DESC
LIMIT 10;
```

**第二步：使用 EXPLAIN ANALYZE 分析执行计划**

```sql
EXPLAIN (ANALYZE, BUFFERS, FORMAT TEXT) SELECT ...;
```

关键指标：
- `actual time`：实际执行时间（启动~结束）
- `rows`：估算行数 vs 实际行数差距大说明统计信息过期
- `Buffers: shared hit`：缓存命中数，`read` 表示磁盘 IO
- `Seq Scan on xxx`：全表扫描，如数据量大应建索引

**第三步：维护统计信息和索引**

```sql
-- 更新表统计信息（MySQL 的 ANALYZE TABLE 等价）
ANALYZE tbl;

-- 查看表膨胀情况
SELECT schemaname, tablename, pg_size_pretty(pg_total_relation_size(schemaname||'.'||tablename))
FROM pg_stat_user_tables
ORDER BY pg_total_relation_size(schemaname||'.'||tablename) DESC LIMIT 10;

-- 查看未使用的索引
SELECT indexrelname, idx_scan, idx_tup_read
FROM pg_stat_user_indexes
WHERE idx_scan = 0;
```

**MySQL vs PostgreSQL 优化对比：**

| 方面 | MySQL | PostgreSQL |
|------|-------|------------|
| 慢查询工具 | slow_query_log + EXPLAIN | pg_stat_statements + EXPLAIN ANALYZE |
| 成本模型 | 简单成本估算 | 更精确的成本模型（可调）|
| 索引类型 | B+ 树为主，部分 Hash | B+ 树、Hash、GiST、SP-GiST、GIN、BRIN 多种 |
| 并行查询 | 8.0+ 支持 | 原生支持并行 Seq Scan/Index Scan |
| 统计信息 | `ANALYZE` 更新直方图 | 自动收集 + 可手动 ANALYZE |
| 建议工具 | MySQL Tuner、Percona Toolkit | pgBadger、pgAdmin、pgBouncer |

> 📎 来源：https://www.postgresql.org/docs/current/using-explain.html

---

## 题目 5：数据库缓存策略——如何设计 Redis + MySQL 的缓存架构？缓存穿透、击穿、雪崩如何解决？

### 参考答案

**经典架构：Cache Aside（旁路缓存）**

```
读：先查 Redis → 命中则返回 → 未命中查 MySQL → 写回 Redis → 返回
写：先写 MySQL → 再删除 Redis（不是更新，防止脏数据）
```

**三个经典问题及解决方案：**

### 缓存穿透（查询不存在的数据）

**问题**：大量请求查询数据库和缓存中都不存在的 key，绕过缓存直接打穿数据库。

**解决方案：**
1. **布隆过滤器（Bloom Filter）**：将所有存在的 key 存入布隆过滤器，查询前先过滤
2. **缓存空值**：对不存在的 key 也缓存一个空值（TTL 短一些，如 60s）

```python
# 伪代码
def get_user(user_id):
    if not bloom_filter.might_contain(user_id):  # 布隆过滤器判断
        return None
    user = redis.get(f"user:{user_id}")
    if user:
        return json.loads(user)
    user = mysql.get(user_id)
    if user:
        redis.setex(f"user:{user_id}", 3600, json.dumps(user))
    else:
        redis.setex(f"user:{user_id}", 60, "NULL")  # 缓存空值
    return user
```

### 缓存击穿（热点 key 过期瞬间大量请求）

**问题**：某个热点 key 过期瞬间，大量并发请求同时穿透到数据库。

**解决方案：**
1. **互斥锁（mutex）**：只允许一个线程从数据库加载数据，其他线程等待
2. **永不过期**：逻辑过期（存储带过期时间戳），后台异步刷新

```python
# 使用 Redis SETNX 实现互斥锁
def get_user_mutex(user_id):
    key = f"user:{user_id}"
    lock_key = f"lock:{key}"
    user = redis.get(key)
    if user:
        return json.loads(user)
    # 尝试获取锁
    if redis.setnx(lock_key, "1"):
        try:
            user = mysql.get(user_id)
            redis.setex(key, 3600, json.dumps(user))
        finally:
            redis.delete(lock_key)
        return user
    else:
        time.sleep(0.1)
        return get_user_mutex(user_id)  # 重试
```

### 缓存雪崩（大量 key 同时过期）

**问题**：大量 key 在同一时间过期，导致大量请求同时穿透到数据库。

**解决方案：**
1. **过期时间加随机偏移**：`TTL = base_ttl + random(0, 300)`
2. **服务降级/限流**：数据库压力过大时返回默认值
3. **高可用架构**：Redis Cluster / 主从复制
4. **预热**：系统启动时主动加载热点数据到缓存

```python
# 设置过期时间时加随机偏移
ttl = 3600 + random.randint(0, 300)
redis.setex(key, ttl, value)
```

**补充：Redis 内存淘汰策略（当内存满时）**

| 策略 | 说明 |
|------|------|
| `noeviction` | 不淘汰，返回错误（默认）|
| `allkeys-lru` | 所有 key 中淘汰最近最少使用 |
| `volatile-lru` | 设置了过期时间的 key 中淘汰 LRU |
| `allkeys-random` | 所有 key 随机淘汰 |
| `volatile-ttl` | 淘汰最快过期的 key |

> 📎 来源：https://redis.io/docs/manual/scaling/

---

> 💡 更多面试题请访问：https://github.com/qq286158530/interview-questions
> 欢迎 Star 和 PR！
