# 数据库性能优化面试题 · 2026-06-08

> 涵盖 MySQL / PostgreSQL 核心知识点：索引优化、查询优化、存储引擎、缓存策略

---

## 题目一：MySQL 中索引失效的常见场景有哪些？如何避免？

### 答案

**索引失效的常见场景：**

1. **使用函数或运算**：对索引列使用 `YEAR(create_time)`、`LEFT(col, 3)` 等函数，或对其做 `+、-、*、/`运算，导致索引失效。
2. **类型转换**：当查询条件中字段类型与索引列类型不匹配时（如字符串列用数值比较），MySQL 会隐式转换，导致全表扫描。
3. **LIKE 前缀通配**：`LIKE '%keyword'` 以通配符开头，无法使用 B-Tree 索引。
4. **OR 连接**：在 WHERE 中使用 OR，且 OR 两边有一边不走索引时，整体退化为全表扫描。
5. **最左前缀原则违反**：在多列联合索引 `(a, b, c)` 中，跳过 `a` 直接用 `b` 或 `c` 会导致索引部分失效。
6. **使用 NOT、<>、!=**：导致索引无法使用。
7. **使用 IS NULL / IS NOT NULL**：在某些存储引擎下可能导致索引失效。
8. **统计信息不准确**：表数据量发生剧烈变化后，MySQL 优化器可能选错执行计划。

**避免策略：**
- 尽量使用索引列的原始形式，避免函数包装
- 确保类型一致，避免隐式转换
- 用 `LIKE 'keyword%'` 替代前缀通配
- 将 OR 改写为 UNION ALL避免索引失效
- 遵循最左前缀原则设计联合索引
- 定期执行 `ANALYZE TABLE` 更新统计信息
- 使用 `EXPLAIN` 分析执行计划，确认是否走索引

**来源：** [MySQL 官方文档 - Optimizing Queries](https://dev.mysql.com/doc/refman/8.0/en/optimization.html)

---

## 题目二：一条 SQL 语句的执行流程是怎样的？MySQL 优化器如何选择执行计划？

### 答案

**SQL 执行完整流程：**

```
SQL → 解析器(Parser) → 预处理器(Preprocessor) → 优化器(Optimizer)
     → 执行器(Executor) → 存储引擎 → 数据文件
```

1. **连接层**：连接线程接收 SQL，验证用户权限
2. **解析器**：词法/语法分析生成解析树（AST）
3. **预处理器**：检查表/列权限、别名解析
4. **优化器**：基于成本模型（CBO）选择最优执行计划
   - 主要决策：使用哪个索引、表的连接顺序、连接算法（NestLoop / HashJoin / MergeJoin）
5. **执行器**：调用存储引擎 API 获取数据
6. **存储引擎**：InnoDB/MyISAM 等实际读写数据

**优化器选择依据：**
- **成本模型**：估算 I/O 成本（读取页数）和 CPU 成本（行数）
- **统计信息**：表行数、数据分布直方图、索引基数（Cardinality）
- **Hint语法**：可通过 `USE INDEX`、`FORCE INDEX` 干预优化器决策

**执行计划查看：**
```sql
EXPLAIN ANALYZE SELECT * FROM orders WHERE status = 1;
```

**来源：** [MySQL 8.0 Reference Manual - Optimizer](https://dev.mysql.com/doc/refman/8.0/en/optimizer-index.html)

---

## 题目三：InnoDB 与 MyISAM 存储引擎的核心区别是什么？如何选择？

### 答案

| 特性 | InnoDB | MyISAM |
|------|--------|--------|
| **事务支持** | 支持 ACID事务 | 不支持事务 |
| **锁粒度** | 行级锁（并发好） | 表级锁 |
| **外键支持** | 支持 | 不支持 |
| **崩溃恢复** | 自动崩溃恢复（redo log） | 无，需手动修复 |
| **索引结构** | 聚簇索引（数据与索引在同一B+树） | 非聚簇索引（索引与数据分离） |
| **COUNT(*)** | 全表扫描 | 保存行数（快） |
| **适用场景** | 核心业务、高并发、需要事务 | 读多写少、不需要事务 |

**InnoDB 的优势：**
- 支持行级锁 + MVCC，并发性能强
- 聚簇索引设计，叶子节点直接存放数据，减少随机 I/O
- 支持崩溃自动恢复，数据安全性高
- 支持外键约束，保证引用完整性

**何时选 MyISAM：**
- 静态/只读数据（如日志表）
- 全文索引需求（InnoDB 5.6+ 也支持，但早期版本 MyISAM 更好）
- 确实不需要事务，且追求 `COUNT(*)` 性能

**建议：** MySQL 8.0+ 默认引擎已是 InnoDB，新项目无脑选 InnoDB。

**来源：** [MySQL 官方文档 - InnoDB vs MyISAM](https://dev.mysql.com/doc/refman/8.0/en/innodb-introduction.html)

---

## 题目四：PostgreSQL 中如何分析慢查询？有哪些关键参数和工具？

### 答案

**1. 开启慢查询日志：**
```sql
-- 在 postgresql.conf 中配置
log_min_duration_statement = 1000  -- 记录超过 1000ms 的查询
log_statement = 'all'               -- 记录所有语句（调试时）
log_lock_waits = on -- 记录锁等待
```

**2. 使用 EXPLAIN ANALYZE：**
```sql
EXPLAIN (ANALYZE, BUFFERS, FORMAT TEXT)
SELECT * FROM orders WHERE user_id = 12345;
```
- `ANALYZE`：实际执行并显示真实时间
- `BUFFERS`：显示缓存命中情况（Shared Hit/Read）
- 重点关注：**actual time**（实际耗时）、**rows**（实际扫描行数）、**Buffers Hit**（缓存命中率）

**3. pg_stat_statements 扩展（生产必备）：**
```sql
-- 开启扩展
CREATE EXTENSION pg_stat_statements;

-- 查询最慢的 Top 5 查询
SELECT query, calls, total_time, mean_time, rows
FROM pg_stat_statements
ORDER BY total_time DESC
LIMIT 5;

-- 查询缓存命中率最低的查询
SELECT query, shared_blks_hit, shared_blks_read,
       ROUND(100.0 * shared_blks_hit / NULLIF(shared_blks_hit + shared_blks_read, 0), 2) AS hit_ratio
FROM pg_stat_statements
ORDER BY hit_ratio ASC
LIMIT 5;
```

**4. 索引建议工具：`pg_stat_user_indexes`**：
```sql
SELECT schemaname, tablename, indexname, idx_scan, idx_tup_read, idx_tup_fetch
FROM pg_stat_user_indexes
ORDER BY idx_scan ASC;
```
扫描数为 0 的索引是"死索引"，可以删除。

**5. 关键调参：**
```ini
shared_buffers = 25% of RAM # 共享缓冲池，建议为系统内存的 25%
effective_cache_size = 75% of RAM  # 告诉优化器可用缓存大小
work_mem = 256MB # 排序/哈希join内存上限
maintenance_work_mem = 512MB       # 维护操作内存（VACUUM/重建索引）
random_page_cost = 1.1             # SSD 设置为 1.1，机械盘设为 4.0
```

**来源：** [PostgreSQL Documentation - Using EXPLAIN](https://www.postgresql.org/docs/current/using-explain.html)

---

## 题目五：数据库缓存策略有哪些？如何设计一个高效的读写缓存架构？

### 答案

**一、缓存策略分类：**

| 策略 | 描述 | 适用场景 |
|------|------|----------|
| **Cache-Aside** | 读：先读缓存，miss 再查DB写入缓存<br>写：先写DB，再删除缓存 | 读多写少（主流方案） |
| **Read-Through** | 缓存自动加载数据，应用不感知 | 缓存层透明场景 |
| **Write-Through** | 写操作同时写缓存和DB | 数据一致性要求高 |
| **Write-Behind** | 写操作先写缓存，再异步批量写DB | 写入性能要求极高 |
| **TTL过期** | 缓存设置过期时间，自然淘汰 | 非关键数据 |

**二、缓存经典问题：**

1. **缓存雪崩**：大量缓存同时过期 → 解决：过期时间加随机值（`TTL = base + rand(0, 300)`）
2. **缓存击穿**：热点 key 过期瞬间大量请求打到 DB → 解决：互斥锁（`SETNX`）或永不过期 + 异步重建
3. **缓存穿透**：查询不存在的数据 → 解决：布隆过滤器（BloomFilter）或缓存空值
4. **数据不一致**：DB 和缓存双写时产生不一致 → 解决：最终一致性 + 延迟双删

**三、高效缓存架构设计：**

```
[应用层] → [Redis Cluster] → [MySQL/PostgreSQL]
              ↓
         [本地缓存 Caffeine/Guava] → 热点数据本地缓存，减少 Redis 压力
```

**Redis 实战配置示例：**
```sql
-- 缓存穿透：存空值，过期时间短
SETEX "user:10086" 300 "NULL"   -- 空值缓存 5 分钟

-- 缓存击穿：互斥锁
SETNX "lock:user:10086" 1 EX10
-- 获取锁成功后查 DB，写入缓存，删除锁

-- 热点数据永不过期 + 异步更新
if (cache.get("user:10086") == null) {
    // 异步线程查询 DB
    threadPool.submit(() -> {
        data = db.query("SELECT * FROM users WHERE id = 10086");
        cache.set("user:10086", data); // 不设过期时间
    });
    return db.query("SELECT * FROM users WHERE id = 10086"); // 兜底
}
```

**来源：** [Redis 官方文档 - Cache Patterns](https://redis.io/docs/manualpatterns/cache/)

---

*整理自 MySQL 8.0 官方文档、PostgreSQL 16官方文档及业界经典面试题库*