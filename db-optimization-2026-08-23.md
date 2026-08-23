# 数据库性能优化面试题

> 📅 推送日期：2026-08-23  
> 📚 内容来源：精选自互联网技术论坛与面试题库

---

## 题目一：MySQL 索引失效的场景有哪些？

### 参考答案

**常见的索引失效场景：**

1. **使用函数或运算**：在索引列上使用函数、运算或表达式，如 `WHERE YEAR(create_time) = 2026`
2. **类型转换**：字符串列与数字比较，如 `WHERE phone = 1380013800`（phone 是 varchar）
3. **LIKE 以通配符开头**：如 `WHERE name LIKE '%张'`，导致全表扫描
4. **复合索引未遵循最左前缀原则**：如创建了 `(a,b,c)` 索引，但查询只引用 `b` 或 `c`
5. **使用 OR 连接非索引列**：如 `WHERE name = '张三' OR age = 20`，其中 age 无索引
6. **IS NOT NULL / IS NULL**：部分场景下优化器可能放弃使用索引
7. **不等于比较**：`<>`、`NOT IN`、`NOT BETWEEN` 可能导致索引失效
8. **数据量过小时**：优化器认为全表扫描更快，可能不使用索引

**优化建议：**
- 尽量使用复合索引时遵循最左前缀原则
- 避免在索引列上使用函数
- 尽量使用覆盖索引减少回表

---

## 题目二：如何定位并优化慢查询？

### 参考答案

**定位慢查询的步骤：**

1. **开启慢查询日志**
   ```sql
   -- 查看慢查询配置
   SHOW VARIABLES LIKE 'slow_query%';
   SHOW VARIABLES LIKE 'long_query_time';
   
   -- 开启慢查询日志
   SET GLOBAL slow_query_log = 'ON';
   SET GLOBAL long_query_time = 1; -- 设置阈值为1秒
   ```

2. **使用 EXPLAIN 分析执行计划**
   ```sql
   EXPLAIN SELECT * FROM orders WHERE status = 1;
   ```
   重点关注：
   - `type`：最好达到 `ref`/`range`，避免 `ALL`（全表扫描）
   - `key`：实际使用的索引
   - `rows`：扫描的行数
   - `Extra`：Using filesort、Using temporary 等警告信息

3. **使用性能分析工具**
   - MySQL：`SHOW PROFILE`、Performance Schema
   - PostgreSQL：`EXPLAIN (ANALYZE, BUFFERS)`

**优化手段：**
- 添加合适索引
- 优化 SQL 结构（避免 SELECT *、减少嵌套子查询）
- 拆分大表（分区表）
- 读写分离、分库分表

---

## 题目三：MySQL InnoDB 与 MyISAM 的区别？如何选择？

### 参考答案

| 特性 | InnoDB | MyISAM |
|------|--------|--------|
| 事务支持 | ✅ 支持 ACID 事务 | ❌ 不支持 |
| 行锁 | ✅ 支持行级锁 | ❌ 只支持表锁 |
| 外键 | ✅ 支持 | ❌ 不支持 |
| 崩溃恢复 | ✅ 支持自动恢复 | ❌ 需手动修复 |
| 全文索引 | 5.6+ 支持 | ✅ 原生支持 |
| 存储结构 | 聚簇索引（数据与主键在一起） | 非聚簇索引（索引与数据分离） |
| 适用场景 | 高并发、事务需求 | 读多写少、不需要事务 |

**选择建议：**
- **InnoDB**：生产环境首选，适合几乎所有业务场景
- **MyISAM**：仅适用于日志表、统计类只读表等特殊场景

**补充：** PostgreSQL 使用 MVCC 机制实现事务，与 InnoDB 的实现原理有本质区别，在高并发场景下表现更稳定。

---

## 题目四：什么是数据库缓存策略？如何设计 Redis + MySQL 缓存？

### 参考答案

**缓存策略模式：**

1. **Cache-Aside（旁路缓存）** — 最常用
   - 读：先查缓存，命中则返回；未命中查数据库并写入缓存
   - 写：先更新数据库，再删除缓存（而非更新缓存）

2. **Read-Through（读穿透）**
   - 缓存自动加载数据，应用只与缓存交互

3. **Write-Through（写穿透）**
   - 同步写入缓存和数据库

**缓存问题及解决方案：**

| 问题 | 解决方案 |
|------|----------|
| 缓存穿透 | 布隆过滤器 / 空值缓存 |
| 缓存击穿 | 互斥锁 / 热点数据永不过期 |
| 缓存雪崩 | 随机过期时间 / 多级缓存 / 高可用架构 |

**示例代码逻辑：**
```python
def get_user(user_id):
    # 1. 查缓存
    user = redis.get(f"user:{user_id}")
    if user:
        return json.loads(user)
    
    # 2. 缓存未命中，查数据库
    user = db.query("SELECT * FROM users WHERE id = ?", user_id)
    
    # 3. 写入缓存
    if user:
        redis.setex(f"user:{user_id}", 3600, json.dumps(user))
    
    return user
```

---

## 题目五：PostgreSQL 中如何优化大表的分页查询？

### 参考答案

**问题背景：**
`SELECT * FROM orders ORDER BY id LIMIT 1000000, 10` 在大表上非常慢，因为要扫描并排序前 100 万条数据。

**优化方案：**

**方案一：使用游标（Keyset Pagination）**
```sql
-- 第一次查询
SELECT * FROM orders 
WHERE id > 1000000 
ORDER BY id 
LIMIT 10;

-- 记录最后一行的 id，下次查询传入
SELECT * FROM orders 
WHERE id > ? 
ORDER BY id 
LIMIT 10;
```
时间复杂度从 O(n) 降为 O(1)，性能稳定不随页码增加而下降。

**方案二：使用覆盖索引**
```sql
-- 创建覆盖索引
CREATE INDEX idx_orders_id_cover ON orders(id) INCLUDE (status, amount);

-- 利用索引覆盖，避免回表
SELECT id FROM orders ORDER BY id LIMIT 1000000, 10;
```

**方案三：并行查询**
```sql
SET max_parallel_workers_per_gather = 4;
SELECT * FROM orders ORDER BY id LIMIT 1000000, 10;
```

**方案四：分表策略**
按时间或业务 ID 哈希分表，将大表拆分为多个小表，缩小查询范围。

---

> 📌 **推荐学习资源**
> - MySQL 官方文档：https://dev.mysql.com/doc/
> - PostgreSQL 官方文档：https://www.postgresql.org/docs/
> - 互联网技术面试题库（牛客网/LeetCode 讨论区）
