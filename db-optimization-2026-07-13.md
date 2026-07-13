# 数据库性能优化面试题

> 📅 2026-07-13 | 来源：综合整理（牛客网、力扣、知乎等）

---

## 题目 1：MySQL InnoDB 引擎中索引失效的场景有哪些？

**参考答案：**

1. **使用 `LIKE` 前缀匹配**：`LIKE '%keyword'` 导致全表扫描
2. **索引列参与计算或函数**：如 `WHERE YEAR(date_col) = 2024`
3. **类型隐式转换**：where 条件类型不匹配，如字符型字段传入数字
4. **使用 `OR` 连接非索引列**：`WHERE col1 = 'a' OR col2 = 'b'`（除非所有列都有索引且用 UNION）
5. **不等于比较**：`!=`、`NOT IN`、`NOT EXISTS` 通常不走索引
6. **组合索引违反最左前缀原则**：如创建 `(a, b, c)` 索引，查询 `WHERE b = 1` 不走索引
7. **`ORDER BY` 与 `WHERE` 条件不连贯**：先执行 WHERE 筛选后再 filesort

> 来源：https://www.nowcoder.com/question/answer/5e7c0b6c8b6d4e7ba4bf0a3b0c9e6f1d（牛客网 MySQL 专题）

---

## 题目 2：如何定位并优化慢查询？

**参考答案：**

**定位步骤：**

1. 开启慢查询日志：`slow_query_log = 1; slow_query_log_file = /var/log/mysql/slow.log; long_query_time = 1`
2. 使用 `EXPLAIN` 分析执行计划，关注 `type`、`key`、`rows`、`Extra` 字段
3. 借助 `SHOW PROCESSLIST` 查看当前连接和查询状态

**优化手段：**

- 检查是否命中索引，添加合适索引
- 避免 `SELECT *`，只查需要的字段
- 优化子查询，改用 JOIN 或临时表
- 分析是否需要分表（水平/垂直拆分）
- 使用 `EXPLAIN ANALYZE`（MySQL 8.0+）查看实际执行计划

> 来源：https://www.leetcode.cn/circle/discuss/5dX8K4/（LeetCode 中国数据库面试题精选）

---

## 题目 3：MySQL InnoDB 与 MyISAM 存储引擎的区别是什么？

**参考答案：**

| 特性 | InnoDB | MyISAM |
|---|---|---|
| 事务支持 | 支持 ACID 事务 | 不支持 |
| 锁粒度 | 行级锁 + 间隙锁 | 表级锁 |
| 外键 | 支持 | 不支持 |
| 崩溃恢复 | 自动崩溃恢复（redo log） | 需手动修复 |
| 全文索引 | MySQL 5.6+ 支持 | 原生支持 |
| 适用场景 | 高并发、事务场景 | 读多写少、表级锁可接受的场景 |
| 存储结构 | `.ibd` 表空间 | `.MYD` + `.MYI` + `.frm` |

> 来源：https://www.zhihu.com/question/19909658（知乎 MySQL 引擎对比讨论）

---

## 题目 4：PostgreSQL 中如何做查询性能优化？

**参考答案：**

1. **使用 `EXPLAIN ANALYZE`**：查看实际执行时间，找出性能瓶颈节点
2. **创建合适的索引**：
   - B-tree 索引：默认，适用于等值和范围查询
   - GIN 索引：适用于数组、全文检索、JSONB
   - GiST 索引：适用于几何类型、范围类型
3. **VACUUM 和 ANALYZE**：定期清理死元组，更新统计信息，帮助优化器生成更优计划
4. **分区表**：对大表按时间或业务字段分区（`PARTITION BY RANGE`）
5. **避免大事务中锁竞争**：使用分段提交，减少长事务对并发的阻塞
6. **配置参数调优**：`shared_buffers`、`work_mem`、`effective_cache_size` 等

> 来源：https://www.postgresql.org/docs/current/performance-tips.html（PostgreSQL 官方文档）

---

## 题目 5：如何设计一个高效的数据库缓存策略？

**参考答案：**

**缓存层级：**
- **应用层缓存**：Redis/Memcached 缓存热点数据
- **数据库层缓存**：InnoDB Buffer Pool 缓存数据页和索引页

**缓存策略：**
1. **Cache-Aside（旁路缓存）**：读时先查缓存，未命中查数据库并写入缓存；写时先更新数据库，再删除/更新缓存
2. **Read-Through**：缓存负责加载数据，应用程序只和缓存交互
3. **Write-Through**：写数据时同步写缓存和数据库，保证强一致性

**缓存问题及解决：**
- **缓存穿透**：布隆过滤器或缓存空值（设置短 TTL）
- **缓存击穿**：热点 key 永不过期 + 互斥锁或逻辑过期
- **缓存雪崩**：过期时间加随机值，或服务降级/限流
- **数据一致性**：采用延迟双删策略（先删缓存 → 更新 DB → 延迟再删缓存）

> 来源：https://xiaolincoding.com/cache/（小林coding 数据库与缓存系列文章）

---

*📌 本题库由 AI 自动整理，每日定时推送至 GitHub | 仓库：qq286158530/interview-questions*
