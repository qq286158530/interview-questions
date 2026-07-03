# 数据库性能优化面试题

> 📅 更新时间：2026-07-03  
> 🏷️ 标签：MySQL | PostgreSQL | 性能优化 | 索引 | 查询优化

---

## 题目一：MySQL 索引失效的场景有哪些？如何避免？

### 参考答案

**常见的索引失效场景：**

1. **使用 SELECT \*** — 只看索引覆盖不了，需要回表
2. **WHERE 子句中使用函数或运算** — `WHERE YEAR(create_time) = 2024`
3. **类型转换** — 字段类型是 INT，传入字符串 `'1'` 会触发隐式转换
4. **LIKE 前缀通配** — `LIKE '%keyword'` 无法使用 B+Tree
5. **OR 连接不同字段** — `WHERE name = 'x' OR age = 20`，非索引列导致全表
6. **复合索引不遵循最左前缀原则** — `(a,b,c)` 只查 `WHERE b=1`
7. **WHERE 子句中对索引列使用 NOT、<>、NOT IN** — 导致全表扫描
8. **统计信息不准确** — ANALYZE TABLE 更新统计信息

**避免策略：**
- 使用 EXPLAIN 分析执行计划
- 尽量使用覆盖索引（covering index）
- 避免隐式类型转换，参数类型与字段类型一致
- 避免在索引列上使用函数
- 用 EXPLAIN 的 type 列检查是否使用索引（const > eq_ref > ref > range > ALL）

> 📚 来源：[MySQL索引失效场景详解](https://blog.csdn.net/ThinkWon/article/details/104588550)

---

## 题目二：如何优化慢查询？请描述排查思路和常用工具。

### 参考答案

**排查思路（SQL 执行慢的 4 大原因）：**

1. **缺乏索引** — 没有建立索引或索引失效
2. **SQL 写得烂** — 全表扫描、嵌套子查询、不当 JOIN
3. **数据量过大** — 单表数据过亿，分库分表
4. **服务器资源瓶颈** — CPU、内存、磁盘 I/O、网络

**常用工具：**

| 工具 | 用途 |
|------|------|
| `EXPLAIN` | 分析 SQL 执行计划，判断索引使用情况 |
| `SHOW PROFILE` | 查看 SQL 各阶段耗时 |
| `SHOW STATUS` | 查看服务器状态（连接数、查询数等） |
| `slow_query_log` | 开启慢查询日志，定位执行慢的 SQL |
| `pt-query-digest` | Percona Toolkit 工具，分析慢查询日志 |
| `EXPLAIN ANALYZE` | MySQL 8.0+，实际执行并分析（带成本估算） |

**优化步骤：**
```
1. 开启慢查询日志，捕获慢 SQL
2. pt-query-digest 分析日志，找出 Top N 慢查询
3. EXPLAIN / EXPLAIN ANALYZE 分析执行计划
4. 检查索引：是否用上索引、是否回表、是否 Using temporary
5. 优化 SQL：改写 JOIN、重构子查询、添加覆盖索引
6. 验证优化效果
```

> 📚 来源：[MySQL慢查询优化思路](https://www.cnblogs.com/sujing/p/11161002.html)

---

## 题目三：MySQL InnoDB 与 MyISAM 存储引擎的区别？如何选型？

### 参考答案

| 特性 | InnoDB | MyISAM |
|------|--------|--------|
| **事务支持** | ✅ 支持 ACID 事务 | ❌ 不支持 |
| **行锁** | ✅ 行级锁，并发高 | ❌ 表级锁 |
| **外键** | ✅ 支持 | ❌ 不支持 |
| **全文索引** | 5.6+ 支持 | ✅ 原生支持 |
| **索引结构** | B+Tree（聚簇索引） | B+Tree（非聚簇） |
| **崩溃恢复** | ✅ 自动恢复 | ❌ 需手动修复 |
| **存储空间** | 较大（支持MVCC） | 较小 |
| **COUNT(*)** | 全表扫描（无缓存） | 有内部计数器，快 |

**选型建议：**

- **InnoDB 优先** — 几乎所有业务场景都选 InnoDB（事务、行锁、崩溃恢复）
- **MyISAM 适用**：`SELECT COUNT(*)` 密集型、只读场景、空间受限（如 MySQL 5.5 以前）
- **判断方法**：`SHOW TABLE STATUS LIKE 'table_name'`

**面试追问点：**
> InnoDB 的聚簇索引和数据文件在一起，主键查询快；但如果主键过大（整表变大），会拖慢性能。

> 📚 来源：[InnoDB与MyISAM区别及选型](https://www.mysql.com/products/enterprise/engine.html)

---

## 题目四：什么是数据库缓存策略？Redis 和 MySQL 如何配合使用？

### 参考答案

**缓存策略核心问题：缓存和数据库的一致性问题**

**常用模式：**

1. **Cache-Aside（旁路缓存）** — 最常用
   ```
   读：先查缓存，命中则返回；未命中查DB，写入缓存，返回
   写：先更新DB，删除缓存（而非更新缓存）
   ```
   ⚠️ 删除而非更新原因：避免并发时缓存脏数据

2. **Read-Through / Write-Through** — 缓存作为代理，自动读写

3. **Write-Behind** — 异步写回，性能最高但有丢失风险

**缓存问题及解决方案：**

| 问题 | 解决方案 |
|------|----------|
| 缓存穿透 | 布隆过滤器 / 空值缓存 |
| 缓存击穿 | 互斥锁 / 热点数据永不过期 |
| 缓存雪崩 | TTL 随机偏移 / 多级缓存 / 熔断降级 |

**Redis + MySQL 配合：**
```sql
-- 示例：用户信息缓存
-- 读：GET user:1001 → 有则返回，无则查MySQL再SET
-- 写：UPDATE user SET name='xx' → DELETE user:1001（不是SET）
```

**过期策略选择：**
- Redis `expire` 设置 TTL
- 惰性删除 + 定期删除混合策略
- 热点数据用 `volatile-ttl` 淘汰

> 📚 来源：[Redis缓存策略与一致性方案](https://redis.io/docs/manual patterns/)

---

## 题目五：PostgreSQL 的 MVCC 机制是什么？它如何提升并发性能？

### 参考答案

**MVCC（Multi-Version Concurrency Control）多版本并发控制**

**核心思想：** 每次修改数据时，不直接覆盖旧数据，而是创建一个新版本。读操作不加锁，只读快照。

**PostgreSQL 的 MVCC 实现：**

```
每行数据有两个隐藏列：
- xmin：插入这条记录的事务ID
- xmax：删除/更新这条记录的事务ID

事务状态：
- 读取时：只读 xmin 提交且 xmax 未生效 的行
- 更新时：在原行打删除标记（xmax），插入新行（xmin）
```

**优势：**
| 特性 | 说明 |
|------|------|
| 读不阻塞写 | 快照读，不需要等锁 |
| 写不阻塞读 | 旧版本不影响读操作 |
| 事务隔离 | 可实现 READ COMMITTED / REPEATABLE READ 等隔离级别 |

**与 MySQL InnoDB 的区别：**

| | PostgreSQL MVCC | InnoDB MVCC |
|--|--|--|
| 实现方式 | 回滚段（Undo Log） | 隐藏列 + ReadView |
| 清理方式 | VACUUM 进程 | Purge 线程 |
| 索引方式 | 堆内元组（Heap Only Tuples） | 聚簇索引 + MVCC |

**性能优化注意：**
- PostgreSQL 需要定期 `VACUUM` 清理死亡元组，否则表会膨胀
- `autovacuum` 自动执行，但大批量更新后可能需要手动 VACUUM

> 📚 来源：[PostgreSQL MVCC机制详解](https://www.postgresql.org/docs/current/mvcc.html)

---

## 📌 延伸学习资源

- [MySQL 官方文档 - 索引优化](https://dev.mysql.com/doc/refman/8.0/en/optimization.html)
- [PostgreSQL 官方文档 - 性能调优](https://www.postgresql.org/docs/current/performance-tips.html)
- [Redis 设计与实现](https://redisbook.com/)
- [数据库索引设计与优化](https://book.douban.com/subject/33423282/)
