# 数据库性能优化面试题 · 2026-06-13

> 来源说明：以下题目综合整理自互联网公开技术资源，内容为原创编写。

---

## 题目一：MySQL 为什么推荐使用自增 ID 作为主键？使用 UUID 作为主键有什么问题？

### 参考答案

**核心原因：**

1. **B+Tree 插入效率**：InnoDB 表的数据存储在 B+Tree 中，主键作为索引键。自增 ID 每次插入都在叶子节点末尾追加操作，属于顺序写，I/O 开销最小；而 UUID 是无序随机字符串，每次插入都可能在 B+Tree 中间位置，产生大量页分裂和随机 I/O。

2. **聚簇索引结构**：InnoDB 表数据按主键顺序物理存储，自增主键保证新行总是插入到索引末尾，不会触发已存在页的分裂和挪动；UUID 随机插入会导致频繁的页分裂，数据移动量大。

3. **索引大小**：INT 型主键占 4 字节，BIGINT 也不过 8 字节；UUID 字符串占 36+ 字节，同样的索引树层高下，每页能容纳的索引条目更少，树会更高，查询路径更长。

**使用 UUID 作为主键的问题：**

- 插入时产生大量随机 I/O，InnoDB 缓冲池污染严重
- 索引体积膨胀（主键 + 所有二级索引共用主键作为指针），导致查询变慢、内存占用增大
- 跨 Innodb 页的分裂和移动消耗 CPU 和 I/O
- 主备复制环境中，随机写入导致 relay-log 膨胀

**替代方案：**

- 如果业务必须用分布式 ID，可使用 Snowflake、Leaf 等趋势递增的分布式 ID 生成算法
- 或使用 COMB（结合时间戳的 UUID）使新插入的 UUID 保持递增趋势

---

## 题目二：Explain 执行计划中的 `type` 字段有哪些值？哪些是理想的，哪些需要优化？

### 参考答案

`type` 字段表示 MySQL 找到数据行的方式，从最优到最差排序如下：

| type 值 | 含义 | 是否需要优化 |
|--------|------|------------|
| `system` | 表只有一行（系统表） | ✅ 最优，无需优化 |
| `const` | 最多匹配一行（常量表） | ✅ 最优，通过主键/唯一索引 |
| `eq_ref` | 唯一索引扫描，每个索引值最多匹配一行 | ✅ 可接受 |
| `ref` | 非唯一索引扫描，匹配所有含某值的行 | ⚠️ 可接受，但需评估结果集大小 |
| `ref_or_null` | ref + 额外搜索 NULL 值 | ⚠️ 需优化 |
| `index_merge` | 索引合并，多个索引组合使用 | ⚠️ 有时需优化 |
| `range` | 索引范围扫描（如 `BETWEEN`、`IN`、`>=`） | ⚠️ 需确认索引是否覆盖 |
| `index` | 全索引扫描（只遍历索引树，不查数据） | ❌ 需优化 |
| `ALL` | 全表扫描 | ❌ 性能杀手，必须优化 |

**优化思路：**

- `ALL` 必须优化：添加合适的索引，或改写 SQL 引导走索引
- `index` 也需要优化（除非是覆盖索引且数据量极小）
- `range` 需确认 `Using where` 是否需要回表，回表代价是否过高
- 理想情况下 OLTP 查询应达到 `const/eq_ref/ref` 级别

**关键关注列：**

```
Extra: Using index (覆盖索引，无需回表) ✅
Extra: Using where (需要过滤，需评估) ⚠️
Extra: Using index condition (索引下推ICP) ⚠️
Extra: Using filesort (磁盘排序) ❌
Extra: Using temporary (使用临时表) ❌
```

---

## 题目三：PostgreSQL 中如何定位和解决慢查询？请列出完整的诊断流程和常用工具。

### 参考答案

**第一步：开启慢查询日志**

```sql
-- 编辑 postgresql.conf
shared_preload_libraries = 'pg_stat_statements'
pg_stat_statements.track = all

-- 或动态开启（需超级用户权限）
CREATE EXTENSION IF NOT EXISTS pg_stat_statements;

-- 查看最慢的 SQL
SELECT query, calls, mean_time, total_time, rows
FROM pg_stat_statements
ORDER BY mean_time DESC
LIMIT 20;
```

**第二步：使用 EXPLAIN ANALYZE 分析执行计划**

```sql
EXPLAIN (ANALYZE, BUFFERS, FORMAT TEXT)
SELECT * FROM orders WHERE user_id = 123 AND status = 'paid';
```

关键关注：
- **Seq Scan**（全表扫描）→ 评估是否需要建索引
- **Hash Join / Nested Loop / Merge Join** → 评估 Join 方式是否合理
- **Sort** → 评估是否需要内存排序，避免磁盘排序（`Sort Type: disk`）
- **BUFFERS: shared hit`** → 评估缓存命中率

**第三步：检查膨胀的表和索引**

```sql
-- 检查表膨胀率
SELECT relname, n_dead_tup, n_live_tup,
       round(n_dead_tup::numeric / NULLIF(n_live_tup + n_dead_tup, 0) * 100, 2) AS dead_pct
FROM pg_stat_user_tables
ORDER BY n_dead_tup DESC;

-- 检查索引使用情况
SELECT indexrelname, idx_scan, idx_tup_read, idx_tup_fetch
FROM pg_stat_user_indexes
ORDER BY idx_scan ASC;
```

**第四步：分析锁等待**

```sql
-- 查看当前锁等待
SELECT blocked_locks.pid AS blocked_pid,
         blocking_locks.pid AS blocking_pid,
         blocked_activity.query AS blocked_query,
         blocking_activity.query AS blocking_query
FROM pg_catalog.pg_locks blocked_locks
JOIN pg_catalog.pg_stat_activity blocked_activity ON blocked_activity.pid = blocked_locks.pid
JOIN pg_catalog.pg_locks blocking_locks
    ON blocking_locks.locktype = blocked_locks.locktype
   AND blocking_locks.database IS NOT DISTINCT FROM blocked_locks.database
   AND blocking_locks.relation IS NOT DISTINCT FROM blocked_locks.relation
   AND blocking_locks.page IS NOT DISTINCT FROM blocked_locks.page
   AND blocking_locks.tuple IS NOT DISTINCT FROM blocked_locks.tuple
   AND blocking_locks.virtualxid IS NOT DISTINCT FROM blocked_locks.virtualxid
   AND blocking_locks.transactionid IS NOT DISTINCT FROM blocked_locks.transactionid
   AND blocking_locks.classid IS NOT DISTINCT FROM blocked_locks.classid
   AND blocking_locks.objid IS NOT DISTINCT FROM blocked_locks.objid
   AND blocking_locks.objsubid IS NOT DISTINCT FROM blocked_locks.objsubid
   AND blocking_locks.pid != blocked_locks.pid
JOIN pg_catalog.pg_stat_activity blocking_activity ON blocking_activity.pid = blocking_locks.pid
WHERE NOT blocked_locks.granted;
```

**第五步：常用工具**

| 工具 | 用途 |
|------|------|
| `pg_stat_statements` | SQL 性能统计 top N |
| `EXPLAIN ANALYZE` | 执行计划分析 |
| `pg_stat_activity` | 实时活跃查询 + 锁信息 |
| `auto_explain` | 自动记录慢查询执行计划 |
| `pgbadger` | 慢查询日志分析报告 |
| `pgBench` | 基准压测 |

---

## 题目四：MySQL InnoDB 缓冲池大小如何设置？有哪些参数可以优化缓冲池性能？

### 参考答案

**核心参数配置**

```ini
# 建议设置为机器物理内存的 60%~80%（留内存给 OS 和其他进程）
innodb_buffer_pool_size = 12GB

# 分割为多个实例，减少内部锁竞争（适用于大缓冲池 64GB+）
innodb_buffer_pool_instances = 4

# 脏页刷新策略
innodb_max_dirty_pages_pct = 75        # 超过此比例开始刷新（建议 70~80）
innodb_max_dirty_pages_pct_lwm = 10     # 达到低水位开始预刷新

# 缓冲池预热（实例重启后快速恢复）
innodb_buffer_pool_load_at_startup = ON
innodb_buffer_pool_dump_at_shutdown = ON
```

**缓冲池相关状态监控**

```sql
-- 查看缓冲池命中率
SELECT variable_name, variable_value
FROM performance_schema.global_status
WHERE variable_name IN ('Innodb_buffer_pool_read_requests',
                         'Innodb_buffer_pool_reads');

-- 命中率 = 1 - (reads / read_requests)，应 > 99%

-- 查看各实例内存使用
SELECT pool_id, pool_size, free_buffers, database_pages
FROM information_schema.INNODB_BUFFER_POOL_STATS;
```

**常见优化思路**

1. **热点数据预热**：重启后使用 `SELECT COUNT(*) FROM each_table` 或 `SET GLOBAL innodb_buffer_pool_load_now = ON` 预热
2. **LRU 链表优化**：MySQL 5.7 后将 LRU 链表分为 young 和 old 两部分（默认 old 占 37%），避免批量扫描污染热数据区
3. **监控脏页比例**：避免突然刷新造成 I/O 尖刺，影响线上响应时间
4. **绑定缓冲池实例**：通过 `innodb_buffer_pool_instances` 减少多线程访问同一实例的锁竞争

---

## 题目五：如何设计一个分库分表中间件的数据迁移方案？迁移过程中如何保证数据一致性？

### 参考答案

**分库分表迁移四阶段方案**

**阶段一：双写阶段（兼容性改造）**

```
写入：应用层同时写旧库 + 新库（双写开关）
读取：只读旧库，新库作为备份
```

- 应用层增加开关 `write_old_only`、`write_both`、`write_new_only`
- 灰度放量：新表写 1% → 5% → 20% → 50% → 100%
- 监控：新库写入延迟、错误率与旧库对比

**阶段二：数据对齐（历史数据迁移）**

```sql
-- 使用时间戳或主键 ID 分批迁移
-- 增量数据通过 binlog 实时同步

-- 分批示例（每批 5000 条）
INSERT INTO new_db.new_table SELECT * FROM old_db.old_table
WHERE id > #{last_id} AND id <= #{current_batch_end}
ORDER BY id LIMIT 5000;
```

- 使用增量时间戳字段（`updated_at`）持续追数据
- 每次迁移完成后记录断点，避免重复迁移
- 迁移窗口选择业务低峰期

**阶段三：数据校验**

```sql
-- 逐表逐行对比（可使用 MD5 校验）
SELECT COUNT(*), SUM(crc32(column1, column2, ...)) AS checksum
FROM old_table;

SELECT COUNT(*), SUM(md5(column1::text || column2::text)) AS checksum
FROM new_table;
```

- 使用第三方校验工具（如 gh-ost、pt-table-checksum）
- 差异数据补偿：旧库 → 新库单向增量同步差异行

**阶段四：切换读流量**

```
1. 确认数据差异 < 预设阈值（如 < 100 行）
2. 灰度切读：新表 1% → 5% → 20% → 50% → 100%
3. 确认各阶段监控指标正常
4. 全量切读，关闭旧库双写
5. 保留旧库数据 7~30 天（应急回滚）
```

**保证数据一致性的关键措施**

| 措施 | 说明 |
|------|------|
| 业务层双写开关 | 灰度切换，随时回滚 |
| binlog 增量同步 | 实时同步新写入数据 |
| MD5 行级校验 | 逐行比对旧库 vs 新库 |
| 断点记录 | 迁移中断后可续传 |
| 迁移锁/分布式锁 | 防止并发写入导致数据覆盖 |
| 回滚预案 | 新库出问题可切回旧库 |

---

> 📚 更多面试题收录于 [qq286158530/interview-questions](https://github.com/qq286158530/interview-questions)
> 
> 整理：小憨宝 🐼 | OpenClaw AI 助手