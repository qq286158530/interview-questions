# 数据库性能优化面试题

> 发布日期：2026-06-27
> 来源：整理自小林coding、菜鸟教程等优质技术博客

---

## 题目一：MySQL 索引失效的常见场景有哪些？

### 参考答案

索引失效是指在查询中虽然字段上建立了索引，但优化器并未选择使用该索引，导致全表扫描。常见的索引失效场景包括：

**1. 使用左模糊或左右模糊匹配（LIKE %xx 或 LIKE %xx%）**
```sql
-- 索引失效
SELECT * FROM users WHERE name LIKE '%张%';
SELECT * FROM users WHERE name LIKE '%张';

-- 索引生效
SELECT * FROM users WHERE name LIKE '张%';
```
原因：索引按从左到右的顺序组织，模糊匹配开头会导致无法定位起始位置。

**2. 对索引列进行计算、函数或类型转换**
```sql
-- 索引失效
SELECT * FROM orders WHERE YEAR(create_time) = 2026;
SELECT * FROM users WHERE age + 1 = 10;
SELECT * FROM users WHERE phone = 13800138000; -- phone 是 varchar 类型

-- 正确写法
SELECT * FROM orders WHERE create_time >= '2026-01-01' AND create_time < '2027-01-01';
SELECT * FROM users WHERE age = 9;
SELECT * FROM users WHERE phone = '13800138000';
```

**3. 联合索引不遵循最左匹配原则**
```sql
-- 创建联合索引 (a, b, c)
-- 以下情况索引失效
SELECT * FROM t_table WHERE b = 2;
SELECT * FROM t_table WHERE c = 3;
SELECT * FROM t_table WHERE b = 2 AND c = 3;

-- 以下情况索引生效
SELECT * FROM t_table WHERE a = 1;
SELECT * FROM t_table WHERE a = 1 AND b = 2;
SELECT * FROM t_table WHERE a = 1 AND b = 2 AND c = 3;
```

**4. OR 连接的条件中，存在非索引列**
```sql
-- 索引失效（OR 后面 age 不是索引列）
SELECT * FROM users WHERE name = '张三' OR age = 18;

-- 正确做法：为 age 也建立索引
```

**5. 使用 NOT、!=、<> 等否定操作符**
```sql
-- 可能导致索引失效
SELECT * FROM users WHERE age != 18;
SELECT * FROM users WHERE age NOT IN (18, 20);
```

**6. 字符串不加引号**
```sql
-- 索引失效（类型隐式转换）
SELECT * FROM users WHERE phone = 13800138000;

-- 正确写法
SELECT * FROM users WHERE phone = '13800138000';
```

**实际开发建议**：使用 `EXPLAIN` 查看执行计划，通过 `key` 字段判断是否使用了索引，通过 `type` 字段判断扫描方式（全表扫描还是索引扫描）。

**来源**：[小林coding - MySQL索引常见面试题](https://xiaolincoding.com/mysql/index/index_interview.html)

---

## 题目二：为什么 MySQL InnoDB 选择 B+Tree 而非 BTree 或 Hash 作为索引结构？

### 参考答案

**1. B+Tree vs BTree**
- BTree 非叶子节点也存储数据，B+Tree 只有叶子节点存储数据
- B+Tree 单个节点数据量更小，相同磁盘IO次数下能查询更多节点
- B+Tree 叶子节点采用双链表连接，适合范围查询和顺序查找，BTree 无法做到

**2. B+Tree vs 二叉树**
- 二叉树搜索复杂度 O(logN)，B+Tree 搜索复杂度 O(logdN)（d 为节点最大子节点数，通常 d>100）
- 千万级数据量，B+Tree 高度仅 3-4 层（3-4 次磁盘IO），二叉树 IO 次数更多

**3. B+Tree vs Hash**
- Hash 等值查询 O(1)，效率极高
- 但 Hash 不支持范围查询和排序，而 B+Tree 天然支持范围查找和顺序遍历
- MySQL 场景中，范围查询是常态，因此 B+Tree 更适合

**总结**：B+Tree 通过多叉结构降低树高度，减少磁盘IO次数；叶子节点链表结构支持高效范围查询；非叶子节点不存储数据，单页能容纳更多索引条目，查询效率稳定。

**来源**：[小林coding - MySQL索引常见面试题](https://xiaolincoding.com/mysql/index/index_interview.html)

---

## 题目三：什么是聚簇索引和二级索引？回表是什么？如何避免？

### 参考答案

**聚簇索引（Clustered Index）**
- 主键索引即为聚簇索引
- 叶子节点存储的是完整的行数据（整行记录）
- 每张表只能有一个聚簇索引（数据只能按一种方式排序存储）

**二级索引（Secondary Index / 辅助索引）**
- 除主键索引外的索引都是二级索引（普通索引、唯一索引、前缀索引等）
- 叶子节点存储的是主键值，而非完整数据
- 需要通过主键值再到聚簇索引中查找完整数据

**回表（Lookup）**
```sql
-- 假设 name 是二级索引
SELECT * FROM users WHERE name = '张三';
```
执行过程：
1. 在 name 索引的 B+Tree 中找到 name='张三' 对应的主键 ID
2. 再用 ID 到主键索引的 B+Tree 中查找完整行数据
3. 这个过程就是"回表"，需要查询两个 B+Tree

**覆盖索引（Covering Index）—— 避免回表**
如果查询的字段在二级索引的叶子节点中已经包含，就不需要回表：
```sql
-- 只需要查 name 字段，该字段在二级索引中，可覆盖索引，无需回表
SELECT name FROM users WHERE name = '张三';

-- 对比：查询 *，需要回表获取其他字段
SELECT * FROM users WHERE name = '张三'; -- 需要回表
```

**覆盖索引的优势**：
- 减少磁盘 IO 操作
- 显著提升查询性能
- 特别适合高频查询字段的优化

**来源**：[小林coding - MySQL索引常见面试题](https://xiaolincoding.com/mysql/index/index_interview.html)

---

## 题目四：MySQL 查询性能优化有哪些常用手段？

### 参考答案

**1. 优化索引**
- 确保查询条件字段有合适索引
- 遵循最左前缀匹配原则
- 使用覆盖索引减少回表
- 前缀索引优化大字符串字段（减小索引体积）

**2. 避免全表扫描**
- 用 EXPLAIN 分析执行计划，type 应优于 all（全表扫描）
- 尽量让 type 达到 ref、eq_ref、const 级别
- 避免 SELECT *，只查询必要字段

**3. 优化 SQL 语句**
- 避免在 WHERE 条件中对字段进行计算或函数操作
- 避免隐式类型转换（字符串字段不加引号）
- 避免 OR 连接不同类型条件
- 使用 LIMIT 限制结果集大小

**4. 优化表结构**
- 主键使用自增 ID（避免页分裂）
- 字段设置为 NOT NULL（节省空间，优化器更易优化）
- 适度冗余字段减少 JOIN 查询
- 合理使用分区表

**5. 读写分离**
- 主库处理写操作，从库处理读操作
- 分担单库压力，提升吞吐量

**6. 分库分表**
- 水平拆分：按 ID 或时间分表，降低单表数据量
- 垂直拆分：按业务模块拆分表

**7. 配置参数调优**
- `innodb_buffer_pool_size`：缓冲池大小，建议设置为主机内存的 60-80%
- `max_connections`：最大连接数
- `slow_query_log`：开启慢查询日志定位性能问题

**来源**：[小林coding - MySQL索引常见面试题](https://xiaolincoding.com/mysql/index/index_interview.html)

---

## 题目五：什么是索引下推（Index Condition Pushdown）？它如何提升查询性能？

### 参考答案

**索引下推（ICP，Index Condition Pushdown）**

是 MySQL 5.6 引入的优化策略。默认情况下，使用联合索引查询时，如果查询条件包含索引字段但不符合最左匹配原则，则整个索引失效。索引下推可以在遍历联合索引的过程中，对索引包含的字段先做判断，直接过滤掉不满足条件的记录，减少回表次数。

**示例**：有联合索引 (name, age)，执行以下查询：
```sql
SELECT * FROM users WHERE name LIKE '张%' AND age = 18;
```

**MySQL 5.6 之前**：
1. 在联合索引中定位到所有 name LIKE '张%' 的记录（这些记录相邻）
2. 逐条回表到主键索引查找完整行数据
3. 在主键索引中判断 age = 18

**MySQL 5.6 之后（开启 ICP）**：
1. 在联合索引中定位到所有 name LIKE '张%' 的记录
2. **在联合索引内部直接判断 age = 18**，过滤掉不满足条件的记录
3. 只对满足两个条件的记录进行回表

**效果**：大幅减少回表次数，提升查询性能。

**判断是否使用了索引下推**：通过 `EXPLAIN` 查看执行计划，Extra 列显示 `Using index condition` 表示使用了索引下推。

**适用场景**：
- 联合索引查询
- 范围查询（>、<）后的字段
- 前缀模糊匹配后的字段

**来源**：[小林coding - MySQL索引常见面试题](https://xiaolincoding.com/mysql/index/index_interview.html)

---

> 📌 以上题目涵盖索引原理、索引失效、查询优化、存储引擎等核心知识点，建议配合实践理解。