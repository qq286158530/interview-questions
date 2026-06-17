# 数据库性能优化面试题

> 📅 日期：2026-06-17
> 🏷️ 标签：MySQL | PostgreSQL | 数据库优化 | 索引 | 面试题

---

## 题目一：为什么 MySQL InnoDB 选择 B+Tree 作为索引的数据结构？

### 参考答案

MySQL InnoDB 选择 B+Tree 作为索引数据结构，主要有以下三个原因：

**1. B+Tree vs B Tree**
- B+Tree 只在叶子节点存储数据，而 B 树的非叶子节点也要存储数据，所以 B+Tree 的单个节点的数据量更小
- 在相同的磁盘 I/O 次数下，B+Tree 能查询更多的节点
- B+Tree 叶子节点采用双链表连接，适合 MySQL 中常见的基于范围的顺序查找，而 B 树无法做到这一点

**2. B+Tree vs 二叉树**
- 对于有 N 个叶子节点的 B+Tree，其搜索复杂度为 O(logdN)，其中 d 表示节点允许的最大子节点个数（通常大于100）
- 即使数据达到千万级别时，B+Tree 的高度依然维持在 3~4 层左右，只需要 3~4 次磁盘 I/O
- 二叉树的搜索复杂度为 O(logN)，检索到目标数据所经历的磁盘 I/O 次数更多

**3. B+Tree vs Hash**
- Hash 在做等值查询时效率很高，搜索复杂度为 O(1)
- 但 Hash 不适合做范围查询，而 B+Tree 索引有着更广泛的适用场景

> 📚 **来源**：[小林 Coding - 索引常见面试题](https://xiaolincoding.com/mysql/index/index_interview.html)

---

## 题目二：什么是联合索引的最左匹配原则？请举例说明

### 参考答案

**联合索引**：通过将多个字段组合成一个索引，该索引被称为联合索引（复合索引）。

**最左匹配原则**：联合索引的 B+Tree 是先按第一个字段排序，在第一个字段相同的情况下再按第二个字段排序。因此，在使用联合索引进行查询时，必须按照从左到右的顺序匹配，否则联合索引会失效。

**示例**：创建联合索引 `(a, b, c)`

✅ **能匹配上联合索引的查询**：
```sql
WHERE a = 1                    -- 匹配 a
WHERE a = 1 AND b = 2          -- 匹配 a, b
WHERE a = 1 AND b = 2 AND c = 3  -- 匹配 a, b, c
```

❌ **无法匹配联合索引的查询**：
```sql
WHERE b = 2                    -- 不匹配（没有 a）
WHERE c = 3                    -- 不匹配（没有 a, b）
WHERE b = 2 AND c = 3          -- 不匹配（没有 a）
```

**特殊说明**：
- `>=`、`<=`、`BETWEEN`、like 前缀匹配（`like 'j%'`）不会停止匹配
- 使用 `>` 或 `<` 范围查询时，范围查询字段后面的字段无法利用联合索引

**索引下推优化**：MySQL 5.6 引入的索引下推（ICP）可以在联合索引遍历过程中，对联合索引中包含的字段先做判断，直接过滤掉不满足条件的记录，减少回表次数。

> 📚 **来源**：[小林 Coding - 索引常见面试题](https://xiaolincoding.com/mysql/index/index_interview.html)

---

## 题目三：什么情况下索引会失效？如何避免？

### 参考答案

**索引失效的常见场景**：

**1. 左模糊匹配（like %xx）**
```sql
-- 索引失效
SELECT * FROM t WHERE name LIKE '%小';
SELECT * FROM t WHERE name LIKE '%小%';

-- 索引有效（前缀匹配）
SELECT * FROM t WHERE name LIKE '小%';
```

**2. 对索引列进行计算、函数、类型转换**
```sql
-- 索引失效
SELECT * FROM t WHERE LEFT(name, 1) = '小';
SELECT * FROM t WHERE id + 1 = 10;
SELECT * FROM t WHERE YEAR(create_time) = 2024;

-- 正确写法
SELECT * FROM t WHERE name LIKE '小%';
SELECT * FROM t WHERE id = 9;
SELECT * FROM t WHERE create_time >= '2024-01-01' AND create_time < '2025-01-01';
```

**3. 不遵循最左匹配原则**
联合索引 `(a, b, c)` 不满足最左前缀时失效。

**4. OR 前后条件不统一**
```sql
-- 索引失效（OR 后的条件列不是索引列）
SELECT * FROM t WHERE a = 1 OR b = 2;

-- 正确写法：分别建立索引或创建联合索引
SELECT * FROM t WHERE a = 1 UNION SELECT * FROM t WHERE b = 2;
```

**5. 索引列使用 NOT、!=、<>**
```sql
-- 通常索引失效
SELECT * FROM t WHERE status != 1;
```

**优化建议**：
- 使用 EXPLAIN 查看执行计划，判断是否使用了索引
- 关注 type 列：All（全表扫描）是最坏情况
- 关注 Extra 列：Using filesort、Using temporary 都需要优化

> 📚 **来源**：[小林 Coding - 索引常见面试题](https://xiaolincoding.com/mysql/index/index_interview.html)

---

## 题目四：MySQL InnoDB 聚簇索引与二级索引有什么区别？

### 参考答案

**聚簇索引（Clustered Index）**：
- 主键索引的 B+Tree 的叶子节点存放的是**实际数据**
- 所有完整的用户记录都存放在聚簇索引的叶子节点里
- 每张表只能有一个聚簇索引（因为数据只能按一种方式排序）

**二级索引（Secondary Index / 辅助索引）**：
- 二级索引的 B+Tree 的叶子节点存放的是**主键值**，而不是实际数据
- 非叶子节点的 key 值是索引列的值

**查询过程的区别**：

使用二级索引查询数据时：
1. 先检索二级索引，找到对应叶子节点，获取主键值
2. 再通过主键索引查询到实际数据

这个过程称为**回表**（需要查询两个 B+Tree）。

**覆盖索引（Covering Index）**：
如果查询的字段能在二级索引的 B+Tree 叶子节点中直接获取到，就不需要回表，称为覆盖索引或索引覆盖。

```sql
-- 假设有联合索引 (product_no, name)
-- 这个查询只需要查联合索引，不需要回表（覆盖索引）
SELECT product_no, name FROM product WHERE product_no = 'P001';
```

**如何选择**：
- 主键字段长度不要太大（影响二级索引占用空间）
- 主键最好使用自增 ID（避免页分裂，提高插入效率）

> 📚 **来源**：[小林 Coding - 索引常见面试题](https://xiaolincoding.com/mysql/index/index_interview.html)

---

## 题目五：PostgreSQL 有哪些常见的性能优化手段？

### 参考答案

**1. 使用 EXPLAIN 和 EXPLAIN ANALYZE 分析查询**
```sql
EXPLAIN ANALYZE SELECT * FROM users WHERE name = '张三';
```
通过执行计划分析查询是否走索引、扫描类型、耗时等。

**2. 创建合适的索引**
```sql
-- 单列索引
CREATE INDEX idx_users_name ON users(name);

-- 联合索引（注意列的顺序）
CREATE INDEX idx_orders_user_status ON orders(user_id, status);

-- 表达式索引
CREATE INDEX idx_users_lower ON users(LOWER(name));
```

**3. 优化查询条件**
- 避免 SELECT *，只查询需要的字段
- 使用 LIMIT 限制结果集
- 避免在索引列上使用函数
- 使用预编译语句（Prepared Statements）

**4. 配置参数调优**
```sql
-- 查看当前配置
SHOW shared_buffers;
SHOW work_mem;
SHOW effective_cache_size;

-- 建议配置（根据服务器内存调整）
ALTER SYSTEM SET shared_buffers = '4GB';
ALTER SYSTEM SET work_mem = '64MB';
ALTER SYSTEM SET effective_cache_size = '12GB';
```

**5. 定期维护**
```sql
-- 更新统计信息
ANALYZE;

-- 清理无用数据
VACUUM;

-- 重建索引
REINDEX INDEX idx_users_name;
```

**6. 使用分区表**
对于大表，按时间或业务进行分区：
```sql
CREATE TABLE orders (
    id BIGSERIAL,
    created_at DATE
) PARTITION BY RANGE (created_at);

CREATE TABLE orders_2024 PARTITION OF orders
    FOR VALUES FROM ('2024-01-01') TO ('2025-01-01');
```

**7. 连接池和缓存**
- 使用 PgBouncer 等连接池
- 合理使用 PostgreSQL 缓存
- 集成 Redis 等外部缓存

> 📚 **来源**：[PostgreSQL Documentation - Performance Tips](https://www.postgresql.org/docs/current/performance-tips.html)

---

## 📖 更多资源

- [小林 Coding - MySQL 索引常见面试题](https://xiaolincoding.com/mysql/index/index_interview.html)
- [PostgreSQL 官方性能优化文档](https://www.postgresql.org/docs/current/performance-tips.html)

---

*本面试题由 AI 助手整理生成，仅供学习参考使用。*
