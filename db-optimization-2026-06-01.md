# 数据库性能优化面试题

> 📅 2026-06-01 | 由小憨宝自动推送

---

## 题目一：MySQL 索引失效的常见场景有哪些？

### 答案

索引失效的常见场景包括：

1. **使用 NOT、<>、!= 操作符**
   - 可使用 `IS NULL` / `IS NOT NULL` 触发索引
   - `WHERE score <> 0` 通常不走索引

2. **使用 LIKE 前面带通配符**
   - `LIKE '%abc'` 前导通配符导致索引失效
   - `LIKE 'abc%'` 可以使用索引

3. **对索引列进行计算或函数**
   - `WHERE YEAR(create_time) = 2024` 不走索引
   - 应改为范围查询：`WHERE create_time >= '2024-01-01' AND create_time < '2025-01-01'`

4. **类型转换**
   - 字符串列用数字查询：`WHERE phone = 13800138000`（phone 为 varchar）
   - 隐式类型转换导致全表扫描

5. **OR 连接条件中有一列无索引**
   - `WHERE name = '张三' OR age = 20`，若 age 无索引则失效
   - 建议：为每个条件列都建索引，或拆分为 UNION

6. **最左前缀原则不匹配**
   - 对联合索引 (A,B,C)，若跳过 A 只查 B、C，则索引失效

7. **使用 IS NULL / IS NOT NULL**
   - 某些引擎（如 InnoDB）在 NULL 值较多时可能不走索引

8. **数据量过小**
   - MySQL 优化器认为全表扫描比索引查找更快时，会放弃索引

**来源**：[MySQL索引失效的常见场景 | 博客园](https://www.cnblogs.com/jianfei33/p/13865047.html)

---

## 题目二：如何优化慢查询？请说出完整的思路和步骤。

### 答案

**Step 1：开启慢查询日志，定位慢 SQL**
```sql
-- 开启慢查询日志
SET GLOBAL slow_query_log = 'ON';
SET GLOBAL long_query_time = 1; -- 超过1秒记录

-- 查看慢查询日志位置
SHOW VARIABLES LIKE 'slow_query_log_file';
```

**Step 2：使用 EXPLAIN 分析执行计划**
```sql
EXPLAIN SELECT * FROM orders WHERE status = 1;
```
重点关注：
- `type`：全表扫描(all)还是索引(range)，尽量避免 all
- `key`：实际使用的索引
- `rows`：扫描行数，越少越好
- `Extra`：Using filesort / Using temporary 表示需要优化

**Step 3：根据分析结果针对性优化**
- 加合适索引（覆盖索引减少回表）
- 避免 SELECT *
- 拆分大表关联（N+1 问题）
- 减少 JOIN 层数
- 利用 EXPLAIN 的 using filesort 判断是否建立合理索引

**Step 4：业务层优化**
- 限制结果集（加 LIMIT）
- 分页优化（游标分页替代 OFFSET）
- 使用缓存减少数据库压力

**Step 5：持续监控**
- 使用 APM 工具（Pinpoint、SkyWalking）
- 定期分析慢查询日志

**来源**：[MySQL慢查询优化 | 掘金](https://juejin.cn/post/7160424072101642263)

---

## 题目三：InnoDB 和 MyISAM 存储引擎的区别是什么？如何选择？

### 答案

| 特性 | InnoDB | MyISAM |
|------|--------|--------|
| **事务支持** | 支持 ACID 事务 | 不支持事务 |
| **锁粒度** | 行级锁 + 表级锁 | 表级锁 |
| **并发能力** | 高（行级锁） | 低（表锁） |
| **外键支持** | 支持 | 不支持 |
| **崩溃恢复** | 自动恢复（MVCC） | 需要修复 |
| **索引结构** | 聚簇索引（数据在索引叶节点） | 非聚簇索引 |
| **COUNT(*)** | 需全表扫描 | 维护了计数器 |
| **适用场景** | 写入多、事务需求 | 读多写少、只读表 |

**选择建议：**
- **InnoDB**：生产环境首选（事务安全、高并发、支持外键）
- **MyISAM**：适用于只读报表、日志表（无事务、崩溃后需手动修复）

**InnoDB 聚簇索引特点：**
- 主键索引的叶节点存储完整数据行
- 二级索引叶节点存储主键值（回表查主键再查数据）
- 因此主键不宜过长，会导致二级索引体积膨胀

**来源**：[InnoDB vs MyISAM 区别对比 | CSDN](https://blog.csdn.net/qq_41884968/article/details/124482634)

---

## 题目四：什么是数据库连接池？为什么要使用连接池？

### 答案

**连接池原理：**
数据库连接池是在内存中预先建立一定数量的数据库连接，所有线程共享这些连接。需要时从池中取空闲连接，使用完后归还池中，避免反复创建/销毁连接的开销。

**不使用连接池的问题：**
- 每次 SQL 操作都要经历：建立 TCP 连接 → 认证 → 执行 → 关闭
- 一次查询可能 90% 时间花在建立/关闭连接上
- 高并发时连接数暴涨，易导致数据库过载

**使用连接池的优势：**
- 复用连接，减少建立/销毁开销（约 10-100 倍性能提升）
- 控制并发数量，防止数据库过载
- 连接可快速获取，减少等待时间
- 统一管理连接生命周期，易于监控

**常见连接池：**
- Java：HikariCP（Spring Boot 默认）、Druid、DBCP
- Node.js：mysql2（内置）、sequelize
- Python：SQLAlchemy（内置连接池）

**配置要点：**
- `maxActive`：最大连接数（一般 = CPU 核心数 × 2）
- `minIdle`：最小空闲连接数
- `maxWait`：获取连接最大等待时间
- `validationQuery`：连接有效性检测 SQL

**来源**：[数据库连接池详解 | 腾讯云开发者社区](https://cloud.tencent.com/developer/news/784444)

---

## 题目五：PostgreSQL 中如何分析和优化一条慢 SQL？

### 答案

**Step 1：开启慢查询日志**
```sql
-- 在 postgresql.conf 中配置
-- logging_collector = on
-- log_min_duration_statement = 1000  -- 记录超过1秒的SQL

-- 重载配置
SELECT pg_reload_conf();
```

**Step 2：使用 EXPLAIN ANALYZE 执行并分析**
```sql
EXPLAIN (ANALYZE, BUFFERS, FORMAT TEXT)
SELECT * FROM orders
WHERE status = 1 AND create_date > '2024-01-01';
```
关键指标：
- `Seq Scan`：是否全表顺序扫描
- `Index Scan / Index Only Scan`：是否使用索引
- `Buffers: shared hit`：命中缓存页数，越少说明 I/O 越大
- `Actual Rows`：实际返回行数，与估算偏差大说明统计信息过期

**Step 3：更新统计信息（统计信息过期会导致执行计划错误）**
```sql
ANALYZE orders;
-- 或
VACUUM ANALYZE orders;
```

**Step 4：强制使用索引测试**
```sql
SET enable_seqscan = off;  -- 关闭顺序扫描，测试索引效果
EXPLAIN ANALYZE SELECT * FROM orders WHERE status = 1;
SET enable_seqscan = on;
```

**Step 5：创建/优化索引**
```sql
-- 创建覆盖索引减少回表
CREATE INDEX idx_orders_status_date ON orders(status, create_date);

-- 表达式索引
CREATE INDEX idx_orders_year ON orders (date_trunc('year', create_date));
```

**Step 6：常用优化手段**
- **分区表**：`PARTITION BY RANGE`，对大表按时间分区
- **统计信息**：`ANALYZE` 定期执行
- **缓存优化**：增大 `shared_buffers`
- **只读副本**：读请求路由到只读副本分担压力

**PostgreSQL 特有工具：**
```sql
-- 查看耗时最长的SQL
SELECT query, calls, mean_time, total_time
FROM pg_stat_statements
ORDER BY total_time DESC LIMIT 10;
```

**来源**：[PostgreSQL 性能优化实战 | PostgreSQL 中文网](https://www.postgresql.org/docs/current/performance-tips.html)

---

> 🔖 每道题目均精选自公开技术资源，供学习交流使用。
> �仓库地址：https://github.com/qq286158530/interview-questions