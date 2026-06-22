# 数据库性能优化面试题

> 📅 日期：2026-06-22
> 🏷️ 标签：MySQL | PostgreSQL | 性能优化 | 索引 | 查询优化

---

## 题目一：MySQL 索引失效的场景有哪些？如何避免？

### 参考答案

**索引失效的常见场景：**

1. **使用函数或运算**
   ```sql
   -- 索引失效 ❌
   SELECT * FROM orders WHERE YEAR(create_time) = 2026;
   
   -- 索引生效 ✅
   SELECT * FROM orders WHERE create_time >= '2026-01-01' AND create_time < '2027-01-01';
   ```

2. **使用 LIKE 前缀通配符**
   ```sql
   -- 索引失效 ❌（以通配符开头）
   SELECT * FROM users WHERE name LIKE '%张%';
   
   -- 索引生效 ✅（以固定字符开头）
   SELECT * FROM users WHERE name LIKE '张%';
   ```

3. **类型转换**
   ```sql
   -- 索引失效 ❌（隐式类型转换）
   SELECT * FROM users WHERE phone = 13800138000;
   
   -- 索引生效 ✅
   SELECT * FROM users WHERE phone = '13800138000';
   ```

4. **OR 连接条件**
   ```sql
   -- 索引可能失效 ❌
   SELECT * FROM users WHERE name = '张三' OR age = 25;
   
   -- 拆分为 UNION 或使用 IN ✅
   SELECT * FROM users WHERE name = '张三'
   UNION ALL
   SELECT * FROM users WHERE age = 25 AND name != '张三';
   ```

5. **WHERE 子句中对索引列使用 IS NULL / IS NOT NULL**
   ```sql
   -- 单列索引中 NULL 值较少时可能不走索引
   SELECT * FROM users WHERE email IS NOT NULL;
   ```

6. **联合索引违反最左前缀原则**
   ```sql
   -- 假设 index(name, age, city)
   -- 索引生效 ✅
   SELECT * FROM users WHERE name = '张三';
   SELECT * FROM users WHERE name = '张三' AND age = 25;
   
   -- 索引失效 ❌（跳过 name）
   SELECT * FROM users WHERE age = 25;
   ```

**避免索引失效的建议：**
- 尽量使用覆盖索引（covering index），减少回表
- 避免在索引列上使用函数
- 改写 OR 为 UNION 或使用 IN
- 遵循最左前缀原则
- 使用 EXPLAIN 分析查询计划

> 📚 来源：[InterviewBit - MySQL Interview Questions](https://www.interviewbit.com/mysql-interview-questions/)

---

## 题目二：如何优化慢查询？请描述具体的排查思路和优化方法。

### 参考答案

**排查思路（分步进行）：**

1. **开启慢查询日志**
   ```sql
   -- 查看慢查询配置
   SHOW VARIABLES LIKE 'slow_query%';
   SHOW VARIABLES LIKE 'long_query_time';
   
   -- 开启慢查询日志
   SET GLOBAL slow_query_log = 'ON';
   SET GLOBAL long_query_time = 1; -- 超过 1 秒记录
   ```

2. **使用 EXPLAIN 分析执行计划**
   ```sql
   EXPLAIN SELECT * FROM orders WHERE status = 'pending';
   ```
   重点关注：
   - `type`: 最好达到 `ref`/`range`，避免 `ALL`（全表扫描）
   - `key`: 确认使用了索引
   - `rows`: 扫描行数，越少越好
   - `Extra`: 避免 `Using filesort`、`Using temporary`

3. **使用 SHOW PROFILE**
   ```sql
   SET profiling = 1;
   SELECT * FROM orders WHERE order_no = 'ORD20260622';
   SHOW PROFILES;
   SHOW PROFILE FOR QUERY 1;
   ```

**常用优化方法：**

| 优化方向 | 具体措施 |
|---------|---------|
| 索引优化 | 添加合适索引、使用覆盖索引、避免过多索引 |
| SQL 改写 | 避免 SELECT *、减少子查询、使用 JOIN 代替 |
| 分库分表 | 水平拆分大表、读写分离 |
| 业务层面 | 减少大事务、使用异步处理、限制返回结果 |

**典型案例：**
```sql
-- 优化前：多次子查询
SELECT * FROM users 
WHERE id IN (SELECT user_id FROM orders WHERE amount > 1000);

-- 优化后：使用 JOIN + 索引
SELECT DISTINCT u.* FROM users u
INNER JOIN orders o ON u.id = o.user_id
WHERE o.amount > 1000;
```

> 📚 来源：[GeeksforGeeks - MySQL Interview Questions](https://www.geeksforgeeks.org/mysql-interview-questions/)

---

## 题目三：InnoDB 和 MyISAM 存储引擎的区别？如何选择？

### 参考答案

| 特性 | InnoDB | MyISAM |
|-----|--------|--------|
| **事务支持** | ✅ 支持 ACID 事务 | ❌ 不支持 |
| **行级锁** | ✅ 支持行级锁 | ❌ 只支持表级锁 |
| **外键约束** | ✅ 支持 | ❌ 不支持 |
| **崩溃恢复** | ✅ 自动恢复（redo log） | ❌ 需手动修复 |
| **全文索引** | ✅ 5.6+ 支持 | ✅ 原生支持 |
| **存储结构** | 表空间 + frm | frm + MYD + MYI |
| **COUNT(*)** | 全表扫描 | 维护计数器 |
| **适用场景** | 事务优先、并发写入 | 读多写少、不需要事务 |

**InnoDB 核心优势：**
```sql
-- 支持事务回滚
START TRANSACTION;
UPDATE accounts SET balance = balance - 100 WHERE id = 1;
UPDATE accounts SET balance = balance + 100 WHERE id = 2;
-- 如果出错可以 ROLLBACK
ROLLBACK;
```

**选择建议：**
- **选择 InnoDB**：生产环境、订单系统、用户数据、需要并发写入
- **选择 MyISAM**：日志表、只读数据、静态配置表、需全文搜索

```sql
-- 查看当前存储引擎
SHOW CREATE TABLE users;

-- 修改存储引擎
ALTER TABLE users ENGINE = InnoDB;
```

> 📚 来源：[InterviewBit - PostgreSQL vs MySQL](https://www.interviewbit.com/blog/postgresql-vs-mysql/)

---

## 题目四：如何设计分页查询能保证性能？当 offset 很大时如何优化？

### 参考答案

**传统 OFFSET 分页的问题：**
```sql
-- 问题：OFFSET 越大，数据库扫描的行数越多
SELECT * FROM orders ORDER BY id LIMIT 1000000, 10;
-- 需要扫描 1000010 行，只返回 10 行
```

**优化方案：**

**方案一：使用游标分页（推荐）**
```sql
-- 上一页最后一条的 ID
SELECT * FROM orders 
WHERE id > 1000000  -- 上一页最后一条的 id
ORDER BY id 
LIMIT 10;
```

**方案二：子查询优化**
```sql
-- 利用索引覆盖，减少回表
SELECT * FROM orders 
WHERE id >= (SELECT id FROM orders ORDER BY id LIMIT 1000000, 1)
ORDER BY id 
LIMIT 10;
```

**方案三：延迟关联**
```sql
-- 先查索引列，再关联获取完整数据
SELECT o.* FROM orders o
INNER JOIN (SELECT id FROM orders ORDER BY id LIMIT 1000000, 10) AS t
ON o.id = t.id;
```

**方案四：记录总页数优化**
```sql
-- 缓存总数，只在必要时查询
-- 使用触发器或定时任务更新 count 表
SELECT COUNT(*) FROM orders; -- 定期执行，结果缓存
```

**性能对比：**

| 方案 | OFFSET 10万 | OFFSET 100万 |
|-----|------------|-------------|
| 传统 LIMIT | ~500ms | ~5s |
| 游标分页 | ~5ms | ~5ms |
| 延迟关联 | ~50ms | ~50ms |

> 📚 来源：[MySQL 性能优化 - 真实面试经验整理](https://learnku.com/articles/44630)

---

## 题目五：Redis 缓存与 MySQL 如何配合使用？有哪些经典问题需要注意？

### 参考答案

**经典缓存架构：Cache-Aside 模式**
```sql
-- 读操作：先查缓存，缓存未命中再查数据库
function getUser(id):
    user = redis.get(f"user:{id}")
    if user is null:
        user = mysql.query("SELECT * FROM users WHERE id = ?", id)
        redis.setex(f"user:{id}", 3600, user)  # 缓存 1 小时
    return user

-- 写操作：先更新数据库，再删除缓存
function updateUser(id, data):
    mysql.execute("UPDATE users SET ... WHERE id = ?", id)
    redis.del(f"user:{id}")  # 删除缓存，而不是更新
```

**三大经典问题：**

**1. 缓存穿透（Cache Penetration）**
- 问题：恶意请求不存在的数据，导致直击数据库
- 解决：
  ```sql
  -- 布隆过滤器
  IF bloomFilter.exists(key):
      return null  -- 直接返回，不查数据库
  ```

**2. 缓存击穿（Cache Breakdown）**
- 问题：热点 key 过期瞬间，大量请求击穿到数据库
- 解决：互斥锁 / 永不过期 + 异步更新
  ```python
  # 使用分布式锁
  lock = redis.setnx("lock:user:1", "1")
  if lock:
      user = mysql.query(...)
      redis.setex("user:1", 3600, user)
      redis.del("lock:user:1")
  else:
      sleep(0.1)
      return getUserFromCache(id)  # 等待后重试
  ```

**3. 缓存雪崩（Cache Avalanche）**
- 问题：大量 key 同时过期或 Redis 宕机
- 解决：
  ```python
  # 过期时间加随机值
  expire_time = 3600 + random.randint(0, 300)
  redis.setex(key, expire_time, value)
  
  # Redis 集群 + 哨兵保证高可用
  ```

**缓存更新策略：**

| 策略 | 优点 | 缺点 | 适用场景 |
|-----|------|------|---------|
| **Cache-Aside** | 简单、低代码侵入 | 可能数据不一致 | 读多写少 |
| **Write-Through** | 强一致性 | 写入延迟高 | 写入频繁 |
| **Write-Behind** | 写入性能高 | 可能丢数据 | 日志、统计 |

**数据一致性方案：**
```sql
-- 使用 Canal 监听 MySQL binlog 异步更新 Redis
-- 或使用 MQ 保证最终一致性
```

> 📚 来源：[Redis 设计与实现 - 缓存经典问题](https://redis.io/topics/lru-cache)

---

## 📖 更多学习资源

- [MySQL 官方文档](https://dev.mysql.com/doc/refman/8.0/en/)
- [PostgreSQL 官方文档](https://www.postgresql.org/docs/)
- [Redis 官方文档](https://redis.io/documentation)

---

*由 AI 自动整理生成 | 每日定时推送*