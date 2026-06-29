# 数据库性能优化面试题 — 2026-06-29

> 本文档整理了 5 道高质量数据库性能优化面试题，涵盖 MySQL 索引原理、查询优化、事务机制、存储引擎，以及 PostgreSQL 查询计划分析等核心知识点。题目来源于小林coding（xiaolincoding.com）等技术博客及 PostgreSQL 官方文档。

---

## 题目一：为什么 MySQL InnoDB 选择 B+Tree 作为索引的数据结构？

### 参考答案

MySQL InnoDB 选择 B+Tree 而不是 B 树、二叉树或 Hash 索引，主要有以下几个原因：

#### 1. B+Tree vs B 树
- **B 树非叶子节点也存储数据**，而 B+Tree 只有叶子节点存储数据。因此 B+Tree 单个节点的数据量更小，在相同的磁盘 I/O 次数下，能查询更多的节点。
- B+Tree 叶子节点采用双向链表连接，**适合 MySQL 中常见的基于范围的顺序查找**，而 B 树无法做到这一点。

#### 2. B+Tree vs 二叉树
- 对于有 N 个叶子节点的 B+Tree，其搜索复杂度为 O(logdN)，其中 d 是节点允许的最大子节点个数（实际应用中 d > 100）。
- 即使数据达到千万级别，B+Tree 的高度依然维持在 3~4 层，意味着一次查询最多只需 3~4 次磁盘 I/O。
- 二叉树每个父节点只有 2 个子节点，搜索复杂度为 O(logN)，检索到目标数据经历的磁盘 I/O 次数更多。

#### 3. B+Tree vs Hash
- Hash 在等值查询时效率很高，搜索复杂度为 O(1)。
- 但 Hash **不适合做范围查询**，而 B+Tree 索引有着更广泛的适用场景。

#### 4. 总结
B+Tree 通过将数据存储在叶子节点并用链表连接，既保证了查询效率（树高仅 3~4 层），又支持高效的区间查询，是索引数据结构的最优选择。

**来源：** [小林coding - MySQL 索引常见面试题](https://xiaolincoding.com/mysql/index/index_interview.html)

---

## 题目二：MySQL 中什么情况下会导致索引失效？

### 参考答案

索引失效是面试中的高频考点，以下几种情况会导致索引无法使用：

#### 1. 不满足最左匹配原则
联合索引 `(a, b, c)` 会按最左边的字段依次排序。当查询条件不包含最左边的字段时，索引失效。
```sql
-- 索引生效
WHERE a = 1 AND b = 2 AND c = 3
WHERE a = 1 AND b = 2

-- 索引失效（跳过了 a）
WHERE b = 2
WHERE c = 3
WHERE b = 2 AND c = 3
```

#### 2. 使用了范围查询（>、<、BETWEEN）后的字段
联合索引遇到范围查询时，**范围查询字段右边的字段无法利用索引**。
```sql
-- 联合索引 (a, b)
-- a 使用了 >，那么 b 无法使用索引
WHERE a > 1 AND b = 2  -- 只有 a 用到了索引
```

#### 3. 索引列上使用了函数或运算
```sql
-- 索引失效
WHERE YEAR(create_time) = 2026
WHERE id + 1 = 100

-- 正确写法
WHERE create_time >= '2026-01-01' AND create_time < '2027-01-01'
```

#### 4. 字符串不加引号
```sql
-- 隐式类型转换导致索引失效
WHERE phone = 13800138000  -- phone 是 varchar 类型
```

#### 5. LIKE 以 % 开头
```sql
-- 索引失效
WHERE name LIKE '%王'

-- 索引生效
WHERE name LIKE '王%'
```

#### 6. OR 连接了非索引列
```sql
-- 如果 name 有索引，age 没有索引，则索引失效
WHERE name = '张三' OR age = 20
```

**来源：** [小林coding - MySQL 索引常见面试题](https://xiaolincoding.com/mysql/index/index_interview.html)

---

## 题目三：请解释 MySQL 的 Buffer Pool 缓存机制及其对性能的影响。

### 参考答案

#### 什么是 Buffer Pool？

Buffer Pool 是 InnoDB 存储引擎在内存中划分的一块缓存区域，用于缓存磁盘上的数据页和索引页，是 MySQL 提升读写性能的核心机制。

#### 工作原理

- **读取数据**：先检查数据是否在 Buffer Pool 中，若存在（命中）则直接返回，否则从磁盘读取并缓存到 Buffer Pool。
- **修改数据**：直接在 Buffer Pool 中修改对应页，将该页标记为**脏页**，后续由后台线程在合适的时机将脏页刷新到磁盘。

#### Buffer Pool 缓存的内容

InnoDB 把存储的数据划分为若干个「页」，以页作为磁盘和内存交互的基本单位，默认页大小为 16KB。Buffer Pool 缓存的内容包括：
- **索引页**（Index Page）
- **数据页**（Data Page）
- **Undo 页**（记录回滚日志）
- **插入缓存**（Insert Buffer）
- **自适应哈希索引**（Adaptive Hash Index）
- **锁信息**等

#### Buffer Pool 对性能的影响

1. **减少磁盘 I/O**：热点数据缓存在内存中，大幅减少磁盘随机读写。
2. **脏页刷新策略**：不会立即将脏页写入磁盘，由后台线程选择合适时机批量刷盘，减少 I/O 压力。
3. **预读机制**：InnoDB 会预读相邻的页，减少未来可能的磁盘访问。

#### 相关配置参数

| 参数 | 说明 | 默认值 |
|------|------|--------|
| `innodb_buffer_pool_size` | Buffer Pool 大小 | 128MB |
| `innodb_buffer_pool_instances` | Buffer Pool 实例数 | 1 |
| `innodb_log_buffer_size` | Redo Log Buffer 大小 | 16MB |

#### 注意事项

- Buffer Pool 基于内存，断电后数据会丢失，但通过 **redo log** 可以实现 crash-safe 恢复。
- 生产环境中，建议将 `innodb_buffer_pool_size` 设置为机器物理内存的 50%~80%`。

**来源：** [小林coding - MySQL Buffer Pool 详解](https://xiaolincoding.com/mysql/log/how_update.html)

---

## 题目四：MySQL 的 undo log、redo log 和 binlog 有什么区别？它们各自的作用是什么？

### 参考答案

这三种日志是 MySQL 事务实现 ACID 特性的关键，面试中经常被问到它们的区别。

#### 三种日志对比

| 特性 | undo log | redo log | binlog |
|------|----------|----------|--------|
| **所属层次** | InnoDB 存储引擎层 | InnoDB 存储引擎层 | MySQL Server 层 |
| **记录内容** | 数据修改**前的值** | 数据修改**后的值** | 数据修改的**逻辑 SQL** |
| **主要作用** | 事务回滚、MVCC | 事务持久性、崩溃恢复 | 数据备份、主从复制 |
| **日志类型** | 逻辑日志（回滚操作） | 物理日志（页修改） | 逻辑日志（SQL 语句） |
| **持久化方式** | 通过 redo log 保证 | 事务提交时刷盘 | 由 sync_binlog 参数控制 |

#### undo log（回滚日志）

- **作用**：保证事务的**原子性**，用于事务回滚和 MVCC。
- **原理**：在事务执行前记录数据的旧值。当事务回滚时，读取 undo log 做相反操作（delete 反向为 insert，insert 反向为 delete，update 反向为用旧值更新）。
- **版本链**：undo log 通过 roll_pointer 指针形成链表，配合 ReadView 实现 MVCC（多版本并发控制）。

#### redo log（重做日志）

- **作用**：保证事务的**持久性**，用于崩溃恢复。
- **原理**：采用 WAL（Write-Ahead Logging）策略——先写日志到 redo log buffer，再写数据到磁盘。崩溃后可根据 redo log 恢复已提交的数据。
- **redo log  vs 数据页写磁盘**：redo log 采用**追加写**（顺序 I/O），比数据页的随机 I/O 高效得多。
- **刷盘时机**：
  - `innodb_flush_log_at_trx_commit=1`（默认）：事务提交时强制刷盘，最安全。
  - `innodb_flush_log_at_trx_commit=0`：每秒刷盘，可能丢失 1 秒数据。
  - `innodb_flush_log_at_trx_commit=2`：写到 Page Cache，由 OS 控制刷盘。

#### binlog（归档日志）

- **作用**：用于**数据备份**和**主从复制**。
- **特点**：binlog 是 Server 层日志，记录的是逻辑变化（如 `UPDATE t SET a=1 WHERE id=2`），是 SQL 级别的记录。

#### 三者协作流程（一条 UPDATE 语句）

1. 事务开始，记录 undo log（旧值）。
2. 执行更新，先修改 Buffer Pool 中的数据页（标记脏页）。
3. 记录 redo log（新值），写入 redo log buffer。
4. 事务提交，redo log 刷盘，binlog 刷盘。
5. 后台线程将脏页刷新到磁盘。

**来源：** [小林coding - MySQL 日志：undo log、redo log、binlog 有什么用？](https://xiaolincoding.com/mysql/log/how_update.html)

---

## 题目五：如何使用 PostgreSQL 的 EXPLAIN 分析查询性能？有哪些关键指标？

### 参考答案

PostgreSQL 的 `EXPLAIN` 命令是分析查询计划的核心工具，通过它可以了解查询优化器如何执行 SQL 语句。

#### EXPLAIN 基本用法

```sql
EXPLAIN SELECT * FROM tenk1 WHERE unique1 < 7000;
```

输出示例：
```
Seq Scan on tenk1  (cost=0.00..470.00 rows=7000 width=244)
  Filter: (unique1 < 7000)
```

#### 关键指标解释

- **cost=0.00..470.00**：左侧是启动成本（返回第一行前的时间），右侧是总成本。
- **rows=7000**：估算返回的行数。
- **width=244**：每行平均字节数。

#### 常见的扫描类型（Scan Types）

| 类型 | 说明 |
|------|------|
| **Seq Scan**（顺序扫描） | 全表扫描，适合小表或无索引的情况 |
| **Index Scan**（索引扫描） | 先扫描索引找到行位置，再访问表 |
| **Bitmap Index Scan**（位图索引扫描） | 先用索引定位所有行位置，用位图批量访问表 |
| **Index Only Scan** | 直接从索引返回数据，无需访问表（覆盖索引） |

#### 影响查询计划的关键参数

| 参数 | 说明 |
|------|------|
| `enable_seqscan` | 是否允许顺序扫描（默认 on）|
| `enable_indexscan` | 是否允许索引扫描（默认 on）|
| `enable_bitmapscan` | 是否允许位图扫描（默认 on）|
| `enable_hashjoin` | 是否允许 Hash 连接（默认 on）|
| `enable_nestloop` | 是否允许嵌套循环连接（默认 on）|

#### 优化建议

1. **看成本估算**：如果 cost 很高，考虑添加合适的索引。
2. **关注扫描方式**：大表上的 Seq Scan 通常意味着需要索引。
3. **检查行数估算**：如果 rows 与实际差异很大，需要运行 `ANALYZE` 更新统计信息。
4. **使用 EXPLAIN ANALYZE**：实际执行并显示真实运行时间：
   ```sql
   EXPLAIN (ANALYZE, BUFFERS) SELECT * FROM tenk1 WHERE unique1 < 7000;
   ```

#### 性能优化技巧

1. **使用分区表**：`enable_partition_pruning=on`（默认）可在查询时跳过无关分区。
2. **合理创建索引**：对高频查询列创建 B-tree 索引（默认类型）。
3. **部分索引**：对数据分布不均匀的表，使用 `WHERE` 创建部分索引。
4. **表达式索引**：对使用函数的查询创建表达式索引，如 `CREATE INDEX ON table (upper(name))`。

**来源：** [PostgreSQL 官方文档 - Using EXPLAIN](https://www.postgresql.org/docs/current/using-explain.html)

---

> 📚 以上题目均经过筛选，涵盖数据库性能优化最核心的知识点。祝面试顺利！
