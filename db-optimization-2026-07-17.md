# 数据库性能优化面试题 — 2026-07-17

> 本题集整理自网络公开技术资料，涵盖 MySQL 与 PostgreSQL 核心知识点，适合面试复习与能力巩固。

---

## 题目一：MySQL 中 InnoDB 索引失效的常见场景有哪些？

### 参考答案

InnoDB 索引失效（无法利用索引走 Range 回退到全表扫描）的常见场景：

1. **索引列参与计算或函数调用**
   ```sql
   -- 索引失效 ❌
   SELECT * FROM orders WHERE YEAR(create_time) = 2026;
   -- 改写为 ✅
   SELECT * FROM orders WHERE create_time >= '2026-01-01' AND create_time < '2027-01-01';
   ```

2. **隐式类型转换**
   当查询条件中字段类型与索引列类型不匹配时，MySQL 会自动做隐式转换，导致索引失效。例如 `user_id` 是 `INT` 但传入了字符串 `'123'`。

3. **使用 `LIKE` 前缀通配**
   ```sql
   -- 索引失效 ❌（%在前面）
   SELECT * FROM users WHERE name LIKE '%张%';
   -- 索引可用 ✅（%在后面）
   SELECT * FROM users WHERE name LIKE '张%';
   ```

4. **`OR` 连接了非索引列**
   ```sql
   -- 如果 status 没有索引，整个查询退化为全表扫描 ❌
   SELECT * FROM orders WHERE user_id = 1 OR status = 'paid';
   ```

5. **使用了 `NOT IN`、`NOT EXISTS`、`<> ` 等否定操作符**
   优化器倾向于放弃索引改为全表扫描。

6. **数据量过小或 `SELECT *` 覆盖不了索引列时优化器主动放弃索引**
   MySQL 优化器基于统计信息判断全表扫描成本更低时，会放弃索引。

> 📎 来源：MySQL 官方文档 — "How MySQL Uses Indexes"  
> https://dev.mysql.com/doc/refman/8.0/en/mysql-indexes.html

---

## 题目二：如何排查和解决 MySQL 慢查询？请描述完整流程。

### 参考答案

**Step 1：开启慢查询日志**
```ini
slow_query_log = 1
slow_query_log_file = /var/log/mysql/slow.log
long_query_time = 1  -- 超过 1 秒记录
```

**Step 2：分析慢查询日志**
使用 `mysqldumpslow` 工具汇总：
```bash
mysqldumpslow -t 5 /var/log/mysql/slow.log
```

**Step 3：使用 `EXPLAIN` 分析执行计划**
```sql
EXPLAIN SELECT * FROM orders WHERE user_id = 100;
```
重点关注：
- `type`：最优为 `const/eq_ref/ref`，最差为 `ALL`（全表扫描）
- `key`：实际使用的索引
- `rows`：扫描行数估算
- `Extra`：是否出现 `Using filesort`、`Using temporary`

**Step 4：针对性优化**
- 加索引：`ALTER TABLE orders ADD INDEX idx_user_id(user_id);`
- 改写 SQL：避免 `SELECT *`，减少覆盖列
- 拆解子查询：改用 `JOIN`
- 限制返回条数：`LIMIT`

**Step 5：使用 `SHOW STATUS`/`SHOW PROFILE` 进一步定位**
```sql
SET profiling = 1;
SHOW PROFILE FOR QUERY 1;
```

> 📎 来源：MySQL Performance Blog — "Analyzing Slow Queries"  
> https://www.mysqltutorial.org/mysql-slow-query-log/

---

## 题目三：PostgreSQL 中如何优化大表的分页查询（`OFFSET` 很大的场景）？

### 参考答案

当 `OFFSET` 值很大时（如 `OFFSET 100000 LIMIT 10`），PostgreSQL 仍然需要扫描并跳过前 10 万行，效率极低。

**优化方案一：游标分页（Keyset Pagination）**
```sql
-- 第一页
SELECT * FROM orders ORDER BY id LIMIT 10;

-- 下一页：基于上一页最后一条的 id 做过滤
SELECT * FROM orders
WHERE id > :last_id
ORDER BY id
LIMIT 10;
```
时间复杂度从 **O(n + m)** 降为 **O(m)**（n 为 OFFSET 值，m 为 LIMIT 值）。

**优化方案二：使用表达式索引覆盖排序**
```sql
-- 为高频排序字段建索引
CREATE INDEX idx_orders_created ON orders(created_at DESC, id DESC);
```

**优化方案三：只查主键再 JOIN**
```sql
SELECT o.* FROM orders o
INNER JOIN (
    SELECT id FROM orders ORDER BY id LIMIT 10 OFFSET 100000
) t ON o.id = t.id;
```

**优化方案四：`OFFSET` 本身要尽量避免**
产品层面引导用户使用"下一页"而非"跳页"，避免大 OFFSET。

> 📎 来源：PostgreSQL 官方文档 — "LIMIT and OFFSET"  
> https://www.postgresql.org/docs/current/queries-limit.html

---

## 题目四：Redis 缓存与数据库双写一致性如何保证？有哪些方案及其权衡？

### 参考答案

**方案一：Cache Aside（旁路缓存）— 最常用**
```
读：先读缓存，缓存未命中再读 DB 并写回缓存
写：先更新 DB，再删除缓存（而非更新缓存）
```
**为什么删而不是更新？** 更新缓存时若并发写可能产生脏数据；删除缓存代价最低，下一次读自然补上。

**方案二：Read Through / Write Through**
由缓存中间件代为读写，对应用层透明，但实现复杂，一般不推荐。

**方案三：延迟双删（解决并发问题）**
```python
# 1. 先删缓存
redis.del(key)
# 2. 更新数据库
db.update(...)
# 3. 延迟一段时间后再删一次（等待并发读完成）
time.sleep(0.5)
redis.del(key)
```

**方案四：订阅 MySQL binlog 异步更新缓存（Canal / DataGear）**
数据库写成功后，通过 binlog 解析异步写 Redis，适合高并发写入场景。

**各方案权衡：**

| 方案 | 一致性 | 复杂度 | 适用场景 |
|------|--------|--------|----------|
| Cache Aside | 最终一致 | 低 | 读多写少 |
| 延迟双删 | 强一致 | 中 | 写多读少 |
| binlog 订阅 | 最终一致 | 高 | 高并发核心业务 |

> 📎 来源：Redis 官方文档 — "Redis persistence"  
> https://redis.io/docs/management/persistence/

---

## 题目五：MySQL InnoDB 的 MVCC 机制是什么？如何解决幻读问题？

### 参考答案

**MVCC（Multi-Version Concurrency Control）多版本并发控制**

InnoDB 每一行数据都带两个隐藏列：
- `DB_TRX_ID`：最近一次修改该行的事务 ID
- `DB_ROLL_PTR`：指向 undo log 链表的指针（用于回滚）

**读已提交（RC）下的 MVCC 行为：**
每次读取时生成一个新的 ReadView，只读取已提交事务的版本。

**可重复读（RR）下的 MVCC 行为：**
事务开始时生成一个 ReadView，整个事务期间复用，保证可重复读。

**如何解决幻读：**
1. **快照读**：由 MVCC 保证，读取事务开始时的快照，不受其他事务 INSERT 影响。
2. **当前读**（`SELECT ... FOR UPDATE / INSERT / UPDATE / DELETE`）：由 Next-Key Lock（记录锁 + 间隙锁）锁住索引区间，阻止其他事务在该区间插入新记录。

```sql
-- 演示 Next-Key Lock 防止幻读
BEGIN;
SELECT * FROM orders WHERE id BETWEEN 100 AND 200 FOR UPDATE;
-- 此时其他事务无法 INSERT id 在 100~200 之间的记录
```

**MVCC + Next-Key Lock 组合**使 InnoDB 在 RR 隔离级别下彻底解决幻读问题。

> 📎 来源：InnoDB 官方文档 — "MVCC (Multi-Version Concurrency Control)"  
> https://dev.mysql.com/doc/refman/8.0/en/innodb-multi-versioning.html

---

*整理：小憨宝 | 适合 MySQL/PostgreSQL 数据库工程师面试复习*
