# 数据库性能优化面试题 — 2026-08-28

> 每日精选 5 道数据库性能优化面试题，涵盖索引优化、查询优化、存储引擎、缓存策略等核心知识点。

---

## 题目 1：联合索引的最左前缀匹配原则是什么？为什么会有这个限制？

**来源参考：** [MySQL 8.0 Reference — Optimization and Indexes](https://dev.mysql.com/doc/refman/8.0/en/optimization-indexes.html)

**答案：**

最左前缀匹配原则是指，在使用联合索引（复合索引）时，查询条件必须从索引的最左列开始，且不能跳过中间的列，否则后续列无法使用索引。

**示例：** 对于索引 `(a, b, c)`：
- `WHERE a = 1 AND b = 2 AND c = 3` ✅ 全部命中
- `WHERE a = 1 AND b = 2` ✅ 命中 a、b
- `WHERE a = 1` ✅ 命中 a
- `WHERE b = 2 AND c = 3` ❌ 无法使用索引（跳过了 a）
- `WHERE a = 1 AND c = 3` ⚠️ 只能用到 a，c 无法使用索引

**原因：** 联合索引在 B+ 树中是按照索引列的顺序依次排序的。先按第一列排序，第一列相同再按第二列排序，以此类推。如果查询条件不包含最左列，数据库无法定位到正确的索引区间，因为后续列的排序是建立在前一列基础上的。

**优化建议：**
- 将选择性高的列放在联合索引前面
- 设计索引时要覆盖高频查询的 WHERE 条件
- MySQL 8.0 引入了索引跳跃扫描（Index Skip Scan），在特定条件下可以放宽此限制

---

## 题目 2：EXPLAIN 执行计划中哪些关键字段需要重点关注？如何判断查询是否有性能问题？

**来源参考：** [MySQL EXPLAIN Output Format](https://dev.mysql.com/doc/refman/8.0/en/explain-output.html)

**答案：**

EXPLAIN 是分析 SQL 执行计划的核心工具，以下字段需要重点关注：

| 字段 | 说明 | 关注点 |
|------|------|--------|
| **type** | 访问类型 | 性能从好到差：`system > const > eq_ref > ref > range > index > ALL`。出现 `ALL`（全表扫描）需优化 |
| **key** | 实际使用的索引 | `NULL` 表示未使用索引 |
| **rows** | 预估扫描行数 | 数值越大性能越差 |
| **Extra** | 额外信息 | `Using filesort`（文件排序）和 `Using temporary`（临时表）需要警惕 |
| **filtered** | 过滤比例 | 百分比越低，说明索引过滤效果越差 |
| **possible_keys** | 可能使用的索引 | 与 `key` 对比，确认是否选择了最优索引 |

**判断性能问题的信号：**
1. `type = ALL`：全表扫描，通常需要添加索引
2. `Extra` 包含 `Using filesort`：ORDER BY 未使用索引，需优化排序
3. `Extra` 包含 `Using temporary`：GROUP BY 或 DISTINCT 产生临时表
4. `rows` 值远大于实际返回行数：索引选择性差
5. `key_len` 过短：联合索引未被充分利用

**实战技巧：** 使用 `EXPLAIN FORMAT=JSON` 或 `EXPLAIN ANALYZE`（MySQL 8.0.18+）可以获取更详细的执行信息，包括实际执行时间。

---

## 题目 3：MySQL 的 InnoDB 和 MyISAM 存储引擎在性能上有哪些关键差异？各自适合什么场景？

**来源参考：** [MySQL Storage Engines](https://dev.mysql.com/doc/refman/8.0/en/storage-engines.html)

**答案：**

| 特性 | InnoDB | MyISAM |
|------|--------|--------|
| **事务支持** | ✅ 支持 ACID 事务 | ❌ 不支持 |
| **行级锁** | ✅ 支持 | ❌ 仅表级锁 |
| **外键** | ✅ 支持 | ❌ 不支持 |
| **崩溃恢复** | ✅ 通过 redo log 恢复 | ❌ 需要修复 |
| **MVCC** | ✅ 支持多版本并发控制 | ❌ 不支持 |
| **全文索引** | ✅（MySQL 5.6+） | ✅ 支持 |
| **存储结构** | 聚簇索引（数据和主键索引在一起） | 非聚簇索引（数据和索引分离） |
| **COUNT(*)** | 需要遍历（有事务可见性问题） | 存储了行数，极快 |
| **压缩** | 支持表压缩 | 支持 myisampack 压缩 |

**性能差异：**
- **读多写少场景：** MyISAM 的 COUNT(*) 更快，无事务开销，读性能略好
- **高并发写入：** InnoDB 的行级锁远优于 MyISAM 的表级锁
- **大数据量：** InnoDB 的聚簇索引在主键查询上更快，但二级索引需要回表
- **内存占用：** InnoDB 的 buffer pool 管理更复杂，但缓存效果更好

**场景建议：**
- **绝大多数场景选 InnoDB：** 事务、并发、崩溃恢复是现代应用的刚需
- **MyISAM 适用场景：** 只读或读多写少的日志表、数据仓库的维度表（MySQL 8.0 已移除 MyISAM 作为默认引擎）

---

## 题目 4：什么是慢查询？如何系统性地定位和优化慢 SQL？

**来源参考：** [MySQL Slow Query Log](https://dev.mysql.com/doc/refman/8.0/en/slow-query-log.html)

**答案：**

**慢查询**是指执行时间超过 `long_query_time`（默认 10 秒）的 SQL 语句。系统性优化流程如下：

### 第一步：开启慢查询日志
```sql
SET GLOBAL slow_query_log = ON;
SET GLOBAL long_query_time = 1;  -- 超过1秒记录
SET GLOBAL log_queries_not_using_indexes = ON;  -- 记录未使用索引的查询
```

### 第二步：分析慢查询
使用 `mysqldumpslow` 或 `pt-query-digest`（Percona Toolkit）分析慢日志：
```bash
pt-query-digest /var/log/mysql/slow.log
```

### 第三步：使用 EXPLAIN 分析执行计划
重点关注 type、key、rows、Extra 字段。

### 第四步：常见优化手段

1. **索引优化：**
   - 添加缺失索引
   - 删除冗余索引和重复索引
   - 使用覆盖索引避免回表
   - 注意索引失效的场景（函数操作、隐式转换、LIKE '%xxx'）

2. **SQL 改写：**
   - 避免 `SELECT *`，只查需要的列
   - 子查询改 JOIN（MySQL 5.6 之前优化器对子查询优化差）
   - LIMIT 优化：大偏移量使用游标分页或延迟关联
   - 避免在 WHERE 条件中对索引列使用函数

3. **表结构优化：**
   - 合适的数据类型（INT vs BIGINT，VARCHAR vs CHAR）
   - 垂直拆分（大字段分离到扩展表）
   - 水平拆分（分库分表）

4. **架构层面：**
   - 读写分离
   - 引入缓存层（Redis）
   - 使用连接池避免连接风暴

---

## 题目 5：MySQL 的 Buffer Pool 工作原理是什么？如何优化 Buffer Pool 的性能？

**来源参考：** [MySQL InnoDB Buffer Pool](https://dev.mysql.com/doc/refman/8.0/en/innodb-buffer-pool.html)

**答案：**

### Buffer Pool 工作原理

InnoDB 的 Buffer Pool 是一块内存区域，用于缓存表数据和索引数据，避免每次查询都访问磁盘。

**核心机制：**
- **页（Page）管理：** 数据以 16KB 的页为单位加载到 Buffer Pool
- **LRU 变体算法：** InnoDB 使用改进的 LRU（最近最少使用）算法，将链表分为 young 区（热数据，约 5/8）和 old 区（新加载数据，约 3/8），防止全表扫描冲刷热数据
- **预读（Read-Ahead）：** 线性预读和随机预读，提前加载可能用到的页
- **脏页刷新：** 修改后的页标记为脏页，由后台线程异步刷盘

### 优化策略

1. **合理设置 Buffer Pool 大小：**
   ```sql
   SET GLOBAL innodb_buffer_pool_size = 8589934592;  -- 8GB，通常设为物理内存的 60%-80%
   ```

2. **多实例 Buffer Pool：** 减少并发争用
   ```sql
   SET GLOBAL innodb_buffer_pool_instances = 8;  -- 每个实例独立管理锁
   ```

3. **监控 Buffer Pool 命中率：**
   ```sql
   SHOW ENGINE INNODB STATUS;  -- 查看 Buffer pool hit rate
   -- 命中率低于 99% 需要关注
   ```
   命中率计算：`1 - (Innodb_buffer_pool_reads / Innodb_buffer_pool_read_requests)`

4. **预热（Warmup）：** 重启后 Buffer Pool 为空，可通过 `innodb_buffer_pool_dump_at_shutdown` 和 `innodb_buffer_pool_load_at_startup` 保存和恢复热数据

5. **调整 old 区比例：** 根据业务访问模式调整 `innodb_old_blocks_pct`（默认 37%）

6. **使用 SSD 替代 HDD：** 即使 Buffer Pool 命中率低，SSD 也能显著降低随机 I/O 的延迟

---

> 📌 **每日积累，持续进步！** 关注更多面试题请访问 [interview-questions 仓库](https://github.com/qq286158530/interview-questions)
