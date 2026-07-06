# 数据库性能优化面试题

> 📅 整理日期：2026-07-06  
> 🐼 来源：综合整理自MySQL/PostgreSQL官方文档及优质技术社区

---

## 题目一：MySQL索引失效的场景有哪些？如何避免？

### 题目
MySQL中哪些情况会导致索引失效？请举例说明如何避免索引失效。

### 答案

**常见的索引失效场景：**

1. **使用函数或运算**
   ```sql
   -- 索引失效
   SELECT * FROM users WHERE YEAR(created_at) = 2026;
   
   -- 正确写法：使用范围查询
   SELECT * FROM users WHERE created_at >= '2026-01-01' AND created_at < '2027-01-01';
   ```

2. **使用LIKE以通配符开头**
   ```sql
   -- 索引失效
   SELECT * FROM users WHERE name LIKE '%张%';
   
   -- 正确写法：LIKE前缀通配
   SELECT * FROM users WHERE name LIKE '张%';
   ```

3. **类型转换**
   ```sql
   -- id为字符串类型，以下查询索引失效
   SELECT * FROM users WHERE id = 123;  -- 隐式类型转换
   
   -- 正确写法
   SELECT * FROM users WHERE id = '123';
   ```

4. **OR连接条件**
   ```sql
   -- 索引失效（OR前后有一个字段没索引）
   SELECT * FROM users WHERE name = '张三' OR age = 18;
   
   -- 正确写法：使用UNION
   SELECT * FROM users WHERE name = '张三'
   UNION ALL
   SELECT * FROM users WHERE age = 18 AND name != '张三';
   ```

5. **不等于比较**
   ```sql
   -- 索引失效
   SELECT * FROM users WHERE status != 1;
   ```

6. **IN子句包含过多元素**
   当IN中元素过多时，优化器可能放弃使用索引。

**避免索引失效的建议：**
- 尽量使用覆盖索引（covering index）
- 避免在索引列上使用函数
- 使用复合索引时遵循最左前缀原则
- 避免隐式类型转换
- 减少OR使用，改用UNION

---

## 题目二：如何分析一条SQL语句的性能？EXPLAIN命令各字段含义是什么？

### 题目
当遇到SQL查询慢的情况时，你是如何分析的？请解释EXPLAIN输出的关键字段。

### 答案

**分析SQL性能的步骤：**

1. 使用`EXPLAIN`或`EXPLAIN ANALYZE`分析执行计划
2. 检查是否有合理的索引
3. 分析慢查询日志
4. 使用`SHOW PROFILE`进行 profiling

**EXPLAIN关键字段详解：**

| 字段 | 含义 |
|------|------|
| **type** | 连接类型，效率从高到低：system > const > eq_ref > ref > range > index > ALL |
| **key** | 实际使用的索引 |
| **rows** | 预计扫描的行数，越少越好 |
| **Extra** | 额外信息，Using filesort/Using temporary表示需要优化 |

**示例分析：**
```sql
EXPLAIN SELECT * FROM orders WHERE user_id = 100 AND status = 1;

-- 理想结果：
-- type: ref (使用索引)
-- key: idx_user_status (使用复合索引)
-- rows: 10 (扫描少量行)
-- Extra: Using index condition (使用索引)
```

**优化建议：**
- type至少达到ref级别，避免index或ALL
- rows过大考虑添加索引或优化SQL
- Extra避免Using filesort和Using temporary

---

## 题目三：MySQL InnoDB引擎的行锁是如何实现的？什么时候会升级为表锁？

### 题目
请描述InnoDB行锁的实现原理，以及什么情况下会发生锁升级。

### 答案

**InnoDB行锁实现原理：**

InnoDB行锁是通过**索引记录锁（Index Record Locks）**实现的。当使用二级索引进行查询时，InnoDB会对匹配的二级索引记录加锁，同时对对应的聚簇索引记录加锁。

```sql
-- 假设age字段有索引
SELECT * FROM users WHERE age = 18 FOR UPDATE;
```

InnoDB会：
1. 在age索引上加锁
2. 同时在主键聚簇索引上加锁

**行锁升级为表锁的情况：**

1. **无索引更新**
   ```sql
   -- 如果name没有索引，会锁全表
   UPDATE users SET age = 20 WHERE name = '张三';
   ```

2. **事务隔离级别为SERIALIZABLE**
   - 当隔离级别为SERIALIZABLE时，普通SELECT也会加锁

3. **锁等待超时**
   - 当某个事务持有锁过多行时，可能触发锁升级

4. **SQL语句不走索引**
   - 全表扫描时对每行加锁，效率低

**避免锁升级的建议：**
- 确保查询条件有合适索引
- 控制事务大小，缩短持锁时间
- 避免不加索引的WHERE条件
- 合理设置`innodb_lock_wait_timeout`

---

## 题目四：PostgreSQL和MySQL在性能优化方面有什么区别？

### 题目
作为后端工程师，如果你需要从MySQL迁移到PostgreSQL，在性能优化方面需要注意哪些差异？

### 答案

**架构差异：**

| 特性 | MySQL | PostgreSQL |
|------|-------|------------|
| 架构 | 单进程多线程 | 多进程架构 |
| MVCC | 在InnoDB中实现 | 原生支持MVCC |
| 索引类型 | B+Tree为主 | B+Tree、GiST、GIN、BRIN等 |
| 锁粒度 | 行级锁 + 表级锁 | 行级锁 + 谓词锁 |

**PostgreSQL特有的优化点：**

1. **丰富的索引类型**
   ```sql
   -- GiST索引：适合几何数据、全文搜索
   CREATE INDEX idx ON documents USING GIN(to_tsvector('english', content));
   
   -- BRIN索引：适合物理存储有序的大表
   CREATE INDEX idx ON logs USING BRIN(created_at);
   ```

2. **物化视图**
   ```sql
   -- 创建物化视图
   CREATE MATERIALIZED VIEW daily_sales AS
   SELECT DATE(created_at) as date, SUM(amount) as total
   FROM orders GROUP BY DATE(created_at);
   
   -- 刷新数据
   REFRESH MATERIALIZED VIEW daily_sales;
   ```

3. **并行查询**
   ```sql
   -- PostgreSQL自动利用多核
   SET max_parallel_workers_per_gather = 4;
   ```

4. **EXPLAIN ANALYZE**
   PostgreSQL的EXPLAIN更详细，支持实际执行计划：
   ```sql
   EXPLAIN (ANALYZE, BUFFERS, FORMAT TEXT) SELECT * FROM orders;
   ```

**迁移注意事项：**
- PostgreSQL对NULL的处理与MySQL不同
- 字符串拼接使用`||`而非`CONCAT`
- 分页使用`OFFSET FETCH`而非`LIMIT`
- 需要关注慢查询日志配置（`log_min_duration_statement`）

---

## 题目五：如何设计一个高效的数据库缓存策略？Redis和数据库如何配合使用？

### 题目
在高并发场景下，如何设计缓存策略来提升数据库性能？请从缓存模式、缓存粒度、缓存失效等方面说明。

### 答案

**缓存模式：**

1. **Cache Aside（旁路缓存）** - 最常用
   ```
   读：先读缓存，缓存未命中则读DB并写入缓存
   写：先写DB，删除缓存（而非更新）
   ```

2. **Read Through**
   缓存自动加载数据，应用只与缓存交互

3. **Write Through**
   写操作同时更新缓存和数据库

**缓存粒度选择：**

| 粒度 | 适用场景 | 优点 | 缺点 |
|------|----------|------|------|
| 缓存整个对象 | 对象较小 | 简单 | 可能浪费空间 |
| 缓存对象字段 | 选择性读取 | 灵活 | 实现复杂 |
| 缓存查询结果 | 列表查询 | 高效 | 通用性差 |

**缓存失效策略：**

```python
# Python伪代码：Cache Aside模式
def get_user(user_id):
    cache_key = f"user:{user_id}"
    user = redis.get(cache_key)
    if user:
        return json.loads(user)
    
    user = db.query("SELECT * FROM users WHERE id = ?", user_id)
    if user:
        redis.setex(cache_key, 3600, json.dumps(user))  # TTL 1小时
    return user

def update_user(user_id, data):
    db.execute("UPDATE users SET ... WHERE id = ?", user_id)
    redis.delete(f"user:{user_id}")  # 删除而非更新
```

**缓存问题及解决方案：**

1. **缓存穿透**
   - 布隆过滤器 / 缓存空值

2. **缓存击穿**
   - 互斥锁 / 永不过期 + 异步更新

3. **缓存雪崩**
   - 过期时间加随机值 / Redis集群

**配置建议：**
```bash
# Redis内存淘汰策略
maxmemory-policy allkeys-lru

# 缓存过期时间
EXPIRE user:123 3600  # 1小时
EXPIRE session:abc 1800  # 30分钟
```

---

## 参考资料

1. [MySQL 8.0 Reference Manual - Optimizing Queries](https://dev.mysql.com/doc/refman/8.0/en/optimization.html)
2. [PostgreSQL Documentation - Performance Tips](https://www.postgresql.org/docs/current/performance-tips.html)
3. [MySQL索引失效场景详解](https://www.cnblogs.com/songfayun/articles/14041424.html)
4. [Redis设计与实现](https://redisbook.com/)
5. [InnoDB锁机制深入分析](https://www.mysqlzh.com/#/doc/3)

---

> 📌 **小憨宝提示**：面试中除了掌握理论，一定要结合实际项目经验来回答哦！建议准备2-3个自己亲手优化的案例。
