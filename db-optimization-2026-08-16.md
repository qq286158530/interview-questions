# 数据库性能优化面试题

> 📅 2026-08-16 | 适用于 MySQL 8.0 / PostgreSQL 14+

---

## 题目 1：MySQL InnoDB 主键索引为什么要建议使用自增 ID 而非 UUID？

### 参考答案

**核心原因：聚簇索引的物理存储顺序**

InnoDB 表的数据行以主键为基准按 B+ 树结构组织，叶子节点直接存储完整数据行（**聚簇索引**）。使用自增 ID 时，新插入的行总是在 B+ 树最右侧叶子节点追加，**顺序写磁盘**，无页面分裂、无大量随机 I/O。

**使用 UUID 的问题：**

| 问题 | 说明 |
|------|------|
| **页面分裂** | UUID 随机生成，新插入的主键可能插入到 B+ 树中间位置，引发页面分裂和数据迁移 |
| **随机 I/O** | 分裂后新页可能不在物理连续位置，磁盘读写退化为随机 I/O |
| **页利用率下降** | 频繁分裂导致页填充率不均匀，页空间浪费 |
| **主键索引体积膨胀** | 随机 UUID 造成大量碎片，索引更大，缓存效率下降 |

**实测数据参考：**
- 插入 1000 万条数据，自增主键比 UUID 快 **3~5 倍**
- UUID 主键索引体积约为自增 ID 的 **1.5~2 倍**

**替代方案：** 如需业务含义的主键，可在主键之上建立唯一索引；或使用雪花算法（Snowflake）等趋势递增 ID。

**来源：** 阿里巴巴《Java 开发手册》、MySQL 官方文档 Chapter 15.6.2.1

---

## 题目 2：如何排查和解决 MySQL 慢查询（Slow Query）？

### 参考答案

**Step 1：开启慢查询日志**

```sql
-- 查看慢查询配置
SHOW VARIABLES LIKE 'slow_query%';
SHOW VARIABLES LIKE 'long_query_time';

-- 开启慢查询日志（持久化需写入 my.cnf）
SET GLOBAL slow_query_log = 'ON';
SET GLOBAL long_query_time = 1;  -- 超过1秒记录
SET GLOBAL slow_query_log_file = '/var/log/mysql/slow.log';
```

**Step 2：分析慢查询日志**

```bash
# 使用 mysqldumpslow 汇总（MySQL 内置工具）
mysqldumpslow -t 5 -s at /var/log/mysql/slow.log

# 使用 pt-query-digest（Percona Toolkit，更强大）
pt-query-digest /var/log/mysql/slow.log
```

**Step 3：使用 EXPLAIN 分析执行计划**

```sql
EXPLAIN ANALYZE
SELECT * FROM orders o
JOIN users u ON o.user_id = u.id
WHERE o.status = 'paid' AND o.created_at > '2026-01-01';
```

关键关注字段：
- `type`：至少达到 `ref`，避免 `ALL`（全表扫描）
- `key`：确认使用了索引，而非 `NULL`
- `rows`：扫描行数是否合理
- `Using temporary`：出现临时表通常意味着需要优化
- `Using filesort`：文件排序，百万级以上数据应尽量避免

**Step 4：常见优化手段**

| 场景 | 优化方法 |
|------|----------|
| 全表扫描 | 添加 WHERE 条件索引 |
| 索引失效 | 避免函数/运算、隐式类型转换；使用覆盖索引 |
| 深度分页 | 改用延迟关联或游标分页（seek method） |
| JOIN 过多 | 拆分查询或增加缓存 |
| 临时表/文件排序 | 优化 GROUP BY/ORDER BY 字段顺序 |

**来源：** MySQL 官方文档 Chapter 8.14.4 "EXPLAIN Output Format"、Percona "Analyzing Slow Queries"

---

## 题目 3：PostgreSQL 中如何通过 EXPLAIN 读懂执行计划并定位性能瓶颈？

### 参考答案

**查看执行计划**

```sql
EXPLAIN (ANALYZE, BUFFERS, FORMAT TEXT)
SELECT u.name, COUNT(o.id) AS order_count
FROM users u
LEFT JOIN orders o ON u.id = o.user_id
WHERE u.created_at > '2026-01-01'
GROUP BY u.id;
```

**关键节点解读**

| 节点关键字 | 含义 | 关注点 |
|-----------|------|--------|
| `Seq Scan` | 顺序扫描（全表） | 数据量大时应避免 |
| `Index Scan` | 索引扫描 | 正常，较优 |
| `Index Only Scan` | 索引覆盖扫描 | 最优，无需回表 |
| `Bitmap Heap Scan` | 位图扫描 | 多条件查询时使用 |
| `Hash Join` | 哈希连接 | 大表 JOIN，等值连接首选 |
| `Nested Loop` | 嵌套循环 | 小表驱动大表时高效 |
| `Sort` | 排序 | 关注是否在内存完成 |
| `Hash Aggregate` | 哈希聚合 | GROUP BY 常用，较优 |

**重点指标**

```
Cost: start_cost..total_cost
  - start_cost：启动成本（返回第一行前开销）
  - total_cost：总成本（返回全部行开销）
  - rows：估算行数
  - actual rows：实际行数（ANALYZE 才有）
  - Buffers: shared hit=XX read=XX（缓存命中 vs 磁盘读取）
```

**性能瓶颈定位技巧**

- **cost 最大的节点** 即为最耗时步骤
- **Buffers read 远大于 hit** 说明大量磁盘 I/O，优先优化
- **rows 估算值与 actual rows 差距大** → 统计信息过期，需 `ANALYZE`
- **Subquery Scan + Function Scan** 可能存在隐式类型转换

**来源：** PostgreSQL 官方文档 Chapter 14.1 "Using EXPLAIN"、pganalyze.com "Reading EXPLAIN Plans"

---

## 题目 4：MySQL 读写分离后，如何解决主从延迟带来的数据不一致问题？

### 参考答案

**问题本质**

主从复制默认使用异步复制，主库写入后从库可能有 **几毫秒到几秒** 的延迟。读写分离场景下，读取从库可能读到过期数据。

**解决方案**

**方案一：业务层面妥协（最终一致性）**
- 对一致性要求高的读操作（订单状态、支付结果）**强制读主库**
- 对时效性要求低的场景（用户资料、历史订单）读从库
```java
@ReadOnly(type = ReadOnlyType.SLAVE)  // Spring 注解示例
public User getUser(Long id) { ... }
```

**方案二：半同步复制（Semi-sync Replication）**
```
主库写入 → 等待至少一个从库确认写入 relay log → 返回客户端
```
```sql
-- 主库安装插件
INSTALL PLUGIN rpl_semi_sync_master SONAME 'semisync_master.so';
SET GLOBAL rpl_semi_sync_master_enabled = 1;

-- 从库安装插件
INSTALL PLUGIN rpl_semi_sync_slave SONAME 'semisync_slave.so';
SET GLOBAL rpl_semi_sync_slave_enabled = 1;
```
延迟显著降低，但仍无法完全消除。

**方案三：GTID + 行格式复制**
使用 `binlog_format = ROW`，配合 GTID（全局事务ID），从库可精确追踪主库事务位置。

**方案四：延迟槽位（Delay Slot）**
```
将某个从库特意设置 N 小时延迟（如1小时），用于紧急回滚
CHANGE MASTER TO MASTER_DELAY = 3600;
```

**方案五：应用层按需路由**
- 写操作后返回写入主库的 `last_insert_id`
- 关键读取带上 `as_of_transaction_id` 验证数据版本

**来源：** MySQL 官方文档 Chapter 17.3.9 "Semisynchronous Replication"、Percona "Semi-Sync Replication Setup"

---

## 题目 5：如何设计一个高效的分页查询，避免深度分页问题？

### 参考答案

**深度分页问题**

```sql
-- 问题：偏移量越大，MySQL 越慢
SELECT * FROM orders ORDER BY id LIMIT 1000000, 20;
```
原因：MySQL 需要先扫描前 100 万行，丢弃前 100 万条，才返回 20 条。

**优化方案**

**方案一：游标分页（Keyset Pagination）**
```sql
-- 首次查询
SELECT * FROM orders WHERE id > 0 ORDER BY id LIMIT 20;

-- 下一页：记住上一页最后一条的 id
SELECT * FROM orders WHERE id > 1234567 ORDER BY id LIMIT 20;
```
时间复杂度 O(1)，不受偏移量影响。

**方案二：延迟关联（Deferred Join）**
```sql
-- 子查询只返回主键，再关联获取完整行
SELECT o.* FROM orders o
INNER JOIN (SELECT id FROM orders ORDER BY id LIMIT 1000000, 20) AS t
ON o.id = t.id;
```
利用覆盖索引加速子查询，只对 20 条记录执行回表。

**方案三：倒排分页**
若按时间倒序分页，用 `WHERE created_at < last_seen_time` 替代 `OFFSET`。

**百万级数据分页策略对比**

| 方案 | 第1页 | 第1000页 | 数据一致性 |
|------|-------|---------|-----------|
| `LIMIT offset, size` | 快 | 极慢 | 稳定 |
| 游标分页 | 稍慢 | 恒定快 | 可能跳行 |
| 延迟关联 | 快 | 恒定快 | 稳定 |

**最佳实践：**
- 总数 `< 10000`：可用 `SQL_CALC_FOUND_ROWS` 或前端缓存总数
- 总数 `> 10000` 或数据频繁变更：**必须用游标分页**
- 搜索场景：Elasticsearch / OpenSearch 替代数据库分页

**来源：** MySQL 官方文档 Chapter 8.8 "Optimizing IN and EXISTS Subqueries"、use-the-index-luke.com "Pagination"

---

*📌 本题库由 AI 自动整理，持续更新中。欢迎 Star：https://github.com/qq286158530/interview-questions*
