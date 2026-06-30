# 数据库性能优化面试题

> 📅 整理日期：2026-06-30  
> 🗂 分类：MySQL / PostgreSQL 性能优化  
> 来源：综合整理（含互联网公开资源）

---

## 题目一：MySQL 索引失效的场景有哪些？如何避免？

### 参考答案

**常见索引失效场景：**

1. **使用 LEFT JOIN / RIGHT JOIN 而非 INNER JOIN 时**，如果连接列上有索引但优化器选择全表扫描，索引失效。
2. **WHERE 子句中使用函数或表达式**，如 `WHERE YEAR(create_time) = 2024` 会导致索引列上的函数运算使索引失效。
3. **WHERE 子句中对索引列进行隐式类型转换**，例如字段是 VARCHAR 但传入数字 `WHERE phone = 13800138000`（MySQL 会将字符串转数字）。
4. **使用 LIKE 前缀通配符**，`WHERE name LIKE '%张%'` 无法使用 B-Tree 索引，只能用全文索引。
5. **多列索引未使用最左前缀**，`INDEX(a, b, c)` 但查询只有 `WHERE b = 1` 或 `WHERE c = 1`。
6. **使用 OR 连接条件**，且 OR 两侧字段索引不一致时，部分场景会退化为全表扫描。
7. **表数据过小时**，优化器认为全表扫描更快而放弃索引。

**避免策略：**
- 尽量使用 INNER JOIN，或确保连接列两边都有索引
- 将函数运算移至等号右侧，如 `WHERE create_time >= '2024-01-01' AND create_time < '2025-01-01'`
- 避免隐式类型转换，保证类型一致
- 尽量使用 LIKE 后缀 `WHERE name LIKE '张%'`
- 遵循最左前缀原则，合理设计复合索引
- 使用 EXPLAIN 分析执行计划，确认索引是否被使用

**来源：** 综合整理（互联网公开面试题库）

---

## 题目二：MySQL InnoDB 与 MyISAM 存储引擎的区别？如何选择？

### 参考答案

| 特性 | InnoDB | MyISAM |
|------|--------|--------|
| **事务支持** | 支持 ACID 事务，提供 commit/rollback | 不支持事务 |
| **行级锁** | 支持行级锁，并发性能高 | 只支持表级锁 |
| **外键约束** | 支持外键 | 不支持外键 |
| **崩溃恢复** | 支持 MVCC + redo log，崩溃恢复能力极强 | 崩溃恢复能力弱 |
| **全文索引** | MySQL 5.6+ 支持 | 支持全文索引 |
| **存储结构** | 聚簇索引（主键索引和数据行在一起） | 非聚簇索引（索引与数据分离） |
| **COUNT(*)** | 无特殊优化，全表扫描计数 | 有专门计数器，快速返回 |
| **适用场景** | 高并发、事务需求、数据可靠性要求高 | 以读为主、很少更新删除、表级锁可接受 |

**选择建议：**
- 绝大多数业务场景选 **InnoDB**（MySQL 5.5+ 默认引擎）
- 只有在确定只需要表级锁、且读多写少、且不需要事务时，才考虑 MyISAM
- MySQL 8.0 已废弃 MyISAM，新项目不要使用

**来源：** 综合整理

---

## 题目三：如何分析一条 SQL 的性能问题？说说 EXPLAIN 的关键字段。

### 参考答案

**分析步骤：**

1. 使用 `EXPLAIN` 或 `EXPLAIN ANALYZE`（MySQL 8.0+）查看执行计划
2. 关注以下关键字段：

| 字段 | 含义 | 关注点 |
|------|------|--------|
| **type** | 访问类型 | 最好的是 `const/eq_ref/ref`，最差是 `ALL`（全表扫描） |
| **key** | 实际使用的索引 | 如果是 NULL 说明没走索引 |
| **rows** | 预计扫描行数 | 越大性能越差 |
| **Extra** | 额外信息 | 避免出现 `Using filesort`、`Using temporary` |
| **possible_keys** | 可能使用的索引 | 为空说明没有可用索引 |
| **select_type** | 查询类型 | SIMPLE/PRIMARY/SUBQUERY 等 |

**常见性能问题 Extra 提示：**
- `Using filesort`：filesort 是 MySQL 额外排序，无法利用索引，需要优化 SQL 或添加合适索引
- `Using temporary`：创建临时表，常见于 GROUP BY、DISTINCT、ORDER BY 组合使用
- `Using index condition`：下推索引过滤，减少回表次数，性能较好
- `Using index`：索引覆盖，所有字段均在索引中，无需回表，性能最优

**优化手段：**
- 添加合适索引
- 避免 SELECT *，尽量只查需要的字段
- 利用索引覆盖（covering index）
- 避免隐式类型转换和函数运算

**来源：** 综合整理

---

## 题目四：PostgreSQL 与 MySQL 在性能优化上的核心差异是什么？

### 参考答案

**1. 索引机制差异**
- MySQL 的 InnoDB 使用 B+Tree 作为默认索引；PostgreSQL 默认使用 B-Tree，但支持更多索引类型（GIN、GiST、SP-GiST、Hash、BRIN 等）
- PostgreSQL 支持**表达式索引**和**部分索引**，在某些场景下比 MySQL 更灵活
- PostgreSQL 支持 **JSON 路径索引**（jsonb 类型），MySQL 5.7+ 也有 JSON 支持但索引能力较弱

**2. 查询优化器**
- PostgreSQL 使用基于成本的优化器（CBO），可调节 `random_page_cost`、`seq_page_cost` 等参数适配 SSD/HDD
- MySQL 优化器相对简单，在复杂查询上可能选择次优执行计划
- PostgreSQL 支持 **SQL 高级特性**（CTE、窗口函数、递归查询），优化器会合理处理

**3. MVCC（多版本并发控制）**
- InnoDB 和 PostgreSQL 都使用 MVCC，但实现方式不同
- PostgreSQL 使用 **VACUUM** 机制清理旧版本，MySQL 使用 purge 线程
- PostgreSQL 的 VACUUM 需要定期执行，否则可能导致膨胀（bloat）

**4. 缓存策略**
- PostgreSQL 依赖 OS 层面的页缓存（shared_buffers 是元数据缓存，不是数据缓存）
- MySQL InnoDB 有独立的 buffer pool 缓存数据页和索引
- PostgreSQL 的 `pg_prewarm` 扩展可以预热缓存

**5. 并发控制**
- PostgreSQL 支持** advisory lock**、**行级锁**、**表级锁**，锁粒度更细
- PostgreSQL 的**序列化快照隔离（SSI）** 是真正的串行化隔离级别，比 MySQL 的 SERIALIZABLE 性能更好

**来源：** 综合整理

---

## 题目五：如何设计一个高效的数据库缓存策略？Redis 与 MySQL 如何配合使用？

### 参考答案

**缓存读写策略（Cache-Aside 最常用）：**

```
读操作：
1. 先查 Redis 缓存
2. 命中 → 返回数据
3. 未命中 → 查 MySQL → 写入 Redis → 返回数据

写操作：
1. 写 MySQL 主库
2. 删除（而非更新）Redis 缓存（避免并发导致数据不一致）
3. 下次读取时自动回填
```

**缓存问题及解决方案：**

| 问题 | 解决方案 |
|------|---------|
| **缓存穿透**（查询不存在的数据） | 布隆过滤器 / 空值缓存 |
| **缓存击穿**（热点 key 过期瞬间大量请求） | 互斥锁 / 永不过期 + 异步更新 |
| **缓存雪崩**（大量 key 同时过期） | 过期时间加随机值 / 多级缓存 |
| **数据不一致** | 延迟双删 / Canal 订阅 binlog |

**Redis 集群方案：**
- **主从复制**：读写分离，主库写从库读
- **Sentinel 哨兵**：主从切换，保证高可用
- **Cluster 模式**：数据分片（16384 个 slot），支持大规模横向扩展

**缓存Key设计建议：**
- 格式：`业务:表:ID:字段`（如 `user:profile:1001`）
- 设置合理 TTL，过期策略根据业务数据更新频率决定
- 大字段（如用户详情 JSON）压缩后再存入 Redis（`SET json_data COMPRESSED`）

**来源：** 综合整理（互联网公开面试题库）

---

> 📌 每道题目均经过筛选，力求覆盖数据库性能优化核心知识点。  
> 建议结合自身项目经验深入理解，而非死记硬背答案。