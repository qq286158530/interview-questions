# 数据库性能优化面试题

> 日期：2026-07-31
> 涵盖：索引优化、查询优化、存储引擎、缓存策略等核心知识点

---

## 题目一：MySQL B+树索引原理与最左前缀匹配原则

**问题：** 请解释 MySQL 中 B+树 索引的工作原理，以及什么是最左前缀匹配原则？在实际开发中，如何合理设计联合索引？

**答案：**

### B+树索引原理

B+树是一种多路平衡搜索树，是 MySQL InnoDB 存储引擎默认的索引数据结构。相比二叉树，B+树通过增加叉数降低树的高度，从而减少磁盘 IO 次数。

**结构特点：**
- 叶子节点包含所有数据，非叶子节点只存储索引列的值（作为路由）
- 叶子节点之间通过双向链表连接，天然支持范围查询
- 树的高度通常为 3~4 层，可支撑千万级数据仅需 3 次磁盘 IO

**为什么不用二叉树？**
访问磁盘的成本约为访问内存的 10 万倍。二叉树在数据量大时树高很高，意味着大量磁盘 IO。B+树通过多叉设计，每层节点可容纳更多数据项，大幅降低树高。

### 最左前缀匹配原则

联合索引 (a, b, c) 的数据结构按 a → b → c 的顺序从左到右建立搜索树。

**匹配规则：**
- 查询条件必须从最左边的列开始，才能使用索引
- `WHERE a = 1 AND b = 2` → 可用索引
- `WHERE a = 1 AND c = 3` → 可用索引（只用到 a）
- `WHERE b = 2 AND c = 3` → **无法使用索引**（跳过 a）
- `WHERE a = 1 AND b > 2 AND c = 3` → a 和 b 可用索引，但 b 的范围查询导致 c 不可用

### 联合索引设计原则

1. **区分度高的列放前面**：区分度 = `COUNT(DISTINCT col) / COUNT(*)`，比例越大扫描行数越少
2. **等值查询优先于范围查询**：范围查询会中断后续列的索引匹配
3. **索引列不参与计算**：`WHERE YEAR(create_time) = 2024` 无法使用索引，应写成 `create_time BETWEEN '2024-01-01' AND '2024-12-31'`
4. **覆盖索引**：如果查询的列都在索引中，可避免回表，进一步提升性能

---

## 题目二：慢查询优化思路与 EXplain 命令解读

**问题：** 当发现一条 SQL 执行很慢时，你的排查和优化思路是什么？请解释 `EXPLAIN` 命令输出中 `type`、`rows`、`Extra` 字段的含义。

**答案：**

### 慢查询优化基本步骤

1. **开启慢查询日志**或使用 `SQL_NO_CACHE` 先运行一遍，确认是否真的很慢
2. **单表分析**：对 WHERE 条件中的字段逐个分析，优先选择区分度高的字段加索引
3. **EXPLAIN 分析执行计划**：确认是否按预期使用索引，是否从锁定记录最少的表开始查询
4. **业务场景分析**：了解数据分布，有些区分度低的字段在特定业务场景下反而高效（如状态字段在业务保证数据不平衡时）
5. **参考建索引原则**添加或调整索引
6. **验证效果**，不符合预期则回到步骤 0 重新分析

### EXPLAIN 关键字段解读

| 字段 | 含义 | 优化目标 |
|------|------|----------|
| **type** | 连接类型，反映查询效率 | 最好达到 `ref`、`eq_ref`，避免 `ALL`（全表扫描） |
| **rows** | 预计扫描的行数 | 越小越好，是优化的核心指标 |
| **Extra** | 额外信息 | 避免 `Using filesort`、`Using temporary` |

**type 从好到差排序：**
```
system > const > eq_ref > ref > range > index > ALL
```

- `const`：主键或唯一索引的等值查询，最多返回一条记录
- `eq_ref`：JOIN 时，主键或唯一索引被驱动表使用
- `ref`：普通索引等值查询
- `range`：索引范围查询（ BETWEEN、IN、>、< 等）
- `ALL`：全表扫描，性能最差

**Extra 常见值：**
- `Using index`：使用了覆盖索引，无需回表
- `Using where`：在存储引擎层之后用 WHERE 过滤
- `Using temporary`：需要创建临时表，常见于 GROUP BY、ORDER BY 与索引不匹配
- `Using filesort`：无法利用索引排序，需要额外的排序操作

---

## 题目三：MySQL InnoDB 与 MyISAM 存储引擎的区别

**问题：** MySQL 常见存储引擎 InnoDB 和 MyISAM 有哪些区别？在实际项目中如何选择？

**答案：**

| 特性 | InnoDB | MyISAM |
|------|--------|--------|
| **事务支持** | 支持 ACID 事务（ COMMIT/ROLLBACK） | 不支持事务 |
| **锁粒度** | 行级锁（并发高） | 表级锁（并发低） |
| **外键约束** | 支持外键 | 不支持外键 |
| **MVCC** | 支持多版本并发控制 | 不支持 |
| **崩溃恢复** | 自动崩溃恢复（redo log） | 需手动修复（myisamchk） |
| **索引结构** | B+树（聚簇索引） | B+树（非聚簇索引） |
| **COUNT(*)** | 全表扫描（无内置计数器） | 维护计数器，性能好 |
| **适用场景** | 写密集、事务需求、高并发 | 读密集、不需要事务、日志型 |

### 核心区别详解

**1. 聚簇索引 vs 非聚簇索引**
- InnoDB：数据文件本身就是按 B+树组织的索引结构，叶子节点直接存储行数据（聚簇索引）
- MyISAM：数据文件和索引文件分离，叶子节点存储的是数据地址（非聚簇索引）

**2. 事务与崩溃恢复**
- InnoDB 通过 redo log 和 undo log 实现事务和崩溃恢复
- MyISAM 没有日志机制，崩溃后可能丢失数据或损坏

**3. 并发处理**
- InnoDB 行级锁 + MVCC 支持高并发写入
- MyISAM 表级锁在并发写入时容易阻塞

### 选型建议

- **需要事务、高并发、数据可靠性** → 选择 InnoDB（互联网公司默认选项）
- **只读场景、需要快速 COUNT(*)** → 可考虑 MyISAM
- **日志系统、只读报表** → MyISAM 可用，但现代方案更推荐 InnoDB + 读写分离

---

## 题目四：数据库缓存策略 — Redis 与 MySQL 如何配合使用

**问题：** 在高并发场景下，如何使用 Redis 配合 MySQL 实现缓存策略？Cache Aside、Read Through、Write Through 三种模式有什么区别？

**答案：**

### 缓存策略模式

#### 1. Cache Aside（旁路缓存）— 最常用

**读：** 应用先查 Redis，命中则返回；未命中则查 MySQL 并写入 Redis。

**写：** 先更新 MySQL，再删除（而非更新）Redis 中的缓存。

> 为什么删除而不是更新？因为更新操作在并发场景下容易产生脏数据（比如先更新数据库再更新缓存，中间有其他请求读到了旧缓存）。

**优点：** 实现简单，命中率低时影响小
**缺点：** 首次命中时会产生两次查询

#### 2. Read Through（读穿透）

应用只与缓存交互，缓存负责从数据库加载数据。应用不感知数据库。

**流程：** 应用 → Redis → （未命中）→ 自动从 MySQL 加载 → 返回

#### 3. Write Through（写穿透）

写入时同时更新缓存和数据库，缓存和数据库保持强一致。

**缺点：** 写入延迟高，实际很少使用

### 缓存经典问题

**1. 缓存穿透：** 查询不存在的数据绕过缓存直接打满数据库
   - 解决：布隆过滤器 / 缓存空值 / 互斥锁

**2. 缓存击穿：** 热点 key 过期瞬间，大量请求打垮数据库
   - 解决：互斥锁 / 热点数据永不过期 / 逻辑过期

**3. 缓存雪崩：** 大量 key 集中过期或 Redis 宕机
   - 解决：随机过期时间 / Redis 高可用 / 限流降级

**4. 缓存与数据库双写不一致：**
   - 在 Cache Aside 模式下，延迟删除策略可能导致短暂不一致
   - 强一致场景建议：分布式锁 + 延迟双删（先删 Redis，再更新 MySQL，延迟再删 Redis）

### 缓存选型建议

- **读多写少** → Redis 缓存，大幅降低数据库压力
- **写多读少** → 不建议缓存，反而增加复杂度
- **数据一致性要求高** → 不要缓存，或使用 Write Through + 短 TTL

---

## 题目五：PostgreSQL 与 MySQL 在性能优化上的差异

**问题：** PostgreSQL 和 MySQL 在索引机制、查询优化器、并发控制方面有哪些不同？如果将业务从 MySQL 迁移到 PostgreSQL，需要注意哪些性能优化点？

**答案：**

### 核心差异对比

| 维度 | MySQL (InnoDB) | PostgreSQL |
|------|---------------|------------|
| **索引类型** | B+Tree（主要）、R-Tree（空间）、Full-text、HASH | B-Tree、R-Tree、GIN（倒排）、GiST、BRIN、覆盖索引 |
| **MVCC** | 简化版（只支持隔离级别） | 完整 MVCC（支持所有隔离级别） |
| **查询优化器** | 相对简单，优化规则少 | 成本优化模型成熟，支持更多统计信息 |
| **并行查询** | 8.0+ 支持部分并行 | 原生支持并行查询（并行 Seq Scan、Index Scan） |
| **表分区** | 支持，但语法和功能较有限 | 支持 LIST、RANGE、HASH 分区，功能更完善 |
| **连接方式** | NL Join、Hash Join、Sort Merge Join | NL Join、Hash Join、Sort Merge Join + 很多变种 |

### PostgreSQL 独特的性能优化手段

**1. GIN 索引 — 适合数组、全文检索、JSONB**
```sql
CREATE INDEX idx_tags ON posts USING GIN (tags);
-- tags 为 text[] 数组类型，可高效查询包含某标签的文章
```

**2. BRIN 索引 — 适合物理有序的大表**
```sql
CREATE INDEX idx_created ON orders USING BRIN (created_at);
-- 对于按时间顺序插入的表，BRIN 体积小且查询效率高
```

**3. 表达式索引**
```sql
CREATE INDEX idx_lower_email ON users (LOWER(email));
-- 查询时不需要对每行执行 LOWER() 函数
```

**4. 并行查询**
PostgreSQL 可自动并行执行 Seq Scan、Hash Join、Aggregate 等操作，在多核 CPU 场景下效果显著。可通过 `max_parallel_workers_per_gather` 调整并行度。

**5. 统计信息与执行计划**
```sql
-- 手动更新统计信息，确保优化器有最新数据
ANALYZE table_name;

-- 查看执行计划
EXPLAIN (ANALYZE, BUFFERS, FORMAT TEXT) SELECT ...
```

### 迁移注意事项

1. **字符集**：PostgreSQL 默认 UTF-8，MySQL 需确认字符集一致性
2. **自增主键**：MySQL 用 `AUTO_INCREMENT`，PostgreSQL 用 `SERIAL` 或 `GENERATED ALWAYS AS IDENTITY`
3. **分页查询**：MySQL 可用 `LIMIT offset OFFSET`，PostgreSQL 推荐使用游标（Keyset Pagination）避免深分页性能问题
4. **HASH 索引 vs B+Tree**：PostgreSQL 的 HASH 索引不支持范围查询，迁移时需注意
5. **连接池**：MySQL 常用 Druid/HikariCP，PostgreSQL 推荐 PgBouncer（事务级连接池）

---

> 📚 **参考来源**
> 1. [美团技术团队 - MySQL索引原理及慢查询优化](https://tech.meituan.com/2014/06/30/mysql-index.html)
> 2. [MySQL 官方文档 - EXPLAIN Output Format](https://dev.mysql.com/doc/refman/8.0/en/explain-output.html)
> 3. [PostgreSQL 官方文档 - Index Types](https://www.postgresql.org/docs/current/indexes-types.html)
> 4. [Redis 设计与实现 - 缓存策略](https://github.com/redisbook/redisbook)
> 5. [数据库内核月报 - MySQL vs PostgreSQL 对比](https://mysql.taobao.org/monthly/)
