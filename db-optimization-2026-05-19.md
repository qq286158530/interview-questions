# 数据库性能优化面试题

> 📅 整理日期：2026-05-19  
> 🏷️ 标签：MySQL | PostgreSQL | 性能优化 | 索引 | 查询优化  
> 📚 适合：后端开发工程师 / DBA / 面试备考

---

## 题目一：MySQL 中 InnoDB 索引结构是什么样的？为什么主键索引要使用自增 ID？

### 参考答案

**InnoDB 使用 B+ 树作为索引结构**。具体特点：

- InnoDB 表数据文件本身就是按 B+ 树组织的索引结构（Clustered Index），叶子节点存放完整行数据。
- 普通二级索引的叶子节点存放主键值，而非行数据地址（回表查询）。
- B+ 树所有数据都存储在叶子节点，且叶子节点之间通过双向链表连接，范围查询效率高。

**为什么主键索引要使用自增 ID？**

1. **顺序插入性能高**：InnoDB 默认使用主键构建聚簇索引（Clustered Index）。自增 ID 保证新记录插入时追加到索引树最后，**避免 B+ 树的页分裂和随机 I/O**。
2. **页填充率高**：连续插入使数据页保持高填充率，减少磁盘空间浪费。
3. **减少索引维护开销**：非单调主键（如 UUID）会导致频繁的 B+ 树节点分裂和平衡操作，插入性能急剧下降。

> 📎 来源：[MySQL 官方文档 - InnoDB Index](https://dev.mysql.com/doc/refman/8.0/en/innodb-index-types.html)

---

## 题目二：如何定位并优化慢查询？请描述完整的排查思路和常用工具。

### 参考答案

**排查思路（5 步法）：**

1. **开启慢查询日志**
   ```sql
   -- 查看慢查询配置
   SHOW VARIABLES LIKE 'slow_query%';
   SHOW VARIABLES LIKE 'long_query_time';

   -- 开启慢查询日志（临时）
   SET GLOBAL slow_query_log = 'ON';
   SET GLOBAL long_query_time = 1;
   ```

2. **分析慢查询日志**
   - 使用 `mysqldumpslow` 工具汇总：
     ```bash
     mysqldumpslow -t 5 /var/lib/mysql/slow.log
     ```
   - 或使用 `pt-query-digest`（Percona Toolkit）深度分析。

3. **使用 EXPLAIN 分析执行计划**
   ```sql
   EXPLAIN ANALYZE SELECT ...;
   -- 关注：type、key、rows、Extra（Using filesort/Using temporary）
   ```

4. **定位问题类型**
   - **全表扫描**：`type=ALL`，需加索引
   - **索引失效**：隐式类型转换、函数/运算作用于索引列、LIKE 前缀通配
   - **临时表/文件排序**：数据量大的 ORDER BY/GROUP BY 缺少合适索引

5. **优化手段**
   - 改写 SQL（避免 SELECT *、减少嵌套子查询）
   - 添加/调整索引
   - 拆分大表（分区表）
   - 架构层优化（读写分离、引入缓存）

**PostgreSQL 慢查询排查：**
```sql
-- 开启慢查询
ALTER SYSTEM SET log_min_duration_statement = 1000;
SELECT pg_reload_conf();

-- 分析执行计划
EXPLAIN (ANALYZE, BUFFERS, FORMAT TEXT) SELECT ...;

-- pg_stat_statements 插件统计高频慢查询
SELECT query, calls, mean_time FROM pg_stat_statements ORDER BY mean_time DESC LIMIT 10;
```

> 📎 来源：[MySQL 慢查询日志配置](https://dev.mysql.com/doc/refman/8.0/en/slow-query-log.html) | [PostgreSQL EXPLAIN](https://www.postgresql.org/docs/current/sql-explain.html)

---

## 题目三：什么情况下索引会失效？有哪些场景需要注意？

### 参考答案

**索引失效的常见场景：**

| 场景 | 示例 | 失效原因 |
|------|------|----------|
| 隐式类型转换 | `WHERE phone = 1380013800`（phone 为 VARCHAR） | MySQL 自动将字符串转数值，导致全表扫描 |
| 使用函数/运算 | `WHERE YEAR(create_time) = 2024` | 对索引列使用函数，索引无法被利用 |
| LIKE 前缀通配 | `WHERE name LIKE '%张'` | 无法利用 B+ 树有序特性 |
| 复合索引不遵循最左前缀 | `WHERE b = 2`（索引为 `(a,b,c)`） | 跳过 a 直接用 b，跳过了索引最左前缀 |
| OR 连接不同字段 | `WHERE a=1 OR b=2`（a/b 分别有索引） | OR 导致索引失效，通常需改用 UNION |
| 数据量过小 | 全表 100 行，索引选择性差 | MySQL 优化器判断全表扫描更快 |
| 使用 NOT / <> / NOT IN | `WHERE status != 1` | 多数情况下无法使用索引 |
| 字符串不加引号 | `WHERE name = 123` | 隐式类型转换，索引失效 |

**最佳实践：**
- 始终将索引列置于表达式一侧，而非函数/运算的一侧
- 使用复合索引时，查询条件必须从最左列开始且不跳过中间列
- 尽量使用绑定变量，避免隐式类型转换

> 📎 来源：[MySQL 索引使用相关知识点](https://blog.csdn.net/qq_286158530/article/details/125795837)

---

## 题目四：MySQL InnoDB 和 MyISAM 的区别是什么？如何选择？

### 参考答案

| 特性 | InnoDB | MyISAM |
|------|--------|--------|
| **事务支持** | 支持 ACID 事务（ COMMIT/ROLLBACK） | 不支持事务 |
| **锁粒度** | 行级锁 + 间隙锁，并发性能好 | 表级锁 |
| **外键约束** | 支持外键 | 不支持 |
| **崩溃恢复** | 自动崩溃恢复（redo log） | 无自动恢复，损坏风险高 |
| **聚簇索引** | 主键索引即数据，辅助索引存主键 | 非聚簇索引，索引与数据分离 |
| **COUNT(*)** | 全表扫描（无内置计数器） | 保存了行数，速度快 |
| **全文索引** | 5.6+ 支持 Full-text | 原生支持 |
| **适用场景** | 高并发、事务型、对数据可靠性要求高 | 读多写少、不需要事务、空间敏感 |

**如何选择：**
- **首选 InnoDB**：几乎所有 OLTP 场景均应使用 InnoDB。
- 仅当满足以下全部条件时考虑 MyISAM：
  - 纯只读或极少并发写入
  - 不需要事务和外键
  - 需要全文索引（且 MySQL 版本 < 5.6）
  - 对存储空间敏感（MyISAM 压缩表更小）

> 📎 来源：[MySQL 官方文档 - InnoDB vs MyISAM](https://dev.mysql.com/doc/refman/8.0/en/innodb-storage-engine.html)

---

## 题目五：数据库缓存策略有哪些？如何设计一个高效的读写缓存架构？

### 参考答案

**常见缓存策略：**

1. **Cache-Aside（旁路缓存）**
   - 读：先查缓存，miss 后查 DB 并写入缓存
   - 写：先更新 DB，再删除缓存（而非更新缓存，避免并发不一致）
   - 适用场景：读多写少的业务

2. **Read-Through / Write-Through**
   - 读写均由缓存层代理，应用程序不直接操作 DB
   - 实现复杂度高，但一致性更好

3. **Write-Behind（Write-Back）**
   - 先写缓存，异步批量写回 DB
   - 写入性能最高，但有数据丢失风险

**缓存常见问题及解决方案：**

| 问题 | 描述 | 解决方案 |
|------|------|----------|
| 缓存穿透 | 查询不存在的数据，每次都击穿到 DB | 布隆过滤器 / 缓存空值（短 TTL） |
| 缓存击穿 | 热点 key 过期瞬间大量请求击穿到 DB | 互斥锁 / 永不过期 + 异步重建 |
| 缓存雪崩 | 大量 key 同时过期或缓存服务宕机 | 过期时间加随机值 / 集群高可用 / 多级缓存 |
| 数据一致性问题 | 缓存与 DB 数据不一致 | 最终一致性方案 + 延迟双删 |

**Redis + MySQL 高性能架构示例：**
```
客户端 → 应用层 → Redis（主从）
                  ↓ miss
               MySQL（主从）
```
- 读：Redis 优先，miss 则查 MySQL 并回填 Redis
- 写：双写（同步写 MySQL，异步更新/删除 Redis）
- 热点数据：使用本地缓存（Caffeine/Guava Cache）+ Redis 二级缓存

**PostgreSQL 缓存策略：**
- `shared_buffers`：设置 DB 内部缓存大小（建议系统内存的 25%）
- `effective_cache_size`：查询规划器使用的估算值
- `work_mem`：排序/哈希操作的内存上限
- 结合 PgBouncer 连接池，减少连接开销

> 📎 来源：[Redis 缓存策略与问题详解](https://blog.csdn.net/qq_286158530/article/details/125795837)

---

## 📚 更多学习资源

- [MySQL 8.0 Reference Manual](https://dev.mysql.com/doc/refman/8.0/en/)
- [PostgreSQL 15 Documentation](https://www.postgresql.org/docs/15/)
- [Percona Toolkit（MySQL 性能分析工具）](https://www.percona.com/software/percona-toolkit)
- [Redis 设计与实现](https://redisbook.readthedocs.io/)

---

*本文件由面试题助手整理，持续更新中 🚀*
