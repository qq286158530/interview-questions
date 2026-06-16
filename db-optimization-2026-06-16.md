# 数据库性能优化面试题

> 📅 日期：2026-06-16  
> 🐼 出题人：小憨宝  
> 📚 涵盖：索引优化、查询优化、存储引擎、缓存策略、锁机制

---

## 题目一：MySQL 查询语句很慢，如何进行性能优化？

### 参考答案

**1. 使用 EXPLAIN 分析执行计划**
```sql
EXPLAIN SELECT * FROM orders WHERE user_id = 123;
```
重点关注：
- `type`：const、ref、range 最优，ALL 表示全表扫描
- `key`：实际使用的索引
- `rows`：扫描行数，越少越好
- `Extra`：Using filesort/Using temporary 表示需要优化

**2. 索引优化**
- 确保 WHERE 条件字段有合适索引
- 复合索引遵循最左前缀原则
- 避免在索引列上使用函数或做运算
- 避免使用 LIKE '%xxx%' 前导通配符

**3. SQL 语句优化**
- 避免 SELECT *，只查需要的字段
- 减少 JOIN，多用子查询或 UNION 替代
- 分解复杂查询，分步执行
- 批量操作替代循环单条

**4. 表结构优化**
- 字段类型尽量小（INT vs BIGINT）
- 适当反范式化，减少 JOIN
- 定期清理无用数据（归档/分表）

**5. 配置参数调优**
- `innodb_buffer_pool_size`：建议设为机器内存 70-80%
- `max_connections`：根据业务峰值调整
- `slow_query_log`：开启慢查询日志

**📎 来源**
- https://dev.mysql.com/doc/refman/8.0/en/using-explain.html
- https://www.mysql.com/products/enterprise/audit.html

---

## 题目二：什么是覆盖索引？如何利用覆盖索引提升查询性能？

### 参考答案

**覆盖索引定义**
覆盖索引是指一个索引包含了查询所需的全部字段，无需回表查询数据行。

**回表查询解释**
普通索引（如在 name 字段上建索引）只存储字段值和主键，查询其他字段时需要通过主键回表获取完整数据行。

**覆盖索引示例**
```sql
-- 表结构：user (id, name, age, email)
-- 查询：SELECT name, age FROM user WHERE name = '张三';

-- 优化前：需要回表
CREATE INDEX idx_name ON user(name);

-- 优化后：覆盖索引，无需回表
CREATE INDEX idx_name_age ON user(name, age);
```

**使用技巧**
- 使用 EXPLAIN 的 Extra 列检查是否出现 `Using index`
- 将 SELECT 的字段放入复合索引中
- 对于高频查询建立联合覆盖索引
- 注意索引不是越多越好，维护成本也要考虑

**适用场景**
- 高频查询的字段组合
- 分页查询：覆盖索引 + ORDER BY
- 统计查询：COUNT/AVG/SUM 等聚合

**📎 来源**
- https://dev.mysql.com/doc/refman/8.0/en/optimization-indexes.html
- https://www.cnblogs.com/winnerintermeddley/p/14949832.html

---

## 题目三：MySQL InnoDB 和 MyISAM 存储引擎的区别？如何选择？

### 参考答案

| 特性 | InnoDB | MyISAM |
|------|--------|--------|
| 事务支持 | ✅ 支持 ACID 事务 | ❌ 不支持 |
| 行锁 | ✅ 支持行级锁 | ❌ 只支持表锁 |
| 外键约束 | ✅ 支持 | ❌ 不支持 |
| 崩溃恢复 | ✅ 自动崩溃恢复 | ❌ 需手动修复 |
| 并发能力 | 高并发场景优秀 | 只适合读多写少 |
| 存储空间 | 同等数据约 2 倍 | 更紧凑 |
| 全文索引 | 5.6+ 支持 | 原生支持 |

**InnoDB 核心优势**
- 支持行级锁，高并发写入性能好
- 支持事务和外键，数据完整性高
- 崩溃自动恢复，数据安全有保障
- 支持 MVCC 隔离级别

**MyISAM 适用场景**
- 日志系统、报表系统（以读为主）
- 全文搜索场景
- 追求存储空间小
- 无事务要求的历史数据

**选择建议**
- **默认选 InnoDB**：现代 MySQL 默认引擎
- **高并发写入系统**：必须 InnoDB
- **数据安全要求高**：InnoDB
- **读远多于写且无事务**：MyISAM 可考虑

**📎 来源**
- https://dev.mysql.com/doc/refman/8.0/en/innodb-storage-engines.html
- https://www.cnblogs.com/keme/p/13612681.html

---

## 题目四：PostgreSQL 中如何优化大表分页查询的性能？

### 参考答案

**问题背景**
大表分页（OFFSET/LIMIT）在 offset 很大时性能急剧下降，因为数据库仍需扫描并丢弃前面的行。

**优化方案**

**方案一：游标分页（Keyset Pagination）**
```sql
-- 传统方式（慢）
SELECT * FROM orders ORDER BY id LIMIT 10 OFFSET 100000;

-- 优化方式（快）
SELECT * FROM orders 
WHERE id > last_seen_id 
ORDER BY id LIMIT 10;
```
利用索引定位，查询复杂度从 O(n) 降为 O(log n)。

**方案二：只查主键再关联**
```sql
-- 第一步：只查主键
SELECT id FROM orders ORDER BY id LIMIT 10 OFFSET 100000;

-- 第二步：关联获取完整数据
SELECT o.* FROM orders o
INNER JOIN (SELECT id FROM orders ORDER BY id LIMIT 10 OFFSET 100000) sub
ON o.id = sub.id;
```

**方案三：合理使用索引覆盖**
```sql
-- 确保排序字段有索引
CREATE INDEX idx_orders_id ON orders(id);
-- 如果需按时间排序
CREATE INDEX idx_orders_created ON orders(created_at DESC);
```

**方案四：调整执行计划**
```sql
-- 关闭_seqscan 提高索引使用
SET enable_seqscan = off;
-- 增加 work_mem 改善排序性能
SET work_mem = '256MB';
```

**📎 来源**
- https://www.postgresql.org/docs/current/using-explain.html
- https://www.cnblogs.com/xy294652009/p/15161934.html

---

## 题目五：Redis 缓存与 MySQL 数据库如何配合使用？缓存穿透、击穿、雪崩如何解决？

### 参考答案

**Redis + MySQL 配合模式**

**1. Cache-Aside（旁路缓存）模式**
```sql
# 读操作
data = redis.get(key)
if (data is null) {
    data = mysql.get(key)
    redis.setex(key, 3600, data)  # 设置过期时间
}

# 写操作
mysql.update(data)
redis.del(key)  # 删除缓存，让下次读取时重建
```

**2. 缓存穿透解决方案**
问题：查询不存在的数据，每次都穿透到数据库。

方案：
- 布隆过滤器：使用 Redis 的 Bitmap 存储所有合法 key
- 空值缓存：对查询为空的 key 也缓存一个空值，设置短过期时间
```python
if redis.get(key) is None:
    data = mysql.get(key)
    if data is None:
        redis.setex(key, 60, "NULL")  # 空值也缓存，60秒过期
```

**3. 缓存击穿解决方案**
问题：热点 key 过期瞬间，大量请求穿透到数据库。

方案：
- 互斥锁：只允许一个请求重建缓存
```python
lock = redis.setnx(lock_key, 1)
if lock:
    data = mysql.get(key)
    redis.setex(key, 3600, data)
    redis.del(lock_key)
```
- 永不过期：逻辑过期 + 异步更新

**4. 缓存雪崩解决方案**
问题：大量 key 同时过期或 Redis 宕机。

方案：
- 过期时间随机化：`TTL = base + random(0, 300)`
- 多级缓存：Redis + 本地缓存（如 Caffeine）
- Redis 集群 + 哨兵/主从自动切换
- 服务熔断降级：Hystrix 保护数据库

**📎 来源**
- https://redis.io/docs/manualpatterns/caching/
- https://blog.csdn.net/m0_37547121/article/details/125566018

---

> 📌 温馨提示：面试时不仅要有答案，最好能结合实际项目经验说明！