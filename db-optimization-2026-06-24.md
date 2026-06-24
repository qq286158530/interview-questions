# 数据库性能优化面试题

> 📅 整理日期：2026-06-24
> 🐼 来源：整理自网络资源 + 知识积累

---

## 题目一：MySQL索引失效的场景有哪些？如何避免？

### 参考答案

**索引失效的常见场景：**

1. **使用函数或运算**
   ```sql
   -- 失效
   SELECT * FROM users WHERE YEAR(created_at) = 2026;
   -- 正确做法
   SELECT * FROM users WHERE created_at >= '2026-01-01' AND created_at < '2027-01-01';
   ```

2. **隐式类型转换**
   ```sql
   -- phone字段是varchar但传入数字，索引失效
   SELECT * FROM users WHERE phone = 13800138000;
   -- 正确
   SELECT * FROM users WHERE phone = '13800138000';
   ```

3. **LIKE以通配符开头**
   ```sql
   -- 失效：'%abc' 无法使用索引
   SELECT * FROM users WHERE name LIKE '%abc';
   -- 可以使用索引
   SELECT * FROM users WHERE name LIKE 'abc%';
   ```

4. **OR连接不同类型列**
   ```sql
   -- 失效
   SELECT * FROM users WHERE id = '1' OR phone = '13800138000';
   -- 正确：为OR条件分别建立索引，或使用UNION
   ```

5. **范围列后不使用索引**
   ```sql
   -- age是索引列，status不是
   -- 可能只使用age的索引，status无法使用
   SELECT * FROM users WHERE age > 18 AND status = 1;
   ```

6. **不等于比较**
   ```sql
   -- != 和 <> 会导致索引失效
   SELECT * FROM users WHERE status != 1;
   ```

7. **NOT NULL/IS NOT NULL**
   ```sql
   -- 可能导致索引失效
   SELECT * FROM users WHERE name IS NOT NULL;
   ```

**如何避免索引失效：**
- 避免在索引列上使用函数、运算
- 隐式类型转换会导致索引失效，类型要匹配
- LIKE查询优先使用前缀匹配
- 尽量使用覆盖索引（covering index）
- 避免使用OR，改用UNION或IN
- 范围查询放在索引最后

---

## 题目二：如何优化慢SQL？说说你的思路和常用命令。

### 参考答案

**优化慢SQL的步骤：**

### 1. 定位慢查询
```sql
-- 查看慢查询日志配置
SHOW VARIABLES LIKE 'slow_query%';
SHOW VARIABLES LIKE 'long_query_time';

-- 开启慢查询日志
SET GLOBAL slow_query_log = 'ON';
SET GLOBAL long_query_time = 1;

-- 查看慢查询日志内容
mysqldumpslow -s t -t 10 /var/lib/mysql/slow.log
```

### 2. 使用EXPLAIN分析
```sql
EXPLAIN SELECT * FROM users WHERE phone = '13800138000';
EXPLAIN ANALYZE SELECT * FROM users WHERE phone = '13800138000';  -- MySQL 8.0+
```

**EXPLAIN关键字段解读：**
- **type**: 访问类型，const > eq_ref > ref > range > index > ALL（尽量达到ref级别）
- **key**: 实际使用的索引
- **rows**: 扫描行数，越少越好
- **Extra**: Using filesort/Using temporary（需要优化）

### 3. 常用优化手段

| 优化方向 | 具体方法 |
|---------|---------|
| 索引优化 | 添加合适索引，使用覆盖索引，避免索引失效 |
| SQL优化 | 避免SELECT *，减少JOIN，合理分页 |
| 结构优化 | 垂直/水平分表，分库分表 |
| 参数优化 | 调整buffer pool大小，连接池配置 |

### 4. 分页优化
```sql
-- 低效：OFFSET大时性能差
SELECT * FROM orders LIMIT 100000, 20;

-- 高效：基于主键的延迟加载
SELECT * FROM orders WHERE id > 100000 LIMIT 20;
```

### 5. 避免全表扫描
- 确保WHERE条件有索引
- 使用EXPLAIN确认索引被使用
- 监控慢查询，持续优化

---

## 题目三：MySQL InnoDB和MyISAM存储引擎的区别？如何选择？

### 参考答案

| 特性 | InnoDB | MyISAM |
|------|--------|--------|
| **事务支持** | 支持ACID事务 | 不支持 |
| **锁粒度** | 行级锁 | 表级锁 |
| **外键** | 支持 | 不支持 |
| **全文索引** | 5.6+支持 | 支持 |
| **崩溃恢复** | 自动恢复 | 需手动修复 |
| **存储结构** | 表空间 | 三个文件(.frm/.MYD/.MYI) |
| **COUNT(*)** | 全表扫描 | 存储记录数 |
| **索引** | 聚集索引 | 非聚集索引 |

**InnoDB核心优势：**
1. **事务安全**：支持COMMIT、ROLLBACK
2. **并发性能**：行级锁，支持高并发
3. **数据完整性**：外键约束
4. **崩溃恢复**：redo log自动恢复

**MyISAM适用场景：**
- 只读数据，或以读为主
- 不需要事务
- 需要全文索引（早期版本）
- 表级锁可接受

**选择建议：**
- **默认选择InnoDB**：现代MySQL默认引擎，适合绝大多数场景
- 日志系统、订单系统等核心业务必须用InnoDB
- 静态配置表、历史数据可用MyISAM

---

## 题目四：数据库缓存策略有哪些？如何设计缓存架构？

### 参考答案

### 缓存策略分类

**1. Cache-Aside（旁路缓存）- 最常用**
```
读：先读缓存，命中返回；未命中读数据库，写入缓存
写：先写数据库，再删除缓存
```

**2. Read-Through（读穿透）**
```
应用只读缓存，缓存负责读库并写入缓存
```

**3. Write-Through（写穿透）**
```
同时写数据库和缓存，保证一致性
```

**4. Write-Behind（异步写）**
```
先写缓存，异步批量写数据库，性能最高但有丢失风险
```

### Redis缓存架构设计

```sql
-- 缓存键设计
user:info:{userId}          -- 用户信息
product:list:{page}         -- 分页列表
product:detail:{productId}  -- 商品详情

-- 过期时间设计
热点数据：TTL 5-30分钟
普通数据：TTL 30分钟-2小时
```

### 缓存问题及解决方案

| 问题 | 原因 | 解决方案 |
|------|------|---------|
| 缓存穿透 | 查询不存在数据 | 布隆过滤器/null值缓存 |
| 缓存击穿 | 热点key过期 | 互斥锁/永不过期+异步更新 |
| 缓存雪崩 | 大量key同时过期 | 过期时间+随机值 |
| 数据一致性 | DB更新后缓存未更新 | 删除缓存而非更新 |

### 多级缓存架构
```
CPU L1/L2 Cache → Redis → MySQL
     ↑              ↑         ↑
   本地缓存      分布式缓存    磁盘
```

### 实践建议
```sql
-- 使用Pipeline减少网络往返
redis-cli --pipe < commands.txt

-- 批量获取
MGET user:info:1 user:info:2 user:info:3

-- 避免大Key
-- 单个Key不超过1MB，总内存控制在10G内
```

---

## 题目五：PostgreSQL相比MySQL有哪些性能优势？如何优化PostgreSQL？

### 参考答案

### PostgreSQL性能优势

1. **MVCC并发控制**：读写不阻塞，优于MySQL的锁机制
2. **索引类型丰富**：B-tree、Hash、GIN、GiST、BRIN等
3. **查询优化器更智能**：基于代价的优化器更先进
4. **表继承和分区表**：原生支持分区，性能优秀
5. **并行查询**：支持并行扫描、JOIN
6. **JSON/JSONB支持**：文档存储，高性能

### PostgreSQL优化手段

**1. 索引优化**
```sql
-- 创建适当索引
CREATE INDEX idx_users_email ON users(email);
CREATE INDEX idx_orders_created ON orders(created_at DESC);

-- 部分索引
CREATE INDEX idx_active_users ON users(email) WHERE status = 'active';

-- 表达式索引
CREATE INDEX idx_users_lower_email ON users(LOWER(email));

-- 查看索引使用
EXPLAIN ANALYZE SELECT * FROM users WHERE email = 'test@example.com';
```

**2. 分区表优化**
```sql
-- 创建分区表
CREATE TABLE orders (
    id BIGSERIAL,
    created_at TIMESTAMP NOT NULL
) PARTITION BY RANGE (created_at);

CREATE TABLE orders_2026 PARTITION OF orders
    FOR VALUES FROM ('2026-01-01') TO ('2027-01-01');

-- 分区裁剪优化查询
SET enable_partition_pruning = on;
```

**3. 统计信息与计划器**
```sql
-- 更新统计信息
ANALYZE orders;

-- 查看表大小
SELECT pg_size_pretty(pg_total_relation_size('orders'));
SELECT pg_size_pretty(pg_relation_size('orders'));

-- 查看慢查询
ALTER SYSTEM SET log_min_duration_statement = 1000;
SELECT pg_reload_conf();
```

**4. 配置参数调优**
```sql
-- 共享缓冲区（建议为内存的1/4）
ALTER SYSTEM SET shared_buffers = '4GB';

-- 工作内存（排序、哈希操作）
ALTER SYSTEM SET work_mem = '256MB';

-- 有效缓存大小（估算）
ALTER SYSTEM SET effective_cache_size = '12GB';

-- 并行worker进程数
ALTER SYSTEM SET max_worker_processes = 8;
ALTER SYSTEM SET max_parallel_workers_per_gather = 4;
```

**5. 查询优化建议**
- 使用EXPLAIN ANALYZE分析执行计划
- 批量插入使用COPY命令
- 避免SELECT *，只取需要的列
- 使用JOIN替代子查询（优化器会自动优化）
- 善用LIMIT和分页

---

## 📚 参考来源

1. MySQL索引失效场景：https://dev.mysql.com/doc/refman/8.0/en/index-btree-hash.html
2. 慢SQL优化：https://dev.mysql.com/doc/refman/8.0/en/optimization.html
3. InnoDB引擎特性：https://dev.mysql.com/doc/refman/8.0/en/innodb-introduction.html
4. Redis缓存策略：https://redis.io/docs/manual/patterns/
5. PostgreSQL性能优化：https://www.postgresql.org/docs/current/performance-tips.html

---

> 🐼 关注公众号「面试题库」，获取更多优质面试题
> GitHub仓库：https://github.com/qq286158530/interview-questions
