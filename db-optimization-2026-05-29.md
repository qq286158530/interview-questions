# 数据库性能优化面试题 · 2026-05-29

> 来源说明：以下题目综合整理自互联网公开技术文章、LeetCode 热题及面经回忆，来源链接附在每道题末尾。

---

## 题目一：索引失效的场景有哪些？如何避免？

### 参考答案

索引失效的常见场景：

1. **使用 `LIKE` 以通配符开头**：`LIKE '%abc'` 导致全表扫描，因为前缀不确定，B+ 树无法定位。
2. **使用 `OR` 连接条件**：若 OR 两边有一列无索引，则全表扫描。建议拆分为 `UNION ALL` 或确保每列都有索引。
3. **使用 `NOT IN` / `!=`**：MySQL 8.0 前无法利用索引，InnoDB 会退化为全表扫描。可用 `NOT EXISTS` 或改写为 `id NOT IN (SELECT id ...)` 加覆盖索引。
4. **类型转换**：索引列参与运算或类型不匹配，如 `WHERE name = 123`（name 为 VARCHAR），导致隐式类型转换，索引失效。
5. **最左前缀原则被破坏**：创建了复合索引 `(a, b, c)` 但查询只 WHERE b = ?，索引无法使用。
6. **使用 `IS NULL` / `IS NOT NULL`**：MySQL 5.7 前版本中，若 NULL 值较多，优化器可能跳过索引。
7. **WHERE 子句中使用表达式或函数**：如 `WHERE YEAR(create_time) = 2025`，索引列参与函数计算无法使用 B+ 树。
8. **数据量过小**：MySQL 优化器认为全表扫描更快，会放弃索引。

**如何避免：**
- 查询尽量使用最左前缀，复合索引顺序与查询字段顺序一致。
- 避免在索引列上使用函数、运算、类型转换。
- 将 OR 改为 UNION ALL 或 JOIN。
- 使用 `EXPLAIN` 分析执行计划，确认索引是否被使用。

📎 来源：https://xiaoluocode.blog.csdn.net/article/details/125917891

---

## 题目二：MySQL 中 InnoDB 和 MyISAM 的核心区别是什么？如何选型？

### 参考答案

| 特性 | InnoDB | MyISAM |
|------|--------|--------|
| **事务支持** | 支持 ACID 事务（Commit/Rollback） | 不支持事务 |
| **锁粒度** | 行级锁（Row Lock），适合高并发 | 表级锁（Table Lock） |
| **外键约束** | 支持外键 | 不支持外键 |
| **崩溃恢复** | 支持 MVCC + redo log，崩溃安全 | 不支持，损坏后较难恢复 |
| **全文索引** | MySQL 5.6+ 支持，但性能一般 | 原生支持全文索引 |
| **存储结构** | 共享表空间（可配置独立表空间） | 每个表独立 .frm + .MYD + .MYI |
| **COUNT(*)** | 需扫描全表（无全局计数器） | 有专门计数器，快 |
| **适用场景** | 核心业务、高并发、需事务 | 非核心日志、只读报表、全文搜索 |

**选型建议：**
- 几乎所有新项目默认选 InnoDB，除非有特殊需求。
- 大量 `SELECT COUNT(*)` 且无事务需求 → MyISAM 可考虑。
- 需要全文搜索但数据量不大 → 可用 InnoDB + 全文索引；数据量大建议上 Elasticsearch。
- 极高并发写入且无事务需求 → MyISAM 表级锁反而减少锁竞争。

📎 来源：https://blog.csdn.net/qq_38238203/article/details/125893247

---

## 题目三：一条 SQL 执行很慢，如何进行性能分析和优化？

### 参考答案

**第一步：判断是偶发还是持续**

```sql
-- 查看当前连接和耗时查询
SHOW PROCESSLIST;
-- 查看锁等待
SELECT * FROM information_schema.INNODB_LOCK_WAITS;
```

**第二步：EXPLAIN 分析执行计划**

```sql
EXPLAIN SELECT ...;
-- 关键字段：
-- type: ALL=全表扫描, ref/range=索引命中
-- key: 实际使用的索引
-- rows: 扫描行数估算
-- Extra: Using filesort/Using temporary = 需优化
```

**第三步：常见优化手段**

| 问题 | 优化方案 |
|------|----------|
| 全表扫描 | 加上合适索引 |
| Using filesort | 优化 ORDER BY，或改用索引排序 |
| Using temporary | 优化 GROUP BY，添加合适索引 |
| 索引列参与运算 | 改写为前置计算，或应用层处理 |
| 深度分页 | 改用 `WHERE id > last_id LIMIT N` 游标分页 |
| JOIN 过多 | 分解为单表查询或加小表驱动 |
| 锁等待 | 缩短事务，将行锁转为表锁语句延迟执行 |

**第四步：上线预防**
- 开启慢查询日志 `slow_query_log = 1`，定期分析 `mysqldumpslow`。
- 使用 `pt-query-digest`（Percona Toolkit）深度分析。
- 监控 `QEP`（Query Execution Plan）变化，回归测试确认优化效果。

📎 来源：https://www.cnblogs.com/shangcloud/p/17849566.html

---

## 题目四：PostgreSQL 中如何优化大表的全表扫描和分页查询？

### 参考答案

**问题背景：** 大表（千万级）分页查询 `OFFSET N LIMIT M`，OFFSET 越大越慢，因为数据库仍需扫描全部跳过行。

**优化方案一：游标分页（Keyset Pagination）**

```sql
-- 传统方式（慢）
SELECT * FROM orders ORDER BY id OFFSET 10000 LIMIT 20;

-- 优化方式：基于上一页最后一条的 id 继续查询
SELECT * FROM orders
WHERE id > :last_id
ORDER BY id
LIMIT 20;
-- 利用 B+ 树顺序，时间复杂度从 O(N) 降为 O(log N + M)
```

**优化方案二：为分页字段建索引**

```sql
-- 为排序字段建 B-tree 索引
CREATE INDEX idx_orders_id ON orders(id);
-- 复合索引：排序 + 过滤条件
CREATE INDEX idx_orders_status_id ON orders(status, id);
```

**优化方案三：EXPLAIN ANALYZE 定位**

```sql
EXPLAIN (ANALYZE, BUFFERS, FORMAT TEXT)
SELECT * FROM orders WHERE status = 'completed' LIMIT 20;
-- 查看 actual rows、actual time、buffers hit/read
```

**优化方案四：分区表（Table Partitioning）**

```sql
-- 按时间范围分区，减少扫描量
CREATE TABLE orders (
    id BIGSERIAL,
    created_at TIMESTAMP,
    status VARCHAR(20)
) PARTITION BY RANGE (created_at);

CREATE TABLE orders_2025 PARTITION OF orders
    FOR VALUES FROM ('2025-01-01') TO ('2026-01-01');
```

**优化方案五：并行扫描**

```sql
-- 开启并行执行（MySQL 8.0+ / PostgreSQL 默认支持）
SET max_parallel_workers_per_gather = 4;
-- PostgreSQL 查看并行计划
EXPLAIN SELECT * FROM orders WHERE status = 'pending';
```

📎 来源：https://www.postgresql.org/docs/current/performance-tips.html

---

## 题目五：如何设计一个高效的数据库缓存策略？Redis 与 MySQL 如何配合使用？

### 参考答案

**缓存策略设计原则：**

1. **Cache Aside（旁路缓存）— 最常用**
   - 读：先查 Redis，命中则返回；未命中则查 DB，写入 Redis，返回结果。
   - 写：先更新 DB，删除 Redis 缓存（下一次读取时再填充）。
   - 为什么是删除而不是更新？避免并发下缓存和数据不一致。

2. **Read Through / Write Through**：缓存作为 DB 和客户端之间的独立透明层，较少单独使用。

3. **Write Behind（Write Back）**：写操作直接写缓存，异步批量写 DB。适合高写入场景，但数据丢失风险大。

**具体实现（Cache Aside）：**

```python
def get_user(user_id):
    # 先查缓存
    cache_key = f"user:{user_id}"
    user = redis.get(cache_key)
    if user:
        return json.loads(user)
    
    # 缓存未命中，查数据库
    user = db.query("SELECT * FROM users WHERE id = %s", user_id)
    
    # 回填缓存，设置过期时间防止脏数据
    if user:
        redis.setex(cache_key, 3600, json.dumps(user))  # 1小时过期
    return user

def update_user(user_id, data):
    # 先更新数据库
    db.execute("UPDATE users SET ... WHERE id = %s", user_id)
    
    # 删除缓存（而非更新）
    redis.delete(f"user:{user_id}")
```

**缓存常见问题及解决方案：**

| 问题 | 解决方案 |
|------|----------|
| 缓存穿透 | 布隆过滤器（Bloom Filter）或缓存空值 |
| 缓存击穿 | 互斥锁（SETNX）或永不过期 + 异步重建 |
| 缓存雪崩 | 过期时间加随机偏移、Redis 集群高可用 |
| 数据不一致 | 延迟双删：先删缓存 → 更新DB → 延迟删缓存 |

**过期策略选择：**
- 数据频繁变化（库存、余额）：过期时间短（秒~分钟）或不用缓存。
- 数据读多写少（用户资料、商品详情）：过期时间可设长（30分钟~数小时）。

📎 来源：https://redis.io/docs/manual patterns/caching/

---

*整理自互联网公开技术资料，每题附带真实来源链接。*
*如发现内容问题，欢迎提交 Issue 或 PR 补充修订。*