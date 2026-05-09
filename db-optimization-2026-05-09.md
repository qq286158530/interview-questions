# 数据库性能优化面试题

> 📅 整理日期：2026-05-09
> 🐘 数据库：MySQL / PostgreSQL
> 🔗 仓库：https://github.com/qq286158530/interview-questions

---

## 题目一：MySQL 中 B+ 树索引的结构是怎样的？为什么比 B 树更适合数据库？

### 参考答案

**B+ 树索引结构：**

B+ 树是一种多路平衡查找树，所有数据都存储在叶子节点，非叶子节点只存储索引键和指针。叶子节点之间通过双向链表连接（MySQL InnoDB 引擎）。

```
        [50]
       /    \
  [20,30]  [70,90]
   / | \     / | \
叶子层: 数据页... ←→ 叶子层: 数据页... ←→ 叶子层...
```

**B+ 树比 B 树更适合数据库的原因：**

| 特性 | B+ 树 | B 树 |
|------|-------|------|
| 查询稳定性 | 所有查询最终都到达叶子节点，时间复杂度固定 O(log n) | 可能在任意节点结束，不稳定 |
| 范围查询 | 叶子节点链表连接，顺序扫描效率高 | 需要中序遍历，跨层级跳转多 |
| 磁盘 I/O | 非叶子节点不存储数据，每层可容纳更多索引项，树的层高更浅 | 非叶子节点也存数据，每页可存索引更少 |
| 空间利用率 | 叶子节点满存数据，利用率高 | 节点存数据和索引，空间碎片多 |

**InnoDB 的额外优化：**
- 自适应哈希索引（AHI）自动加速热点查询
- 支持主键索引和辅助索引，主键索引叶子节点存储完整行数据
- 辅助索引叶子节点存储主键值，查询回表

**面试追问：** 为什么建议使用自增主键而不是 UUID？
> 答：自增主键插入是顺序的，不会导致 B+ 树的叶子节点频繁分裂和页合并，插入效率更高，索引更紧凑。

---

## 题目二：什么是覆盖索引？如何利用覆盖索引优化查询？

### 参考答案

**覆盖索引的定义：**

一个索引包含了查询所需的所有列（SELECT、WHERE、JOIN 的列），无需回表（查询索引树即可得到结果），这就是**覆盖索引**（Covering Index）。

**示例：**

```sql
-- 表结构
CREATE TABLE orders (
    id BIGINT PRIMARY KEY AUTO_INCREMENT,
    user_id INT NOT NULL,
    order_time DATETIME NOT NULL,
    amount DECIMAL(10,2),
    INDEX idx_user_time (user_id, order_time)
);

-- 命中覆盖索引（无需回表）
SELECT user_id, order_time FROM orders WHERE user_id = 100;

-- 无法覆盖索引（amount 列不在索引中，需回表）
SELECT user_id, order_time, amount FROM orders WHERE user_id = 100;
```

**利用 EXPLAIN 分析：**

```sql
EXPLAIN SELECT user_id, order_time FROM orders WHERE user_id = 100;
-- Extra 列显示 "Using index" 表示使用了覆盖索引
```

**覆盖索引的优化思路：**

1. **尽量让 SELECT 的列都在索引中**：将需要查询的列加入索引
2. **遵循最左前缀原则**：合理设计复合索引列顺序
3. **利用索引下推（Index Condition Pushdown, ICP）**：MySQL 5.6+ 支持在索引层就完成部分 WHERE 条件过滤

**PostgreSQL 的对应机制：**
PostgreSQL 使用 **Index Only Scan** 实现覆盖索引，效果相同。

**注意事项：**
- 覆盖索引会增大索引体积，平衡查询性能与写入开销
- 过多覆盖索引会影响 INSERT/UPDATE/DELETE 性能
- 频繁更新的表慎用

---

## 题目三：MySQL 查询很慢，如何定位和优化？

### 参考答案

**Step 1：定位慢查询**

```sql
-- 开启慢查询日志（5秒以上）
SET GLOBAL slow_query_log = 'ON';
SET GLOBAL long_query_time = 5;
SET GLOBAL slow_query_log_file = '/var/log/mysql/slow.log';

-- 查看当前会话慢查询状态
SHOW VARIABLES LIKE 'slow_query%';

-- 通过 EXPLAIN 分析执行计划
EXPLAIN SELECT * FROM orders WHERE user_id = 100;

-- 查看连接数和状态
SHOW STATUS LIKE 'Threads_connected';
SHOW PROCESSLIST;
```

**Step 2：EXPLAIN 关键字段解读**

| 字段 | 含义 | 优化关注点 |
|------|------|-----------|
| type | 访问类型（const/eq_ref/ref/range/index/ALL） | 尽量避免 ALL（全表扫描） |
| key | 实际使用的索引 | NULL 表示未用索引 |
| rows | 扫描行数估算 | 越大越需要优化 |
| Extra | 额外信息 | Using filesort、Using temporary 需要优化 |

**Step 3：常见优化手段**

```sql
-- 1. 避免 SELECT *，只查需要的列
SELECT id, amount FROM orders WHERE user_id = 100;

-- 2. 避免在索引列上使用函数
-- 错误：WHERE YEAR(order_time) = 2025
-- 正确：WHERE order_time >= '2025-01-01' AND order_time < '2026-01-01'

-- 3. 使用 LIMIT 优化分页（深度分页问题）
-- 错误：SELECT * FROM orders LIMIT 100000, 10;
-- 正确：SELECT * FROM orders WHERE id > 100000 LIMIT 10;

-- 4. 合理使用提示（HINT）
SELECT * FROM orders USE INDEX (idx_user_id) WHERE user_id = 100;

-- 5. 分解大查询，减少锁竞争
-- 多次小查询替代一次大查询
```

**Step 4：数据库层面优化**

```sql
-- 分析表索引统计信息
ANALYZE TABLE orders;

-- 查看表状态
SHOW TABLE STATUS FROM db_name LIKE 'orders';

-- 调整 InnoDB 缓冲池大小（建议物理内存的 60-80%）
SET GLOBAL innodb_buffer_pool_size = 4 * 1024 * 1024 * 1024;
```

---

## 题目四：InnoDB 和 MyISAM 存储引擎的核心区别是什么？如何选型？

### 参考答案

| 特性 | InnoDB | MyISAM |
|------|--------|--------|
| 事务支持 | 支持 ACID 事务（commit/rollback） | 不支持事务 |
| 锁粒度 | 行级锁 + 间隙锁 | 表级锁 |
| 并发能力 | 高并发场景表现好 | 大数据量写并发差 |
| 崩溃恢复 | 支持 crash-safe，有 redo log | 损坏后难恢复 |
| 外键约束 | 支持外键 | 不支持 |
| 索引结构 | B+ 树（聚簇索引） | B+ 树（非聚簇索引） |
|COUNT(*) | 全表扫描，较慢 | 有专门计数器，快 |
| 全文索引 | 5.6+ 支持 | 支持（较早版本就有） |
| 适用场景 | 核心业务、高并发、需事务 | 读多写少、不需要事务 |

**InnoDB 聚簇索引 vs MyISAM 非聚簇索引：**

```
InnoDB（聚簇索引）:
  主键索引叶子节点直接存储行数据（和数据页在一起）
  辅助索引叶子节点存储主键值

MyISAM（非聚簇索引）:
  所有索引叶子节点都存储数据行地址（指向 .MYD 文件中的位置）
```

**崩溃恢复对比：**
- InnoDB：redolog + undolog，支持脏页刷新到磁盘，崩溃后自动恢复
- MyISAM：无日志，崩溃后可能丢失数据或索引损坏

**选型建议：**

> 几乎所有新项目都应选择 InnoDB，除非：
> 1. 需要大量 COUNT(*) 查询且无事务需求（如日志统计）
> 2. 存储引擎固定为 MyISAM 的遗留系统
> 3. 确实需要全文索引且版本 < MySQL 5.6

**MySQL 8.0+ 的变化：**
- 默认存储引擎强制 InnoDB
- 新增 redo log 日志表（InnoDB 专用）
- 废弃 MyISAM 系统表

---

## 题目五：如何设计分库分表方案？分片键如何选择？

### 参考答案

**分库分表的动机：**

单库单表在数据量超过千万级时，查询性能急剧下降，需要水平拆分。

**拆分策略：**

```
垂直拆分（按业务列拆分）:
  用户表: user_id, name, email → 用户库
  订单表: order_id, user_id, amount → 订单库

水平拆分（按数据行拆分）:
  订单表 → 订单库A（user_id % 4 == 0）
  订单表 → 订单库B（user_id % 4 == 1）
  订单表 → 订单库C（user_id % 4 == 2）
  订单表 → 订单库D（user_id % 4 == 3）
```

**分片键（Sharding Key）选择原则：**

1. **业务热点列**：查询最频繁的条件作为分片键
2. **数据均匀分布**：避免热点倾斜（如按省份分片导致某省数据过多）
3. **跨分片查询可控**：关联查询尽量在同分片

**常见的分片算法：**

```sql
-- 1. 哈希取模（Hash Mod）
shard_id = user_id % 4
-- 优点：数据分布均匀
-- 缺点：扩容困难（数量变化时迁移大量数据）

-- 2. 范围分片（Range）
shard_id = user_id / 1000000
-- 优点：天然支持范围查询
-- 缺点：容易产生热点（如新用户集中在最新分片）

-- 3. 一致性哈希（Consistent Hashing）
-- 优点：扩容时只需迁移部分数据
-- 缺点：实现复杂，需引入虚拟节点

-- 4. 查表法（Lookup Table）
-- 预先维护分片映射表，灵活控制数据分布
```

**扩容解决方案：**

| 方案 | 原理 | 迁移量 |
|------|------|--------|
| 翻倍扩容 | 从 N 扩到 2N，哈希取模变为 2N | 约 50% 数据迁移 |
| 一致性哈希扩容 | 环形哈希空间 | 约 1/(N+1) 迁移 |
| 离线双写 | 开启双写，增量数据同步 | 停机切换 |

**分库分表后的挑战：**

1. **跨分片聚合（COUNT、SUM、GROUP BY）**：需要汇总中间结果
2. **跨分片分页**：先跨分片查询再归并排序，深度分页极慢
3. **分布式事务**：使用 Seata、TCC 等分布式事务方案
4. **唯一ID生成**：不能依赖自增主键，需使用 Snowflake、UUID 等

**推荐中间件：**
- ShardingSphere（Java 系，成熟度高）
- MyCat（老牌，但社区活跃度下降）
- Vitess（YouTube 开源，Kubernetes 友好）

---

## 📚 参考来源

1. **MySQL 官方文档 - InnoDB Storage Engine**
   https://dev.mysql.com/doc/refman/8.0/en/innodb-storage-engine.html

2. **MySQL 官方文档 - Optimization and Indexes**
   https://dev.mysql.com/doc/refman/8.0/en/optimization.html

3. **PostgreSQL Documentation - Performance Tips**
   https://www.postgresql.org/docs/current/performance-tips.html

4. **掘金 - MySQL 索引总结**
   https://juejin.cn/post/6844903603884113933

5. **美团技术博客 - MySQL 索引原理及慢查询优化**
   https://tech.meituan.com/2014/06/30/mysql-index.html
