# 数据库性能优化面试题

> 📅 更新时间：2026-06-11  
> 🐼 出题人：小憨宝  
> 📚 涵盖：MySQL / PostgreSQL 索引优化、查询优化、存储引擎、缓存策略

---

## 题目一：InnoDB 为什么推荐使用自增主键？

### 参考答案

**InnoDB 表的数据组织方式为主键顺序索引（Clustered Index），即数据文件按照主键顺序紧凑存储。** 使用自增主键有以下优势：

1. **插入性能高**：新记录总是追加到数据页末尾，避免了页分裂（Page Split）和数据行移动的开销。
2. **索引占用更小**：主键为整型（INT/BIGINT），非唯一二级索引中存储主键值即可，指针更小，索引树更矮。
3. **缓存友好**：自增主键的顺序写入使得相邻记录大概率落在同一数据页或相邻页，提升 Buffer Pool 的命中率。
4. **减少碎片**：随机主键（如 UUID）会导致数据频繁插入到已有页中间，引发大量页分裂和索引碎片。

**反例**：使用 UUID 或字符串作为主键，每次插入都可能触发 B+ 树的重新平衡（平衡二叉树特性），插入性能可下降 10 倍以上。

> 📎 来源：[MySQL 官方文档 - Clustered Index](https://dev.mysql.com/doc/refman/8.0/en/innodb-data-storage.html)

---

## 题目二：如何定位并优化慢查询？（MySQL 为例）

### 参考答案

**Step 1：开启慢查询日志**
```sql
-- 临时开启（会话级）
SET GLOBAL slow_query_log = 'ON';
SET GLOBAL long_query_time = 1;  -- 超过1秒记录
SET GLOBAL slow_query_log_file = '/var/log/mysql/slow.log';
```

**Step 2：使用 EXPLAIN 分析执行计划**
```sql
EXPLAIN SELECT ... FROM ...
EXPLAIN ANALYZE ...  -- MySQL 8.0+，显示实际运行信息
```

重点关注字段：
- `type`：最好达到 `ref/range`，避免 `ALL`（全表扫描）
- `key`：实际使用的索引
- `rows`：扫描行数估算
- `Extra`：避免出现 `Using filesort`、`Using temporary`

**Step 3：建立合适的索引**
- 针对 `WHERE`、`ORDER BY`、`GROUP BY` 涉及的列建立复合索引
- 遵循最左前缀原则
- 避免在索引列上使用函数或做运算

**Step 4：业务层优化**
- 分页优化：`WHERE id > last_id LIMIT n` 替代 `OFFSET`
- 减少 JOIN，使用延迟关联
- 读写分离，分担压力

> 📎 来源：[MySQL 慢查询日志配置](https://dev.mysql.com/doc/refman/8.0/en/slow-query-log.html)

---

## 题目三：PostgreSQL 的 MVCC 机制是如何工作的？它对性能有何影响？

### 参考答案

**MVCC（Multi-Version Concurrency Control）** 让每个事务看到的数据库快照一致，写操作不阻塞读操作。

**PostgreSQL 实现细节：**
- 每行数据有两个隐藏列：`xmin`（插入事务ID）和 `xmax`（删除/更新事务ID）
- UPDATE 不原地修改，而是标记旧行为 `xmax`，插入新行（物理上会产生行版本膨胀）
- 事务提交后通过 `VACUUM` 清理死元组，释放空间

**对性能的影响：**

| 影响 | 说明 |
|------|------|
| ✅ 读写不阻塞 | 大幅提升并发能力 |
| ⚠️ 空间膨胀 | 频繁更新会产生大量dead tuple |
| ⚠️ 查询膨胀 | 长事务会阻止 VACUUM 清理，导致表臃肿 |
| ⚠️ 索引膨胀 | 同样会产生索引碎片 |

**优化建议：**
```sql
-- 配置自动清理
ALTER SYSTEM SET autovacuum = ON;
ALTER SYSTEM SET autovacuum_vacuum_threshold = 50;
ALTER SYSTEM SET autovacuum_analyze_threshold = 50;

-- 手动清理大表
VACUUM ANALYZE big_table;
```

> 📎 来源：[PostgreSQL MVCC 官方文档](https://www.postgresql.org/docs/current/mvcc.html)

---

## 题目四：如何设计一个高效的数据库缓存策略？

### 参考答案

**缓存层次设计：**

```
请求 → 应用层缓存(Redis/Memcached) → 数据库
         ↓ 未命中
      数据库查询 → 写入缓存 → 返回
```

**核心策略：**

1. **缓存粒度控制**
   - 推荐缓存「热点查询结果」而非单行数据
   - 例如：用户资料 → 缓存整个用户对象，而非每个字段单独缓存

2. **Key 命名规范**
   ```
   {业务}:{实体}:{标识}:{版本}
   例如：shop:product:123:v2
   ```

3. **失效策略**
   - **LRU（Least Recently Used）**：内存满时淘汰最久未使用的
   - **TTL（Time To Live）**：设置过期时间，MySQL/Redis 均支持
   - **主动删除**：数据变更时删除对应缓存（Cache Aside 模式）

4. **缓存穿透、击穿、雪崩应对**
   - **穿透**：布隆过滤器 + 空值缓存
   - **击穿**：互斥锁 / 永不过期策略
   - **雪崩**：过期时间加随机偏移量 + 熔断降级

5. **MySQL Query Cache 的坑**（MySQL 8.0 已移除）
   - Query Cache 在多核下存在全局锁竞争，MySQL 8.0 直接移除
   - 建议使用应用层 Redis 替代

> 📎 来源：[Redis 缓存策略最佳实践](https://redis.io/docs/manual/patterns/)

---

## 题目五：MySQL 半同步复制（Semi-Sync Replication）是什么？它解决了什么问题？

### 参考答案

**背景问题：**
- 异步复制模式下，主库提交事务后立即返回客户端成功，不等从库落盘
- 若主库宕机且从库未同步，已提交事务会**丢数据**

**半同步复制原理：**
```
主库提交 → 等待至少1个从库确认收到Relay Log → 返回客户端
```

**工作流程：**
1. 主库事务提交后，将事件写入二进制日志（Binlog）
2. 主库等待从库响应（通过 `ACK` 确认）
3. 收到至少1个从库的 ACK 后，返回执行结果给客户端
4. 若等待超时（默认 10s），自动降级为异步复制

**相关配置：**
```sql
-- 主库安装插件
INSTALL PLUGIN rpl_semi_sync_master SONAME 'semisync_master.so';
SET GLOBAL rpl_semi_sync_master_enabled = ON;

-- 从库安装插件
INSTALL PLUGIN rpl_semi_sync_slave SONAME 'semisync_slave.so';
SET GLOBAL rpl_semi_sync_slave_enabled = ON;
```

**解决了什么问题：**
- ✅ 至少有一个从库持有数据副本（1arca 提交），主从切换时不丢数据
- ✅ 满足金融级业务对数据一致性的要求
- ⚠️ 增加了事务提交延迟（等待ACK），不适合对延迟极度敏感的场景

> 📎 来源：[MySQL 半同步复制官方文档](https://dev.mysql.com/doc/refman/8.0/en/replication-semisync.html)

---

## 📌 小结

| 知识点 | 核心关键词 |
|--------|-----------|
| 索引设计 | 自增主键、复合索引、最左前缀 |
| 慢查询优化 | EXPLAIN、覆盖索引、避免filesort |
| PostgreSQL | MVCC、VACUUM、autovacuum |
| 缓存策略 | Redis、TTL、LRU、缓存击穿/穿透/雪崩 |
| 主从复制 | 半同步复制、ACK、Binlog |

> 📂 本题库已推送至 GitHub：[qq286158530/interview-questions](https://github.com/qq286158530/interview-questions)
