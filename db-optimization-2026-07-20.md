# 数据库性能优化面试题（2026-07-20）

> 本文件收录 5 道高质量 MySQL/PostgreSQL 数据库性能优化面试题，涵盖索引优化、查询优化、存储引擎、缓存策略等核心知识点。

---

## 题目一：MySQL 中索引失效的常见场景有哪些？

### 参考答案

索引失效是指明明建立了索引，但查询时 MySQL 却没有使用索引，导致全表扫描。以下是几种常见的索引失效场景：

**1. 使用函数或运算**
```sql
-- 索引失效：YEAR() 函数作用于索引列
SELECT * FROM orders WHERE YEAR(create_time) = 2026;

-- 正确做法：改造查询条件
SELECT * FROM orders WHERE create_time >= '2026-01-01' AND create_time < '2027-01-01';
```

**2. LIKE 开头通配符**
```sql
-- 索引失效：LIKE 以 % 开头
SELECT * FROM users WHERE name LIKE '%张%';

-- 索引有效：LIKE 以后缀匹配
SELECT * FROM users WHERE name LIKE '张%';
```

**3. 隐式类型转换**
```sql
-- phone 是 VARCHAR 类型，传入数字导致隐式转换
SELECT * FROM users WHERE phone = 13800138000;  -- 触发 CAST

-- 正确做法：使用字符串
SELECT * FROM users WHERE phone = '13800138000';
```

**4. OR 连接条件不匹配**
```sql
-- OR 两边有一列没有索引，导致全表扫描
SELECT * FROM users WHERE id = 1 OR email = 'a@b.com';

-- 建议拆分为 UNION 或确保所有列都有索引
SELECT * FROM users WHERE id = 1
UNION ALL
SELECT * FROM users WHERE email = 'a@b.com' AND id IS NULL;
```

**5. 组合索引不符合最左前缀原则**
```sql
-- 组合索引 INDEX(idx, name, age)
-- 索引失效：跳过 idx，直接使用 name
SELECT * FROM users WHERE name = '张三';

-- 索引有效
SELECT * FROM users WHERE idx = 1;
SELECT * FROM users WHERE idx = 1 AND name = '张三';
```

**6. 使用 NOT、!=、<> 操作符**
```sql
-- 索引失效：负向查询
SELECT * FROM users WHERE status != 1;
SELECT * FROM users WHERE status NOT IN (1, 2);
```

**7. ORDER BY 的坑**
```sql
-- 组合索引 ORDER BY 也需要遵循最左前缀
-- INDEX(a, b, c)
SELECT * FROM t ORDER BY b, c;  -- 不使用索引
SELECT * FROM t ORDER BY a, b;  -- 使用索引
```

---

## 题目二：Explain 执行计划各字段含义是什么？如何根据它优化慢查询？

### 参考答案

`EXPLAIN` 是分析 SQL 性能的核心工具，返回各字段含义如下：

| 字段 | 含义 |
|------|------|
| **id** | SELECT 查询的序列号，id 越大越先执行 |
| **select_type** | 查询类型：SIMPLE（简单查询）、PRIMARY（主查询）、SUBQUERY（子查询）、DERIVED（派生表） |
| **table** | 访问的表名 |
| **type** | 访问类型，常见：`const`（主键/唯一索引）→ `eq_ref` → `ref` → `range` → `index` → `ALL（全表扫描）` |
| **possible_keys** | 可能使用的索引 |
| **key** | 实际使用的索引 |
| **key_len** | 索引使用长度，越小越好 |
| **rows** | 预估计扫描的行数，越少越好 |
| **Extra** | 额外信息：`Using filesort`、`Using temporary`、`Using index` |

### 优化慢查询的实战步骤

**Step 1：检查 type 是否为 ALL（最糟糕）**
```sql
EXPLAIN SELECT * FROM orders WHERE status = 1;
-- type = ALL 说明全表扫描，需要加索引
```

**Step 2：检查是否出现 Using filesort（文件排序）**
```sql
-- 慢查询示例：未建立索引导致 filesort
EXPLAIN SELECT * FROM orders ORDER BY create_time DESC;
-- Extra: Using filesort

-- 优化：为排序字段建立索引
ALTER TABLE orders ADD INDEX idx_create_time(create_time);
```

**Step 3：检查是否出现 Using temporary（临时表）**
```sql
-- GROUP BY 或 DISTINCT 可能产生临时表
EXPLAIN SELECT name, COUNT(*) FROM users GROUP BY name;

-- 优化：在 GROUP BY 字段上建索引
ALTER TABLE users ADD INDEX idx_name(name);
```

**Step 4：检查 key_len 是否足够短**
- key_len 越大说明索引使用越不充分，考虑调整索引列顺序

**Step 5：检查 rows 是否过大**
- rows 反映预估计扫描行数，可以通过 `ANALYZE TABLE` 更新统计信息

---

## 题目三：MySQL InnoDB 与 MyISAM 存储引擎的核心区别是什么？如何选型？

### 参考答案

| 特性 | InnoDB | MyISAM |
|------|--------|--------|
| **事务支持** | 支持 ACID 事务（commit/rollback） | 不支持 |
| **行锁粒度** | 行级锁，并发性能好 | 表级锁，并发差 |
| **外键约束** | 支持外键 | 不支持 |
| **崩溃恢复** | 支持 crash-safe，有 redo log | 需手动修复 |
| **

全文索引** | 5.6+ 支持（FULLTEXT） | 原生支持 |
| **存储结构** | 聚簇索引（主键索引即数据） | 非聚簇索引（索引与数据分离） |
| **COUNT(*)** | 全表扫描（无内置计数器） | 有内置计数器，快 |
| **适用场景** | 核心业务、高并发、需事务 | 读多写少、不需要事务 |

### 面试加分点：聚簇索引 vs 非聚簇索引

```sql
-- InnoDB（聚簇索引）
-- 主键叶子节点直接存储完整行数据
-- 二级索引叶子节点存储主键值

-- MyISAM（非聚簇索引）
-- 所有索引叶子节点存储的是行数据的物理地址
```

### 选型建议

- **用 InnoDB**：几乎所有场景都优先选 InnoDB，现代 MySQL 8.0+ 的默认引擎
- **用 MyISAM**：仅当有极致 SELECT 性能需求（无并发写入、无事务）且数据可丢失间接场景

---

## 题目四：如何设计一个合理的分页查询？深度分页为什么会慢？

### 参考答案

### 普通 LIMIT 分页的问题

```sql
-- 第 10000 页，每页 20 条
SELECT * FROM orders LIMIT 200000, 20;
```

深度分页（offset 很大）会先扫描前 200000 条记录再丢弃，非常低效。

### 优化方案一：延迟关联（子查询）

```sql
-- 优化：用子查询先定位 ID，再关联
SELECT o.* FROM orders o
INNER JOIN (
    SELECT id FROM orders
    ORDER BY create_time DESC
    LIMIT 200000, 20
) t ON o.id = t.id;
```

### 优化方案二：游标分页（Keyset Pagination间接）

```sql
-- 记录上一页最后一条的 ID 或时间戳
SELECT * FROM orders
WHERE id < #{last_id}   -- 上一页最后一条的 ID
ORDER BY id DESC
LIMIT 20;
```

### 优化方案三：降级处理

- 超过指定页数（如 > 1000 页)直接返回错误或不允许跳页
- 配合总页数缓存，减少 COUNT(*) 的开销

### 优化方案四：覆盖索引

确保分页查询的所有字段都在索引中，MySQL 可直接返回索引而不回表：

```sql
-- 假设 status 和 create_time 在索引中
SELECT id, status, create_time
FROM orders
WHERE status = 1
ORDER BY create_time DESC
LIMIT 200000, 20;
-- Extra: Using index（索引覆盖，无需回表）
```

---

## 题目五：Redis 缓存与 MySQL 如何配合使用？缓存穿透、缓存击穿、缓存雪崩是什么？如何解决？

### 参考答案

### 常见缓存架构：Cache-Aside

```
读：先查 Redis，未命中 → 查 MySQL → 回填 Redis → 返回
写：先更新 MySQL → 删除 Redis（而非更新，避免不一致）
```

### 缓存穿透

**问题**：大量请求查询一个不存在的数据（key 不存在），绕过缓存直击数据库。

**解决**：
1. **布隆过滤器（Bloom Filter）**：将所有合法 key 存入布隆过滤器，拦截非法请求
2. **缓存空值**：将 NULL 结果也存入 Redis，并设置较短过期时间（如 30s）
3. **参数校验**：做好请求参数合法性校验

### 缓存击穿

**问题**：热点 key 过期瞬间，大量并发请求直接打到数据库。

**解决**：
1. **互斥锁（Mutex）**：只允许一个请求回填缓存，其他等待
   ```sql
   -- Redis SETNX 实现
   SET lock_key value NX EX 10
   ```
2. **永不过期**：热点数据不设置过期时间，由后台异步更新
3. **逻辑过期**：key 中存储数据 + 逻辑过期时间，过期时先返回旧数据再异步更新

### 缓存雪崩

**问题**：大量 key 同时过期，或 Redis 宕机，导致数据库压力暴增。

**解决**：
1. **过期时间随机化**：给 key 的 TTL 加随机偏移量
   ```sql
   SET key value EX (3600 + rand() % 300)
   ```
2. **Redis 高可用**：部署 Redis Cluster 或哨兵，保证可用性
3. **多级缓存**：本地缓存（如 Caffiene）+ Redis + MySQL，多层拦截
4. **流量控制**：使用限流（Redis + Lua 或更加“gateway）削减峰值流量
5. **提前预热**：系统启动时主动加载热点数据到缓存

### 缓存一致性问题

**删除而非更新**：先更新数据库再删除缓存，若删除失败则产生不一致。

**解决**：采用**延迟双删**策略
```sql
1. 删除 Redis
2. 更新 MySQL
3. （异步）休眠 N ms 后再删除 Redis（清除脏数据）
```

---

> 📌 以上题目均经过筛选，适合中高级工程师面试准备。建议结合自身项目经验补充细节。
