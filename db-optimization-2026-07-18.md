# 数据库性能优化面试题

> 📅 日期：2026-07-18
> 🏷️ 分类：MySQL / PostgreSQL 性能优化

---

## 题目一：MySQL 中 InnoDB 和 MyISAM 存储引擎有哪些核心区别？如何选择？

### 参考答案

**InnoDB 与 MyISAM 的核心区别：**

| 特性 | InnoDB | MyISAM |
|------|--------|--------|
| **事务支持** | 支持 ACID 事务，支持提交和回滚 | 不支持事务 |
| **行级锁** | 支持行级锁，并发性能好 | 只支持表级锁 |
| **外键约束** | 支持外键约束 | 不支持外键 |
| **崩溃恢复** | 支持自动崩溃恢复，有 redo log | 崩溃后可能损坏 |
| **全文索引** | MySQL 5.6+ 支持 | 原生支持全文索引 |
| **存储方式** | 数据和索引一起存储 (.ibd) | 数据和索引分开存储 (.MYD, .MYI) |
| **COUNT(*) 性能** | 全表扫描，较慢 | 有专门计数器，很快 |

**选择建议：**

- 需要**事务支持**、**行级锁**、**并发写入**场景 → 选择 InnoDB
- 只需要**全文搜索**，且表以读为主 → MyISAM（已逐渐被淘汰）
- **MySQL 8.0+ 默认存储引擎为 InnoDB**，生产环境强烈建议使用

**来源：** [FullStack.Cafe - MySQL Interview Questions](https://www.fullstack.cafe/blog/mysql-interview-questions)

---

## 题目二：什么是数据库索引？聚簇索引（Clustered Index）和非聚簇索引（Non-Clustered Index）有什么区别？

### 参考答案

**索引的定义：**
索引是数据库表中一种特殊的数据结构，用于加速数据检索。它类似于书籍的目录，可以快速定位到目标数据，而不需要扫描整个表。

**聚簇索引 vs 非聚簇索引：**

| 特性 | 聚簇索引 | 非聚簇索引 |
|------|----------|------------|
| **数据存储** | 数据行物理顺序与索引顺序一致 | 索引与数据分开存储 |
| **表只能有** | 只能有 1 个聚簇索引 | 可以有多个非聚簇索引 |
| **查询性能** | 范围查询极快 | 点查询需回表 |
| **插入性能** | 插入时可能需要移动数据页 | 插入较快 |
| **空间消耗** | 较小（索引即数据） | 较大（额外存储索引结构） |

**InnoDB 中的具体表现：**
- **聚簇索引**：表的主键索引就是聚簇索引，数据按主键顺序物理存储
- **非聚簇索引**：二级索引存储主键值，查询时需要"回表"根据主键再查聚簇索引

**最佳实践：**
- 用自增主键或短主键（整型）作为聚簇索引，避免 UUID 等长字符串
- 频繁查询的列建立非聚簇索引，但注意索引数量不要过多（影响写入性能）

**来源：** [InterviewBit - MySQL Interview Questions](https://www.interviewbit.com/mysql-interview-questions/)

---

## 题目三：如何优化慢查询？列举你常用的 SQL 优化方法和工具。

### 参考答案

**常用 SQL 优化方法：**

1. **使用 EXPLAIN 分析查询计划**
   ```sql
   EXPLAIN SELECT * FROM orders WHERE customer_id = 100;
   ```
   重点关注：type（最好到 ref/range）、key（使用的索引）、rows（扫描行数）

2. **避免 SELECT *，只查询需要的字段**
   减少网络传输和内存占用，让覆盖索引发挥更大作用

3. **合理使用 LIMIT 分页**
   ```sql
   -- 低效：OFFSET 大时很慢
   SELECT * FROM orders LIMIT 10000, 20;
   
   -- 高效：基于主键
   SELECT * FROM orders WHERE id > 10000 LIMIT 20;
   ```

4. **避免在索引列上使用函数或计算**
   ```sql
   -- 低效：索引失效
   WHERE YEAR(created_at) = 2026
   
   -- 高效：范围查询
   WHERE created_at >= '2026-01-01' AND created_at < '2027-01-01'
   ```

5. **使用 ANALYZE TABLE 更新统计信息**
   帮助优化器选择更好的执行计划

**常用诊断工具：**
- **慢查询日志** (`slow_query_log`)
- **Performance Schema** (MySQL 5.6+)
- **pt-query-digest** (Percona Toolkit)
- **MySQL Workbench** 的可视化 EXPLAIN

**来源：** [PostgreSQL Documentation - Performance Tips](https://www.postgresql.org/docs/current/performance-tips.html)

---

## 题目四：PostgreSQL 中如何使用 EXPLAIN 和 EXPLAIN ANALYZE 诊断查询性能？

### 参考答案

**EXPLAIN vs EXPLAIN ANALYZE：**

- **EXPLAIN**：只显示优化器估算的执行计划，不实际执行
- **EXPLAIN ANALYZE**：实际执行查询，并显示真实的运行时统计信息

**输出关键字段解析：**

```
EXPLAIN ANALYZE SELECT * FROM tenk1 WHERE unique1 < 100;

QUERY PLAN
------------------------------------------------------------------------------
 Bitmap Heap Scan on tenk1 (cost=5.06..224.98 rows=100 width=244)
                      (actual time=0.030..0.450 rows=100 loops=1)
   Recheck Cond: (unique1 < 100)
   Heap Blocks: exact=90
   Buffers: shared hit=92
   -> Bitmap Index Scan on tenk1_unique1 (cost=0.00..5.04 rows=100 width=0)
                      (actual time=0.013..0.013 rows=100 loops=1)
         Index Cond: (unique1 < 100)
         Buffers: shared hit=2
```

**Cost 参数含义：**
- `cost=5.06..224.98`：预估启动成本..总成本
- 成本单位是磁盘页面读取（默认 seq_page_cost=1.0）

**Actual 参数含义：**
- `actual time=0.030..0.450`：实际耗时（毫秒）
- `rows=100`：实际返回行数
- `loops=1`：该节点执行次数
- `Buffers: shared hit=92`：共享缓冲区命中/读取数

**优化方向：**
- 比较估算值与实际值，差异大说明统计信息过时
- 关注 Seq Scan（全表扫描），考虑添加索引
- Hash Join 和 Merge Join 的 cost 对比帮助选择最佳 join 方式

**来源：** [PostgreSQL Documentation - Using EXPLAIN](https://www.postgresql.org/docs/current/using-explain.html)

---

## 题目五：什么是数据库缓存策略？如何设计一个有效的缓存层？

### 参考答案

**数据库缓存策略主要类型：**

1. **Query Cache（查询缓存）** ⚠️ MySQL 8.0 已移除
   - 缓存 SELECT 语句及其结果
   - 适用于重复查询
   - 缺点：任何数据修改都会失效相关缓存

2. **Buffer Pool（缓冲池）**
   - InnoDB 将磁盘数据页缓存到内存
   - 通过 `innodb_buffer_pool_size` 配置（建议设为机器内存的 60-80%）
   - 通过 `innodb_buffer_pool_instances` 分区减少锁竞争

3. **应用层缓存（Redis/Memcached）**
   ```
   缓存模式：
   Cache Aside（旁路缓存）
   ├── 读：先读缓存 → 命中返回；未命中读DB → 写缓存 → 返回
   └── 写：先写DB → 删除缓存（不是更新）
   
   Read Through
   └── 缓存自动加载数据，应用只操作缓存
   
   Write Through
   └── 同步写DB和缓存
   ```

**缓存失效策略：**
- **TTL（Time To Live）**：设置过期时间
- **LRU（Least Recently Used）**：淘汰最久未使用的
- **LFU（Least Frequently Used）**：淘汰访问频率最低的

**缓存问题及解决方案：**

| 问题 | 解决方案 |
|------|----------|
| 缓存穿透 | 布隆过滤器 / 空值缓存 |
| 缓存击穿 | 互斥锁 / 永不过期 + 异步更新 |
| 缓存雪崩 | TTL 随机化 / 多级缓存 / 高可用 |

**最佳实践：**
- 热点数据才值得缓存（访问频率高）
- 避免大 Key（单 Key 不超过 1MB）
- 缓存预热（系统启动时加载热点数据）
- 监控缓存命中率，及时调整策略

**来源：** [InterviewBit - MySQL Advanced Questions](https://www.interviewbit.com/mysql-interview-questions/)

---

> 📚 更多面试题：[Interview Questions GitHub 仓库](https://github.com/qq286158530/interview-questions)
