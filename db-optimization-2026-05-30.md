# 数据库性能优化面试题

> 📅 更新时间：2026-05-30
> 🏷️ 标签：MySQL | PostgreSQL | 性能优化 | 索引 | 查询优化

---

## 题目一：MySQL InnoDB 索引失效的常见场景有哪些？

### ✅ 答案

InnoDB 索引失效的常见场景包括：

**1. 违背最左前缀原则**
复合索引 `(a, b, c)` 中，若跳过 `a` 直接用 `b`，或跳过 `a、b` 直接用 `c`，索引失效。

**2. 使用函数或运算**
```sql
-- 索引失效
WHERE YEAR(create_time) = 2026
WHERE id + 1 = 10

-- 正确写法
WHERE create_time >= '2026-01-01' AND create_time < '2027-01-01'
WHERE id = 9
```

**3. 类型转换**
```sql
-- phone 是 varchar，传入数字 13800138000
-- MySQL 隐式将文字转为数字，全表扫描
WHERE phone = 13800138000  -- 索引失效
WHERE phone = '13800138000' -- 正常使用索引
```

**4. LIKE 模糊匹配以通配符开头**
```sql
WHERE name LIKE '%张%'   -- 全表扫描
WHERE name LIKE '张%'    -- 可以使用索引
```

**5. 使用 OR 连接不同列**
```sql
-- or 两边有一个不在索引中，整个查询索引失效
WHERE a = 1 OR b = 2  -- 除非 a、b 都有独立索引
```

**6. 范围查询右侧列失效**
```sql
-- 复合索引 (a, b, c)
WHERE a = 1 AND b > 10 AND c = 3  -- c 的索引失效
```

**7. 使用 NOT NULL / IS NOT NULL**
```sql
-- 如果列上允许 NULL，IS NOT NULL 可能不走索引
WHERE email IS NOT NULL
```

**排查方法：** 使用 `EXPLAIN` 查看 `type` 列，ALL 表示全表扫描，`range` 是正常范围扫描。

---

## 题目二：如何排查和优化慢查询（Slow Query）？

### ✅ 答案

**第一步：开启慢查询日志**

```sql
-- 查看是否开启
SHOW VARIABLES LIKE 'slow_query_log%';
SHOW VARIABLES LIKE 'long_query_time%';

-- 开启（临时）
SET GLOBAL slow_query_log = 'ON';
SET GLOBAL long_query_time = 1; -- 超过1秒记录

-- 永久开启：在 my.cnf 中配置
slow_query_log = 1
slow_query_log_file = /var/lib/mysql/slow.log
long_query_time = 1
```

**第二步：分析慢查询日志**

使用 `mysqldumpslow` 工具：
```bash
mysqldumpslow -t 5 -s at /var/lib/mysql/slow.log    # 查看前5条最慢的查询
mysqldumpslow -t 5 -s c /var/lib/mysql/slow.log    # 按次数排序前5
```

使用 `pt-query-digest`（Percona Toolkit）：
```bash
pt-query-digest --explain h=localhost,u=root,p=xxx /var/lib/mysql/slow.log
```

**第三步：使用 EXPLAIN 分析执行计划**

```sql
EXPLAIN SELECT * FROM orders WHERE status = 1;
EXPLAIN ANALYZE SELECT * FROM orders WHERE status = 1;  -- MySQL 8.0+
```

重点关注字段：
- `type`: `const > eq_ref > ref > range > index > ALL`（后者需优化）
- `key`: 实际使用的索引
- `rows`: 扫描行数，越少越好
- `Extra`: `Using filesort`、`Using temporary` 需重点优化

**第四步：优化手段**

| 场景 | 优化方案 |
|------|---------|
| 无索引 | 添加合适索引 |
| 索引不合理 | 调整索引列顺序，覆盖索引 |
| 深分页 | 改用游标分页：`WHERE id > last_id LIMIT 20` |
| 深分页 | 延迟关联：`SELECT * FROM orders o INNER JOIN (SELECT id FROM orders LIMIT 100000,20) t ON o.id=t.id` |
| 全表扫描 | 拆分为小查询 + 并行 |
| 单表过大 | 分库分表、冷热分离 |

---

## 题目三：MySQL 与 PostgreSQL 的存储引擎差异及性能特点？

### ✅ 答案

**MySQL（可插拔存储引擎）**

| 引擎 | 特点 | 适用场景 |
|------|------|---------|
| InnoDB | 行级锁，支持事务，外键，MVCC | OLTP，默认选择 |
| MyISAM | 表级锁，不支持事务 | 读多写少，只读表 |
| Memory | 内存存储，哈希索引 | 临时表，高速缓存 |

**PostgreSQL（单引擎，统一高并发）**

- 无存储引擎概念，PostgreSQL 服务端统一管理
- MVCC 实现：每行带有 `xmin/xmax` 版本号
- 支持多种索引类型： B-tree、Hash、GiST、GIN、BRIN、复合索引
- 适合复杂查询（子查询、CTE、窗口函数）

**性能对比关键点**

| 对比项 | MySQL InnoDB | PostgreSQL |
|--------|-------------|-----------|
| 并发模型 | MVCC + 行级锁 | MVCC + 多版本并发 |
| 写入性能 | 高（单表写入） | 写入时需维护 VACUUM |
| 复杂查询 | 较弱 | 强（优化器更智能） |
| 索引类型 | 主要 B-tree | B-tree + Hash + GiST + GIN + BRIN |
| 水平扩展 | 需第三方中间件 | 原生 FDW 跨库查询 |

**实际选择建议：**
- 金融、电商等强事务场景 → MySQL InnoDB
- 数据分析、GIS、JSON 存储 → PostgreSQL
- 大量简单读取 → MyISAM（历史场景）

---

## 题目四：数据库缓存策略有哪些？如何避免缓存雪崩、穿透、击穿？

### ✅ 答案

**常见缓存策略**

**1. Cache-Aside（旁路缓存）——最常用**
```
读：Cache → 未命中 → 读DB → 写Cache → 返回
写：更新DB → 删除Cache（不是更新Cache）
```

**2. Read-Through（读穿透）**
```
应用只读Cache，Cache负责从DB加载数据（应用无感知）
```

**3. Write-Through（写穿透）**
```
同时写DB和Cache，任一失败则全部回滚
```

**4. Write-Behind（写回）**
```
只写Cache，异步写DB（速度快但有数据丢失风险）
```

**三大缓存问题及解决方案**

**❄️ 雪崩：大量缓存同时过期，导致DB压力骤增**

```
原因：缓存服务宕机 / 大量key同时过期
```

解决方案：
- 过期时间 + 随机值：`TTL = base + random(0, 300)`
- 互斥锁：只允许一个请求回源
- 永不过期 + 异步更新
- Redis 集群高可用

**🕳️ 穿透：查询不存在的数据，Cache和DB都没有**

```
原因：恶意请求 / 非正常业务数据（如id=-1）
```

解决方案：
- 布隆过滤器（Bloom Filter）
- 空值缓存：`SET key NULL_VALUE EX 60`
- 严格参数校验

**⚡ 击穿：热点 key 突然过期，大量请求击穿到 DB**

```
原因：某个热点 key 突然过期
```

解决方案：
- 热点 key 不过期 + 异步更新
- 互斥锁：只有一个请求回源更新缓存
- 逻辑过期时间（不实际过期，异步更新）

**Redis 实战配置**
```bash
# 过期时间分散，避免同时过期
SET product:detail:{id} {json} EX 600 + random(0,300)

# 互斥锁（Redisson）
RLock lock = redisson.getLock("product:detail:" + id);
lock.lock(10, TimeUnit.SECONDS);
```

---

## 题目五：如何设计一个高效的数据库分页方案？

### ✅ 答案

**深分页问题（OFFSET 过大）**

```sql
-- 问题：MySQL 先扫描前100000行，再返回20行，极慢
SELECT * FROM orders ORDER BY id LIMIT 100000, 20;

-- EXPLAIN 结果 rows=100020，成本极高
```

**优化方案一：游标分页（推荐）**

```sql
-- 第一次查询
SELECT * FROM orders ORDER BY id LIMIT 20;

-- 下一页：记住上一页最后一条的 id
SELECT * FROM orders
WHERE id > :last_id
ORDER BY id LIMIT 20;
-- 利用主键索引 O(1) 定位，时间复杂度恒定
```

**优化方案二：延迟关联**

```sql
SELECT o.* FROM orders o
INNER JOIN (
    SELECT id FROM orders ORDER BY id LIMIT 100000, 20
) t ON o.id = t.id;
-- 先用索引定位 id，再关联回原表取数据
```

**优化方案三：覆盖索引 + 记录位置**

```sql
-- 子查询利用覆盖索引快速定位 id 列表
SELECT id, create_time FROM orders
WHERE user_id = 100
ORDER BY id
LIMIT 100000, 20;

-- 应用层拿到 last_id 后再查详情
SELECT * FROM orders WHERE id IN (:ids);
```

**大数据量分表策略**

| 策略 | 原理 | 适用场景 |
|------|------|---------|
| 范围分表 | 按 id 或时间 range 分表 | 流水记录 |
| 哈希分表 | user_id % N 分表 | 用户数据 |
| 搜索型 | ES + DB 混合 | 复杂条件检索 |

**PostgreSQL 的优化：利用 Keyset**

```sql
-- Keyset 分页（比 OFFSET 高效）
SELECT * FROM orders
WHERE (create_time, id) > ('2026-01-01', 12345)
ORDER BY create_time, id
LIMIT 20;
```

**性能对比**

| 方案 | 100页 | 1000页 | 10000页 |
|------|-------|--------|---------|
| LIMIT offset | 50ms | 500ms | 5000ms |
| 游标分页 | 1ms | 1ms | 1ms |
| 延迟关联 | 30ms | 30ms | 30ms |

> 🎯 核心原则：避免大 OFFSET，利用索引 + 主键有序性做游标，复杂度从 O(N) 降到 O(1)。

---

*整理自：牛客网、LeetCode、知乎、掘金等技术社区*