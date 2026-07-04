# 数据库性能优化面试题 · 2026-07-04

> 涵盖 MySQL / PostgreSQL 核心知识点：索引设计、查询调优、存储引擎、缓存策略、锁与并发

---

## 题目一：MySQL 中索引失效的常见场景有哪些？

### 参考答案

**导致索引失效的典型场景：**

1. **对索引列做函数/表达式计算**
   ```sql
   -- 索引失效 ❌
   SELECT * FROM orders WHERE YEAR(create_time) = 2026;

   -- 正确写法 ✅
   SELECT * FROM orders WHERE create_time >= '2026-01-01' AND create_time < '2027-01-01';
   ```

2. **使用 LIKE 前缀通配符**
   ```sql
   -- 索引失效 ❌（%在开头）
   SELECT * FROM users WHERE name LIKE '%张%';

   -- 正确写法 ✅
   SELECT * FROM users WHERE name LIKE '张%';  -- 索引可用
   ```

3. **数据类型隐式转换**
   ```sql
   -- phone 是 VARCHAR，传入数字会触发隐式转换，索引失效 ❌
   SELECT * FROM users WHERE phone = 13800138000;
   ```

4. **OR 连接条件中任一列无索引**
   ```sql
   -- age 有索引，name 没有 → 全表扫描 ❌
   SELECT * FROM users WHERE age = 25 OR name = '张三';
   ```

5. **WHERE 子句中对索引列使用了 NOT、<>、!=**
   多数情况下导致全表扫描。

6. **联合索引违背最左前缀原则**
   ```sql
   -- 联合索引 (a, b, c)
   WHERE b = 2          -- ❌ 未使用左前缀
   WHERE a = 1 AND c = 3  -- ⚠️ 只能用到 a 的索引
   ```

7. **表统计信息不准确**
   可通过 `ANALYZE TABLE t;` 重新统计。

> 📎 来源：MySQL 5.7 Reference Manual — [How to Avoid Indexes Not Used](https://dev.mysql.com/doc/refman/5.7/en/indexes.html)

---

## 题目二：如何优化慢查询？请描述完整的分析思路和常用工具。

### 参考答案

**Step 1：定位慢查询**
```sql
-- 开启慢查询日志
SET GLOBAL slow_query_log = 'ON';
SET GLOBAL long_query_time = 1;  -- 超过1秒记录

-- 查看慢查询日志位置
SHOW VARIABLES LIKE 'slow_query_log%';
```

**Step 2：使用 EXPLAIN 分析执行计划**
```sql
EXPLAIN SELECT * FROM orders WHERE status = 1;

-- 关键字段：
-- type:    性能从优到差：const > eq_ref > ref > range > index > ALL
-- key:     实际使用的索引
-- rows:    扫描行数，越少越好
-- Extra:   Using filesort / Using temporary → 需要优化
```

**Step 3：常见优化手段**

| 手段 | 说明 |
|------|------|
| 添加合适索引 | 为 WHERE / JOIN / ORDER BY 列建索引 |
| 分页优化 | 用延迟关联（子查询 + 覆盖索引）替代 `LIMIT 100000, 10` |
| SELECT 精简 | 只查必要字段，避免 `SELECT *` |
| 分解大查询 | 将复杂查询拆为多个简单查询 |
| 避免子查询 | MySQL 5.6 前子查询性能差，改用 JOIN |

**Step 3：使用工具**

- **MySQL Workbench / EXPLAIN ANALYZE**（MySQL 8.0+）：查看实际执行成本
- **pt-query-digest**：分析慢查询日志的 top N 慢查询
- **Performance Schema**：系统级 query 追踪

> 📎 来源：Percona Blog — [MySQL Performance: Slow Query Logs and EXPLAIN](https://www.percona.com/blog/category/mysql-performance/)

---

## 题目三：MySQL InnoDB 与 MyISAM 存储引擎的核心区别是什么？如何选择？

### 参考答案

| 特性 | InnoDB | MyISAM |
|------|--------|--------|
| **事务支持** | ✅ 支持 ACID 事务 | ❌ 不支持 |
| **行级锁** | ✅ 支持，并发好 | ❌ 表级锁 |
| **外键** | ✅ 支持 | ❌ 不支持 |
| **崩溃恢复** | ✅ 自动崩溃恢复（redo log） | ❌ 损坏率高 |
| **索引** | 聚簇索引（数据与索引同文件） | 非聚簇索引 |
| **COUNT(*)** | 全表扫描（无内部计数器） | 维护内部计数器，快 |
| **全文索引** | 5.6+ 支持 | ✅ 原生支持 |

**如何选择：**

- **选 InnoDB**：生产环境、大多数业务系统、需事务和高并发、数据可靠性优先
- **选 MyISAM**：读多写少、日志型表、全文检索场景（历史遗留场景）

> 📎 来源：MySQL 8.0 Reference Manual — [InnoDB vs MyISAM](https://dev.mysql.com/doc/refman/8.0/en/innodb-intersection.html)

---

## 题目四：什么是数据库连接池？为什么要用连接池？常见连接池有哪些？

### 参考答案

**什么是连接池？**

数据库连接池是一种复用数据库连接的机制。预先建立一组连接存放在池中，程序需要访问数据库时从池中获取，用完后归还池中而非关闭，避免反复创建/销毁连接的开销。

**为什么要用连接池：**

- TCP 三次握手 + 认证耗时，单次可能 10-100ms
- 高并发场景下连接数暴涨，数据库扛不住
- 连接数受数据库 `max_connections` 限制

**典型配置参数：**
```properties
# HikariCP 示例
maximumPoolSize=20      # 最大连接数
minimumIdle=5           # 最小空闲连接
connectionTimeout=30000 # 获取连接超时(ms)
idleTimeout=600000      # 空闲存活时间(ms)
maxLifetime=1800000     # 连接最大生命周期(ms)
```

**主流连接池：**

| 名称 | 语言/框架 | 特点 |
|------|-----------|------|
| HikariCP | Java | 性能最优，社区主流 |
| Druid | Java | 阿里出品，功能丰富（含监控） |
| c3p0 | Java | 老牌，稳定但偏慢 |
| pgBouncer | PostgreSQL | PostgreSQL 专用连接池 |
| Pgpool-II | PostgreSQL | 连接池 + 负载均衡 + 缓存 |

> 📎 来源：HikariCP GitHub — [Why HikariCP?](https://github.com/brettwooldridge/HikariCP/wiki/About-Pool-Sizing)

---

## 题目五：PostgreSQL 中如何进行 SQL 查询调优？有哪些不同于 MySQL 的关键技巧？

### 参考答案

**1. 使用 EXPLAIN ANALYZE 分析真实执行计划**

```sql
EXPLAIN (ANALYZE, BUFFERS, FORMAT TEXT)
SELECT u.name, COUNT(o.id) as order_count
FROM users u
LEFT JOIN orders o ON u.id = o.user_id
WHERE u.status = 1
GROUP BY u.id;
```

- `BUFFERS`：显示共享块命中/读取引擎缓存情况
- `ANALYZE`：实际执行并计时（注意：数据会被修改，可加 `FORMAT JSON` 规避）

**2. 善用 Partial Index（部分索引）**

```sql
-- 只为"活跃用户"建索引，索引更小更快
CREATE INDEX idx_users_active ON users(last_login) WHERE status = 'active';
```

**3. 表达式索引 vs 函数索引**

```sql
-- 对经常查询的函数结果建索引
CREATE INDEX idx_orders_month ON orders (DATE_TRUNC('month', create_time));

SELECT * FROM orders WHERE DATE_TRUNC('month', create_time) = '2026-01-01';
```

**4. PostgreSQL 特有的性能工具**

```sql
-- 查看实时慢查询（pg_stat_statements 插件）
SELECT query, calls, mean_time, total_time
FROM pg_stat_statements
ORDER BY total_time DESC
LIMIT 10;

-- 查看表膨胀情况
SELECT relname, n_dead_tup, n_live_tup, last_vacuum
FROM pg_stat_user_tables;
```

**5. 批量插入优化**

```sql
-- 使用 COPY 替代 INSERT（性能提升10-100倍）
COPY orders FROM '/tmp/orders.csv' WITH (FORMAT csv);

-- 或批量 INSERT
INSERT INTO orders(id, name) VALUES (1, 'a'), (2, 'b'), (3, 'c');
```

**6. VACUUM 与 ANALYZE 维护**

```sql
-- 手动回收死元组并更新统计信息
VACUUM ANALYZE orders;
```

> 📎 来源：PostgreSQL Documentation — [Using EXPLAIN](https://www.postgresql.org/docs/current/using-explain.html)  
> 📎 来源：PostgreSQL Wiki — [Performance Optimization Tips](https://wiki.postgresql.org/wiki/Performance_Optimization)

---

*整理by AI助手 · 欢迎 PR：https://github.com/qq286158530/interview-questions*
