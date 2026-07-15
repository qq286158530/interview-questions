# 数据库性能优化面试题 — 2026-07-15

> 本次收录 5 道高质量数据库性能优化面试题，涵盖 MySQL 与 PostgreSQL 核心知识点。

---

## 题目 1：MySQL InnoDB 索引失效的常见场景有哪些？

**参考答案：**

InnoDB 索引失效是面试高频考点，以下几种场景最常见：

| 场景 | 说明 |
|------|------|
| **函数/运算** | `WHERE YEAR(create_time) = 2026` 对索引列使用函数，导致全表扫描 |
| **类型转换** | `WHERE phone = 13800138000`（phone 为 varchar），隐式类型转换使索引失效 |
| **左模糊匹配** | `LIKE '%keyword'` 或 `LIKE '%keyword%'` 无法使用 B+Tree 索引；`LIKE 'keyword%'` 可以 |
| **OR 条件** | `WHERE col_a = 1 OR col_b = 2`，若 col_a/col_b 分别有索引但无复合索引，可能只用到部分索引 |
| **不等于判断** | `!=` / `<>` 对索引列使用不等于，MySQL 优化器常放弃索引走向全表 |
| **NOT NULL 判断** | 对允许 NULL 的列使用 `IS NULL`，部分场景优化器选择全表 |
| **复合索引违反最左前缀** | `INDEX(a, b, c)` 查询 `WHERE b = 1` 或 `WHERE c = 1` 不走索引 |
| **数据量过小** | 优化器认为全表扫描更快时，即使列有索引也不使用（可通过 `FORCE INDEX` 强制） |

**优化建议：**
- 尽量避免在索引列上使用函数，考虑生成列（Generated Column）索引
- 使用显式类型匹配，避免隐式转换
- 复合索引严格遵循最左前缀原则
- 用 `EXPLAIN` 分析执行计划，确认索引是否被正确使用

> 📎 来源：[MySQL 官方文档 — How MySQL Uses Indexes](https://dev.mysql.com/doc/refman/8.0/en/mysql-indexes.html)

---

## 题目 2：如何定位并优化一条慢 SQL？请描述完整流程。

**参考答案：**

**Step 1：定位慢查询**
```sql
-- 开启慢查询日志（临时）
SET GLOBAL slow_query_log = 'ON';
SET GLOBAL long_query_time = 1; -- 超过1秒记录
SET GLOBAL slow_query_log_file = '/var/lib/mysql/slow.log';

-- 查看当前会话
SHOW VARIABLES LIKE 'slow_query%';
```
生产环境推荐使用 `performance_schema` 或 DBA 平台（如 ClickHouse、Sentry）持续采集。

**Step 2：分析执行计划**
```sql
EXPLAIN FORMAT=JSON SELECT ...;
-- 或 MySQL 8.0+可视化 EXPLAIN
EXPLAIN ANALYZE SELECT ...;  -- 包含实际执行时间
```
关注：
- `type`：const > eq_ref > ref > range > index > ALL（全表扫描）
- `key`：实际使用的索引
- `rows`：扫描行数
- `Extra`：Using filesort、Using temporary 等警告

**Step 3：针对性优化**
- **添加合适索引**：`CREATE INDEX idx_col ON table(col);`
- **避免 SELECT \***，只查必要字段
- **拆分复杂查询**：大表分页用延迟关联
- **覆盖索引**：让查询所有字段都在索引树中完成，避免回表
- **改写 SQL**：用 JOIN 替代子查询（MySQL 对子查询优化较弱）

**Step 4：验证效果**
用 `SHOW GLOBAL STATUS LIKE 'Questions';` 对比优化前后 QPS，用 `EXPLAIN` 确认索引被正确使用。

> 📎 来源：[Percona — How to analyze MySQL slow query log](https://www.percona.com/blog/how-to-analyze-mysql-slow-query-log/)

---

## 题目 3：MySQL InnoDB 与 PostgreSQL 的 MVCC 机制有什么区别？

**参考答案：**

两者都使用 MVCC（多版本并发控制）实现事务隔离，但实现细节差异显著：

### MySQL InnoDB
- **实现方式**：每行数据隐藏两列 — `DB_TRX_ID`（事务ID）和 `DB_ROLL_PTR`（回滚指针）
- **版本链**： UPDATE 时将旧行放入 **Undo Log**，新行指向旧版本形成链式结构
- **读取方式**：Read Committed 用 **当前快照 + undo log 回滚未提交**；Repeatable Read 用 **事务开始时的快照**
- **一致性非锁定读**：普通 SELECT 不用加锁，通过 undo 构建历史版本
- **问题**：Repeatable Read 下仍可能出现 **幻读**（InnoDB 通过 Next-Key Lock 解决）

### PostgreSQL
- **实现方式**：每行有 `xmin`（创建事务ID）和 `xmax`（删除/更新事务ID）
- **版本存储**：旧版本存在 **表堆（Table Heap）** 中，通过 `VACUUM` 清理死亡元组
- **事务快照**：`TransactionId` 数组判断可见性，Snapshot 包含 `xmin/xmax/catalog`
- **VACUUM 机制**：后台进程清理过期行版本，避免膨胀
- **Index-Only Scan**：索引仅存储索引列，需 VACUUM 后才能真正避免堆表访问

### 核心对比

| 特性 | InnoDB | PostgreSQL |
|------|--------|------------|
| 旧版本存储 | Undo Log（独立区域） | 表堆本身（原地保留旧版本） |
| GC 机制 | Purge 线程 | VACUUM 后台进程 |
| 索引结构 | B+Tree | B-Tree（默认）+ 多种索引类型 |
| 事务快照粒度 | 事务级别 | Session 级别 |
| 幻读防护 | Next-Key Lock | SI/SSI（快照隔离+谓词锁） |

> 📎 来源：[MySQL 8.0 Reference — InnoDB Multi-Versioning](https://dev.mysql.com/doc/refman/8.0/en/innodb-multi-versioning.html)  
> 📎 来源：[PostgreSQL Documentation — MVCC](https://www.postgresql.org/docs/current/mvcc.html)

---

## 题目 4：Redis 缓存与 MySQL 如何配合使用？缓存穿透、击穿、雪崩如何处理？

**参考答案：**

### 常见读写模式

**Cache-Aside（旁路缓存，最常用）：**
```
读：先查缓存 → 缓存未命中 → 查DB → 写入缓存 → 返回
写：更新DB → 删除缓存（而非更新缓存）
```

**为什么要删除而不是更新？**  
并发写入时，"更新缓存"可能导致脏数据；删除后下次读自然填充正确值。

---

### 缓存穿透（查询不存在的数据）

**问题**：大量请求查询 DB 和缓存都不存在的 key（如恶意刷接口），绕过缓存直击 DB。

**解决方案：**
1. **布隆过滤器（BloomFilter）**：在缓存层前加布隆过滤器，存在则查 DB，不存在直接返回 null
2. **缓存空值**：DB 返回 null 也写入缓存，设置短 TTL（如 60s）
3. **参数校验**：做好接口防护，过滤非法 id

---

### 缓存击穿（热点 key 过期瞬间大量涌入）

**问题**：某个热点 key 过期瞬间，所有请求同时穿透到 DB。

**解决方案：**
1. **互斥锁/分布式锁**：SETNX + TTL，只允许一个线程重新加载 DB
2. **热点数据永不过期**：用逻辑过期代替物理过期（后台异步更新）
3. **预热**：系统启动或低峰期提前加载热点 key

---

### 缓存雪崩（大量 key 集中过期或 Redis 宕机）

**问题**：大量 key 同一时刻过期，或 Redis 集群崩溃，导致 DB 压力剧增。

**解决方案：**
1. **过期时间随机化**：`TTL = base + random(0, 600)` 防止集中过期
2. **Redis 高可用**：哨兵（Sentinel）或集群（Cluster）保证可用性
3. **本地缓存兜底**：双层缓存，Redis 挂了用本地缓存（如 Caffeine）
4. **限流熔断**：Sentinel/Hystrix 限流，保护 DB
5. **提前演练**：做好 Redis 主从切换演练

> 📎 来源：[Redis 官方文档 — Redis + MySQL caching patterns](https://redis.io/docs/manualpatterns/caching/)

---

## 题目 5：PostgreSQL 中如何做分页查询优化？深分页问题如何解决？

**参考答案：**

### 普通 OFFSET 分页的问题

```sql
SELECT * FROM orders ORDER BY id LIMIT 20 OFFSET 100000;
-- OFFSET 越大，MySQL/PostgreSQL 都需扫描并丢弃前 OFFSET 行
-- 百万级 offset 会退化为全表扫描
```

### 优化方案一：游标分页（Keyset Pagination）

```sql
-- 第一页
SELECT * FROM orders ORDER BY id DESC LIMIT 20;

-- 下一页：基于上一页最后一条的 id
SELECT * FROM orders
  WHERE id < :last_seen_id   -- 上一页最小id
  ORDER BY id DESC
  LIMIT 20;
```

- **优点**：查询时间恒定为 O(1)，不受 OFFSET 影响
- **缺点**：不支持跳页，只能"下一页"体验
- **适用**：无限滚动、Feed 流、API 分页

### 优化方案二：延迟关联（Deferred Join）

```sql
-- 内层只查主键，外层再关联获取完整数据
SELECT o.* FROM orders o
  INNER JOIN (
    SELECT id FROM orders ORDER BY id LIMIT 20 OFFSET 100000
  ) t ON o.id = t.id;
```

- 内层子查询在索引列上完成（覆盖索引），速度快
- 外层关联只取 20 条回表，IO 大幅减少

### 优化方案三：使用表达式索引 + 分区

```sql
-- 对时间+状态建复合索引，减少回表
CREATE INDEX idx_orders_time_status ON orders (created_at, status);

-- 按时间分区，减少扫描范围
CREATE TABLE orders (
    id BIGSERIAL,
    ...
) PARTITION BY RANGE (created_at);
```

### 方案对比

| 方案 | 适用场景 | 缺点 |
|------|----------|------|
| OFFSET/LIMIT | 小数据量、需跳页 | 深分页 O(n) |
| 游标分页 | 无限滚动、Feed | 不支持跳页 |
| 延迟关联 | 中等深度分页 | 需主键有序 |
| 表达式索引 | 多条件分页 | 维护成本 |

> 📎 来源：[PostgreSQL Wiki — Keyset Pagination](https://wiki.postgresql.org/wiki/Keyset_pagination)  
> 📎 来源：[Lukas Fittl — Deep Pagination with PostgreSQL](https://www.fittl.com/posts/2018-04-18-deep-pagination-with-postgresql)

---

*本文件由 AI 面试题助手自动生成，每日更新*
*仓库地址：https://github.com/qq286158530/interview-questions*
