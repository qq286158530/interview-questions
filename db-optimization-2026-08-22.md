# 数据库性能优化面试题

> 📅 整理日期：2026-08-22  
> 🐘 技术栈：MySQL · PostgreSQL  
> 💼 适用岗位：后端开发 / DBA / 数据库工程师

---

## 题目一：B+树索引的原理是什么？为什么 MySQL 采用 B+树而不是 B 树？

### 参考答案

**B+树 vs B树的核心区别：**

1. **非叶子节点不存储数据**：B+树的非叶子节点只存储索引（键），所有数据都集中在叶子节点。这使得非叶子节点能容纳更多的索引项，树的高度更低，磁盘 I/O 更少。
2. **叶子节点链表连接**：B+树的叶子节点通过双向指针连接成一个有序链表，范围查询只需定位起点后顺序遍历，无需回溯。
3. **查询稳定性**：B树的查询路径可能终止于任意层级（找到数据即返回），而B+树无论查找成功与否，都必须走到叶子节点，路径长度固定，利于预估查询成本。

**MySQL 选择 B+树的原因：**

| 特性 | B树 | B+树 |
|------|-----|------|
| 树高 | 较高（数据分散在各层） | 较低（数据只在叶子） |
| 范围查询 | 需中序遍历，跨节点需回溯 | 叶子链表，顺序扫描 |
| 磁盘预读 | 不友好 | 顺序读友好 |
| 并发控制 | 锁粒度大 | 叶子节点锁即可 |

**索引优化实践：**
- 区分度高的列放在复合索引最左侧
- 避免在索引列上使用函数或运算（导致索引失效）
- 覆盖索引：SELECT 字段全部在索引中时无需回表

> 📚 **来源**：[MySQL 索引背后的数据结构及算法原理](https://blog.csdn.net/tongdanping/article/details/79878337) — CSDN 经典文章

---

## 题目二：如何用 EXPLAIN 分析慢查询？关键字段分别代表什么？

### 参考答案

`EXPLAIN` 是 MySQL 优化的核心工具，在查询前加上即可分析执行计划。

**关键字段解析：**

```sql
EXPLAIN SELECT * FROM orders WHERE customer_id = 100 AND status = 'paid';
```

| 字段 | 含义 | 理想值 |
|------|------|--------|
| `type` | 访问类型，反映查询方式 | `const` > `eq_ref` > `ref` > `range` > `index` > `ALL` |
| `key` | 实际使用的索引 | 非 NULL |
| `rows` | 估算扫描行数 | 越少越好 |
| `Extra` | 附加信息 | 应避免 `Using filesort`、`Using temporary` |

**常见 type 值从好到差：**
- `const`：主键或唯一索引等值查询（最多1行）
- `eq_ref`：关联查询中，被驱动表的索引是主键或唯一键
- `ref`：普通索引等值查询
- `range`：索引范围扫描（> < BETWEEN IN）
- `index`：全索引扫描
- **ALL**：全表扫描（最差，必须优化）

**Extra 常见警告：**
- `Using filesort`：无法利用索引排序，需额外的排序操作 → 必须优化
- `Using temporary`：使用了临时表 → 必须优化
- `Using index condition`：索引下推，减少回表次数
- `Using where`：服务层过滤，索引未能覆盖所有条件

**优化步骤：**
1. 定位 `type=ALL` 或 `type=ALL` + `rows` 过大的查询
2. 检查 `Extra` 是否有 `Using filesort`
3. 添加/调整索引后再次 `EXPLAIN` 对比

> 📚 **来源**：[MySQL EXPLAIN 详解](https://dev.mysql.com/doc/refman/8.0/en/explain-output.html) — MySQL 官方文档

---

## 题目三：InnoDB 和 MyISAM 存储引擎的核心区别是什么？如何选择？

### 参考答案

**架构层面对比：**

| 特性 | InnoDB | MyISAM |
|------|--------|--------|
| 事务支持 | ✅ 支持 ACID 事务 | ❌ 不支持 |
| 行级锁 | ✅ 支持 | ❌ 只支持表级锁 |
| 外键约束 | ✅ 支持 | ❌ 不支持 |
| MVCC | ✅ 支持（高并发） | ❌ 不支持 |
| 崩溃恢复 | ✅ 自动恢复（redo log） | ❌ 需手动修复 |
| 索引结构 | B+树 + 自适应哈希 | B+树 |
| 表级锁 | 可在行锁基础上降级 | 粒度粗 |
| COUNT(*) | 全表扫描（无内部计数器） | 读取内部计数器（快） |

**InnoDB 的核心优势：**
- **事务安全**：rollback/提交保证数据一致性
- **高并发**：行锁 + MVCC 减少读写冲突
- **崩溃恢复**：redo log + undo log 保障持久性

**MyISAM 的适用场景：**
- 以读为主、很少并发写入的场景（如数仓的历史报表）
- 需要全文索引的场景（InnoDB 5.6+ 也支持，但 MyISAM 曾是唯一选择）
- 表级锁对业务影响可控的情况

**生产环境推荐：**
> **绝大多数场景使用 InnoDB**。MySQL 8.0+ 已将默认存储引擎改为 InnoDB，MyISAM 仅保留兼容用途。

> 📚 **来源**：[MySQL 存储引擎 InnoDB vs MyISAM 对比](https://www.mysql.com/products/enterprise/storage.html) — MySQL 官方

---

## 题目四：PostgreSQL 的 shared_buffers 和 effective_cache_size 参数如何调优？

### 参考答案

这两个参数直接影响 PostgreSQL 的内存使用和查询规划，理解它们是调优 PG 的基础。

**shared_buffers（共享缓冲区）：**

- **作用**：PostgreSQL 将表和索引数据缓存在此，是数据库的"前台缓存"
- **默认值**：`128MB`（生产环境通常太小）
- **推荐值**：机器内存的 **25%**（Linux 上建议不超过 8GB，配合 OS page cache 使用）
- **过高风险**：占用过多导致系统换页，反而降低性能

```conf
# 示例：64GB 机器
shared_buffers = 16GB
```

**effective_cache_size（有效缓存大小）：**

- **作用**：**查询规划器的参考值**，告诉 PG 操作系统和 PostgreSQL 合计能提供多少缓存空间，用于估算索引扫描 vs 全表扫描的成本
- **默认值**：`4GB`
- **推荐值**：机器内存的 **75%**（Linux page cache + shared_buffers 之和）

```conf
# 示例：64GB 机器
effective_cache_size = 48GB
```

**两者关键区别：**

| 参数 | 实际分配内存？ | 影响查询规划？ | 设置依据 |
|------|-------------|------------|---------|
| shared_buffers | ✅ 是，PG 真实使用 | ❌ 不影响 | 机器内存 25% |
| effective_cache_size | ❌ 否，纯参考 | ✅ 是，影响索引选择 | 机器内存 75% |

**调优步骤：**
1. `vmstat 1` 观察 si/so（换入/换出），如有大量换页说明 shared_buffers 过大
2. `pg_stat_user_indexes` 查看索引使用频率
3. ` EXPLAIN ( buffers )` 开启缓存信息，观察实际命中/读取的块数

> 📚 **来源**：[PostgreSQL 官方文档 — Resource Consumption](https://www.postgresql.org/docs/current/runtime-config-resource.html)

---

## 题目五：如何设计分页查询避免深度分页的性能问题？

### 参考答案

**深度分页问题的本质：**

```sql
-- 第10000页，每页20条
SELECT * FROM orders ORDER BY id LIMIT 20 OFFSET 10000;
```

这条 SQL 的执行过程是：MySQL 先读取前 10020 条记录，然后丢弃前 10000 条，返回最后 20 条。**OFFSET 越大，浪费越多**。

**方案一：游标分页（Cursor-based Pagination）— 最佳方案**

```sql
-- 第一页
SELECT * FROM orders WHERE id > 0 ORDER BY id LIMIT 20;

-- 下一页：记住上一页最后一条的 id
SELECT * FROM orders WHERE id > 15000 ORDER BY id LIMIT 20;
```

- 时间复杂度：O(1)，无论翻到第几页
- 要求：必须使用单调递增的唯一键（主键或有序索引）
- **这是 Facebook、Twitter 等大厂实际使用的方式**

**方案二：子查询优化（过渡方案）**

```sql
-- 先定位id范围，再查数据（需覆盖索引）
SELECT * FROM orders
WHERE id >= (SELECT id FROM orders ORDER BY id LIMIT 1 OFFSET 10000)
LIMIT 20;
```

**方案三：延迟关联（减少回表）**

```sql
-- 仅查主键，再关联获取完整数据
SELECT o.* FROM orders o
INNER JOIN (SELECT id FROM orders ORDER BY id LIMIT 20 OFFSET 10000) AS t
ON o.id = t.id;
```

**方案四：倒排索引 + 条件过滤**

如果业务允许按时间范围筛选，先过滤时间条件再分页：

```sql
SELECT * FROM orders
WHERE created_at >= '2026-01-01' AND created_at < '2026-02-01'
ORDER BY id LIMIT 20 OFFSET 10000;
```

**总结对比：**

| 方案 | 实现难度 | 适用场景 | 性能 |
|------|---------|---------|------|
| 游标分页 | 低 | 有单调递增主键 | O(1)，最优 |
| 子查询优化 | 低 | 过渡方案 | 较好 |
| 延迟关联 | 中 | SELECT 字段多 | 较好 |
| 条件过滤 | 低 | 可按范围筛选 | 取决于索引 |

> 📚 **来源**：[Stripe API 分页设计 — Cursor-based Pagination](https://stripe.com/docs/pagination/casts-and-parameters)

---

*整理 by 数据库面试题助手 | 欢迎 Star ⭐ [GitHub 仓库](https://github.com/qq286158530/interview-questions)*
