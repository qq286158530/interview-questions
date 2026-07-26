# 数据库性能优化面试题

> 日期：2026-07-26
> 来源：整理自业界经典面试题库

---

## 题目1：MySQL索引失效的场景有哪些？

### 参考答案

**索引失效的常见场景：**

1. **使用函数或运算**
   - `WHERE YEAR(create_time) = 2026` — 对索引列使用函数
   - `WHERE id + 1 = 10` — 对索引列进行运算

2. **类型转换**
   - 字段为字符串，但传入数字：`WHERE phone = 1380013800`（phone是varchar）
   - 隐式类型转换导致索引失效

3. **使用LIKE以通配符开头**
   - `WHERE name LIKE '%张'` — 前导通配符无法使用索引
   - `WHERE name LIKE '张%'` — 可以使用索引

4. **使用OR连接条件**
   - `WHERE id = 1 OR name = '张三'` — OR两端非同一索引列
   - 解决：拆分为UNION或使用IN

5. **联合索引违反最左前缀原则**
   - 联合索引为(a,b,c)，但查询条件只有`WHERE b = 1`

6. **使用NOT、!=、<>操作符**
   - 可能导致全表扫描

7. **IS NULL / IS NOT NULL**
   - 部分场景下索引失效

8. **数据量过小时** — MySQL优化器认为全表扫描更快

**建议：** 用`EXPLAIN`分析SQL执行计划，确认索引是否被使用。

来源：https://tech.meituan.com/2014/06/30/mysql-index.html

---

## 题目2：如何排查和解决MySQL慢查询？

### 参考答案

**排查步骤：**

1. **开启慢查询日志**
   ```sql
   -- 查看是否开启
   SHOW VARIABLES LIKE 'slow_query_log';
   
   -- 设置慢查询阈值（秒）
   SET GLOBAL slow_query_log = 'ON';
   SET GLOBAL long_query_time = 1;
   
   -- 查看日志位置
   SHOW VARIABLES LIKE 'slow_query_log_file';
   ```

2. **使用EXPLAIN分析执行计划**
   ```sql
   EXPLAIN SELECT * FROM users WHERE name = '张三';
   ```
   关注：`type`、`key`、`rows`、`Extra`列

3. **使用SHOW PROFILE**
   ```sql
   SET profiling = 1;
   SELECT ...;
   SHOW PROFILES;
   SHOW PROFILE FOR QUERY 1;
   ```

4. **使用Performance Schema**
   ```sql
   SELECT * FROM performance_schema.events_statements_summary_by_digest 
   ORDER BY SUM_TIMER_WAIT DESC LIMIT 10;
   ```

**解决措施：**

- 优化索引（添加合适索引、删除冗余索引）
- 优化SQL语句（避免SELECT *、减少嵌套子查询）
- 读写分离（主从复制）
- 分库分表
- 使用缓存（Redis）

来源：https://www.cnblogs.com/shangxia/p/13855436.html

---

## 题目3：InnoDB和MyISAM存储引擎的区别？如何选择？

### 参考答案

| 特性 | InnoDB | MyISAM |
|------|--------|--------|
| 事务支持 | 支持ACID事务 | 不支持 |
| 锁粒度 | 行级锁 | 表级锁 |
| 并发能力 | 高 | 低 |
| 外键 | 支持 | 不支持 |
| 崩溃恢复 | 自动恢复 | 需手动修复 |
| 全文索引 | 5.6+支持 | 原生支持 |
| 存储空间 | 约2倍 | 较小 |
| 适用场景 | 写多、事务需求 | 读多、空间受限 |

**选择建议：**

- **选择InnoDB：** 大多数场景默认选择，尤其当需要事务、行级锁、外键、崩溃恢复能力时
- **选择MyISAM：** 只读场景、表数据量小、不需要事务、追求极小存储空间

**补充：** MySQL 8.0+已移除MyISAM，强烈建议使用InnoDB。

来源：https://dev.mysql.com/doc/refman/8.0/en/innodb-introduction.html

---

## 题目4：什么是数据库连接池？Redis缓存与数据库如何保持一致性？

### 参考答案

**数据库连接池：**

连接池是一种管理数据库连接的技术，通过预先建立并复用连接，避免频繁创建/销毁连接的开销。

常见连接池：HikariCP、Druid、C3P0

**缓存一致性策略：**

1. **Cache Aside（旁路缓存）** — 最常用
   - 读：先读缓存，缓存命中直接返回；未命中查数据库并更新缓存
   - 写：先更新数据库，再删除缓存（不是更新）

2. **Read Through**
   - 缓存负责从数据库加载数据，应用只与缓存交互

3. **Write Through**
   - 写操作同时更新缓存和数据库

4. **延迟双删**（解决并发问题）
   ```java
   // 1. 先删除缓存
   redis.del(key);
   // 2. 更新数据库
   db.update();
   // 3. 延迟再删（等并发读完成）
   Thread.sleep(100);
   redis.del(key);
   ```

**常见问题：** 缓存穿透（布隆过滤器）、缓存击穿（热点key永不过期）、缓存雪崩（随机过期时间）

来源：https://cloud.google.com/archive/redis-caching-patterns.html

---

## 题目5：PostgreSQL如何进行性能调优？

### 参考答案

**1. 慢查询分析**
```sql
-- 开启pg_stat_statements
CREATE EXTENSION pg_stat_statements;

-- 查看最慢的SQL
SELECT query, calls, mean_time, total_time 
FROM pg_stat_statements 
ORDER BY mean_time DESC LIMIT 10;
```

**2. 索引优化**
```sql
-- 查看索引使用情况
SELECT idx_scan, idx_tup_read, indexrelname 
FROM pg_stat_user_indexes 
WHERE idx_scan = 0;

-- 创建合适索引
CREATE INDEX idx_users_email ON users(email);
CREATE INDEX idx_orders_status ON orders(status) WHERE status != 'completed';
```

**3. 配置参数调优**
```sql
-- 共享缓冲区（建议OS内存的25%）
ALTER SYSTEM SET shared_buffers = '4GB';

-- 工作内存（单个排序操作）
ALTER SYSTEM SET work_mem = '256MB';

-- 有效缓存大小
ALTER SYSTEM SET effective_cache_size = '12GB';
```

**4. VACUUM和 ANALYZE**
```sql
-- 清理死元组
VACUUM ANALYZE users;

-- 自动清理配置
ALTER SYSTEM SET autovacuum = on;
```

**5. EXPLAIN ANALYZE分析执行计划**
```sql
EXPLAIN (ANALYZE, BUFFERS, FORMAT TEXT) SELECT * FROM orders WHERE status = 'pending';
```

**6. 分区表**（大表优化）
```sql
CREATE TABLE orders (
    id BIGSERIAL,
    created_at DATE
) PARTITION BY RANGE (created_at);

CREATE TABLE orders_2026 PARTITION OF orders
    FOR VALUES FROM ('2026-01-01') TO ('2027-01-01');
```

来源：https://www.postgresql.org/docs/current/performance-tips.html

---

*整理不易，如果对你有帮助，欢迎star⭐*
