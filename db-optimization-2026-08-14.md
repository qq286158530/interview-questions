# 数据库性能优化面试题（2026-08-14）

## 第1题：MySQL 索引失效的场景有哪些？如何避免？

**参考答案：**

MySQL 索引失效的常见场景包括：

1. **使用函数或计算**：在索引列上使用函数或进行计算，如 `WHERE YEAR(create_time) = 2024` 或 `WHERE id + 1 = 10`，导致索引无法被使用。正确做法是将计算移到等式右侧：`WHERE create_time >= '2024-01-01' AND create_time < '2025-01-01'`。

2. **类型转换**：当查询条件中字段类型与索引列类型不一致时，MySQL 会进行隐式类型转换，可能导致索引失效。例如 `WHERE phone = 13800138000`（phone 为 varchar 类型）。

3. **LIKE 前面带通配符**：`LIKE '%abc'` 或 `LIKE '%abc%'` 会导致全表扫描，因为索引按前缀匹配。`LIKE 'abc%'` 可以使用索引。

4. **OR 连接条件**：当 OR 前后有一个列没有索引时，整个查询会跳过索引改用全表扫描。解决方案是为没有索引的列建立索引，或使用 UNION 分开查询。

5. **最左前缀原则被破坏**：对于联合索引 (a, b, c)，如果查询条件缺少 a 或跳过中间列（如 `WHERE a=1 AND c=2`），则会跳过 b 导致索引部分失效。

6. **范围查询右边的列**：联合索引中范围条件（>、<、BETWEEN、LIKE）右边的列无法使用索引。如 `WHERE a=1 AND b>2 AND c=3`，c 列无法使用索引。

7. **不等于（!= 或 <>）**：使用不等于条件时通常无法利用索引。

8. **IS NULL / IS NOT NULL**：部分情况下 MySQL 优化器会跳过索引，准确取决于数据分布。

**避免策略：**
- 保持 SQL 简单，避免在索引列上使用函数
- 确保类型匹配
- 尽量使用覆盖索引
- 用 EXPLAIN 分析查询计划

---

## 第2题：解释 MySQL InnoDB 存储引擎的 MVCC 机制及其在读已提交和可重复读隔离级别下的表现。

**参考答案：**

**MVCC（Multi-Version Concurrency Control）多版本并发控制**，是 InnoDB 实现的基于乐观锁的并发控制机制，通过保存数据的多个版本快照来实现读写不冲突。

**核心数据结构：**
- **undo log**：存储数据行修改前的镜像版本，形成版本链
- **read view**：读取操作发生时生成，记录当前活跃事务 ID 列表（trx_ids）和最小活动事务 ID（min_trx_id）、最大事务 ID（max_trx_id）

**工作原理：**
对于每行数据，InnoDB 额外维护两个隐藏列：`DB_TRX_ID`（最近修改的事务 ID）和 `DB_ROLL_PTR`（指向 undo log 的指针）。更新时将旧数据写入 undo log，指针构成版本链。

读取时根据 read view 和版本链找到符合条件的数据版本：
- 数据的 trx_id < min_trx_id：数据在该事务开始前已提交，可见
- trx_id 在 trx_ids 列表中：该数据由活跃事务修改，不可见，通过版本链找更早版本
- trx_id > max_trx_id：该数据在该事务开始后修改，不可见，找更早版本

**不同隔离级别的区别：**

| 隔离级别 | 每次 SELECT 生成 read view 时机 | 可重复读 |
|---------|---------------------------|---------|
| **读已提交（RC）** | 每次 SELECT 都重新生成 read view | 同一查询多次执行可能看到不同结果（其他事务已提交） |
| **可重复读（RR）** | 第一次 SELECT 时生成，整个事务复用同一个 read view | 同一事务中多次读取结果一致 |

**在 RR 下解决幻读问题：**
除了 MVCC，RR 级别还会使用 `Next-Key Lock`（记录锁+间隙锁）来锁定索引区间，防止其他事务在区间内插入新记录，从而解决幻读。

---

## 第3题：如何进行慢查询优化？请描述完整的排查和优化流程。

**参考答案：**

**慢查询优化标准流程：**

**Step 1：定位慢查询**
```sql
-- 开启慢查询日志
SET GLOBAL slow_query_log = 'ON';
SET GLOBAL long_query_time = 1; -- 超过1秒记录
SET GLOBAL slow_query_log_file = '/var/log/mysql/slow.log';

-- 查看慢查询日志
mysqldumpslow -s t -t 10 /var/log/mysql/slow.log
```

**Step 2：分析执行计划**
```sql
EXPLAIN SELECT ...;
EXPLAIN ANALYZE SELECT ...; -- 实际执行并返回时间（MySQL 8.0+）
```
关键字段：
- `type`：访问类型，从好到差为 system > const > eq_ref > ref > range > index > ALL
- `key`：实际使用的索引
- `rows`：扫描行数，越少越好
- `Extra`：Using filesort / Using temporary 表示需要额外排序或临时表，需要优化

**Step 3：检查索引**
- 是否存在合适的索引
- 是否符合最左前缀原则
- 是否可以使用覆盖索引（Extra: Using index）

**Step 4：优化技巧**
1. **单表查询优化**：先定位返回记录最少的表，从它开始单字段查询找出区分度最高的列加索引
2. **避免 SELECT ***：只查询需要的字段，减少网络传输和内存消耗
3. **分页优化**：`LIMIT 10000, 10` 应改为 `WHERE id > 10000 LIMIT 10`
4. **JOIN 优化**：小表驱动大表（`小表 LEFT JOIN 大表`），确保关联字段有索引
5. **批量操作**：多次 INSERT 合并为 `INSERT INTO t VALUES (...), (...), (...)`
6. **避免子查询**：在 MySQL 5.7 及以前，子查询会产生临时表，改为 JOIN 或先查后关联

**Step 5：业务层面**
- 分析业务场景，合理使用缓存
- 读写分离，主从分担查询压力
- 分库分表分散数据量

---

## 第4题：MySQL InnoDB 和 MyISAM 存储引擎的区别是什么？如何选择？

**参考答案：**

| 特性 | InnoDB | MyISAM |
|-----|--------|--------|
| **事务支持** | 支持 ACID 事务（commit/rollback） | 不支持事务 |
| **锁粒度** | 行级锁 + 间隙锁 | 表级锁 |
| **并发能力** | 高（行级锁支持更多并发） | 低（表锁并发差） |
| **外键约束** | 支持外键 | 不支持外键 |
| **崩溃恢复** | 自动崩溃恢复（redo log） | 需手动修复（myisamchk） |
| **全文索引** | 5.6+ 支持 FULLTEXT | 支持 FULLTEXT |
| **存储结构** | 表空间（tablespace） | 数据文件(.MYD)+索引文件(.MYI) |
| **COUNT(*)** | 全表扫描（无内部计数器） | 维护内部计数器，快 |
| **适用场景** | 核心业务、需要事务、高并发 | 读多写少、不需要事务 |

**InnoDB 的核心优势：**
- 支持行级锁，高并发下性能优秀
- 支持事务，适合金融、订单等核心系统
- 崩溃自动恢复，数据安全性高
- 支持外键，保证参照完整性

**选择建议：**
- **选择 InnoDB**：几乎所有场景，尤其是写入密集型、需要事务、数据可靠性要求高的业务
- **选择 MyISAM**：仅适用于极少量数据（< 1GB）、纯读场景、历史归档表，且接受表锁带来的性能损失

**MySQL 8.0+ 默认存储引擎已是 InnoDB**，MyISAM 已不再维护和更新。

---

## 第5题：PostgreSQL 中如何分析慢查询？EXPLAIN 输出各字段含义是什么？

**参考答案：**

**执行计划分析：**

```sql
EXPLAIN (ANALYZE, BUFFERS, FORMAT TEXT) SELECT ...;
```

- `EXPLAIN`：只显示计划，不执行
- `EXPLAIN ANALYZE`：实际执行并显示真实时间和计划
- `BUFFERS`：显示缓冲区命中情况
- `FORMAT TEXT/JSON/YAML`：输出格式

**EXPLAIN 输出字段解释：**

```
Seq Scan on users  (cost=0.00..445.00 rows=10000 width=244)
                   (actual time=0.011..5.123 rows=10000 loops=1)
                   Buffers: shared hit=345
```

- **cost=startup..total**：启动成本和总成本，以磁盘 page 读取为单位
  - `0.00`：读取第一行前的成本（如排序启动时间）
  - `445.00`：读完所有行的估计成本
- **rows=10000**：估计返回行数
- **width=244**：每行平均字节数
- **actual time**：实际耗时（启动..结束）单位毫秒
- **loops=1**：该节点执行次数
- **Buffers: shared hit=345**：从共享缓冲区命中 345 个块（hit=缓存命中，read=磁盘读取）

**常见节点类型：**
- **Seq Scan**：顺序扫描全表，适合小表或返回大量数据
- **Index Scan**：索引扫描后回表取数据
- **Bitmap Index Scan + Bitmap Heap Scan**：位图扫描，先扫描索引标记行位置，再批量取数据，适合中大量数据
- **Nested Loop**：嵌套循环连接，适合小表驱动
- **Hash Join / Hash**：哈希连接，大表关联
- **Sort / Limit / Aggregate**：排序、限制、聚合操作节点

**性能优化关键指标：**
1. **cost 值**：越低越好，重点关注 startup cost 过高
2. **actual time vs estimated**：estimated 和 actual 差距过大说明统计信息过时，需 `ANALYZE`
3. **Buffers read vs hit**：read 过多说明缓存不足，考虑增加 `shared_buffers` 或优化查询减少扫描量
4. **loops**：过高的 loops 值说明需要优化，如子节点重复执行

**优化建议：**
- 差距大时运行 `ANALYZE table_name` 更新统计信息
- 避免 Seq Scan 大表，全表扫描超过 5-10% 时考虑加索引
- 使用覆盖索引减少回表
- 合理设置 `random_page_cost`（SSD 上可设为接近 seq_page_cost）

---

> 本文档由 AI 自动生成并推送至 GitHub 仓库
> 来源参考：
> - 美团技术团队：https://tech.meituan.com/2014/06/30/mysql-index.html
> - PostgreSQL 官方文档：https://www.postgresql.org/docs/current/using-explain.html
> - GitHub interview_internal_reference：https://github.com/0voice/interview_internal_reference
