# 数据库性能优化面试题

> 📅 更新时间：2026-05-21  
> 🏷️ 分类：MySQL / PostgreSQL  
> 🎯 难度：中级 ~ 高级

---

## 题目 1：MySQL 索引失效的场景有哪些？如何避免？

### 参考答案

索引失效是面试中的高频问题，主要有以下几种场景：

**1. 违背最左前缀原则**
```sql
-- idx(a, b, c)，查询条件不包含最左前缀
WHERE b = 1    -- ❌ 索引失效（跳过了 a）
WHERE a = 1 AND c = 2  -- ⚠️ c 无法使用索引（b 被跳过）
```

**2. 使用函数或表达式**
```sql
WHERE YEAR(created_at) = 2024   -- ❌ 索引失效
WHERE created_at >= '2024-01-01'  -- ✅ 正确写法
```

**3. 隐式类型转换**
```sql
-- phone 是 varchar，但传入整数
WHERE phone = 13000000000  -- ❌ 触发隐式转换，索引失效
WHERE phone = '13000000000'  -- ✅ 正确
```

**4. LIKE 以通配符开头**
```sql
WHERE name LIKE '%王'   -- ❌ 索引失效
WHERE name LIKE '王%'   -- ✅ 可以使用索引
```

**5. 范围查询后的列无法使用索引**
```sql
-- idx(a, b, c)
WHERE a > 5 AND b = 1  -- ❌ a 是范围，b 索引失效
WHERE a = 5 AND b > 1  -- ✅ b 可以使用索引
```

**6. 使用 OR 连接且两边字段不同**
```sql
WHERE a = 1 OR b = 2  -- ❌ 索引失效（除非 OR 两边都有索引）
WHERE a = 1 OR a = 2  -- ✅ 可以使用索引
```

### 避坑建议
- 建立复合索引时，考虑查询频率，合理安排列顺序
- 避免在 WHERE 中对索引列做运算
- 用 EXPLAIN 分析执行计划，确认索引是否被用到

**来源：** 高性能MySQL（第三版）、MySQL 8.0官方文档

---

## 题目 2：一条 SQL 执行很慢，如何定位问题？请描述排查思路。

### 参考答案

**Step 1：确认是偶发还是持续**

```sql
-- 查看当前连接和正在执行的SQL
SHOW PROCESSLIST;
-- 或查询 performance_schema
SELECT * FROM information_schema.PROCESSLIST WHERE COMMAND != 'Sleep';
```

**Step 2：用 EXPLAIN 分析执行计划**

```sql
EXPLAIN SELECT ...
-- 重点关注：
-- type: 尽可能接近 const/eq_ref/ref
-- key: 实际使用了哪个索引
-- rows: 扫描行数，越少越好
-- Extra: 是否出现 filesort、temporary 等警告
```

**Step 3：检查索引使用情况**

```sql
-- 查看表的所有索引
SHOW CREATE TABLE orders;

-- 强制使用某个索引（诊断用）
SELECT * FROM orders USE INDEX (idx_date) WHERE ...
```

**Step 4：分析慢查询日志**

```ini
# my.cnf 配置
slow_query_log = 1
slow_query_log_file = /var/log/mysql/slow.log
long_query_time = 1  -- 超过1秒记录
```

**Step 5：常见原因及解决方案**

| 原因 | 表现 | 解决方案 |
|------|------|----------|
| 缺少索引 | rows 扫描量巨大 | 添加合适的索引 |
| 统计信息过期 | rows 不准确 | ANALYZE TABLE |
| 关联顺序错误 | 多表关联效率低 | 调整 JOIN 顺序或添加提示 |
| 深分页 | OFFSET 大时极慢 | 改用子查询或延迟关联 |
| 锁等待 | State 显示 locked | 减少事务时长，降低隔离级别 |

**来源：** 极客时间《MySQL实战45讲》、Percona Toolkit

---

## 题目 3：MySQL InnoDB 和 MyISAM 存储引擎的区别？如何选择？

### 参考答案

| 特性 | InnoDB | MyISAM |
|------|--------|--------|
| 事务支持 | ✅ ACID事务 | ❌ 不支持 |
| 行锁 | ✅ 支持 | ❌ 表锁 |
| 外键 | ✅ 支持 | ❌ 不支持 |
| 崩溃恢复 | ✅ 自动恢复 | ❌ 需手动修复 |
| 全文索引 | ✅（5.6+） | ✅ |
| 存储结构 | 主键聚簇索引 | 非聚簇索引 |
| 计数性能 | COUNT(*) 全表扫描 | 有内部计数器，快 |

### InnoDB 的核心优势

**1. 聚簇索引（Clustered Index）**
- 数据文件按主键顺序存储，叶子节点包含完整数据
- 二级索引叶子节点存储主键值而非行指针，减少回表

**2. MVCC（多版本并发控制）**
- 读不加锁，写不阻塞读，大幅提升并发性能
- 通过 Read View 实现快照读

**3. 行级锁 + 间隙锁**
- 支持更细粒度的锁，减少锁冲突
- 间隙锁（Gap Lock）防止幻读

### 选择建议

- **选 InnoDB：** 生产环境、大并发业务、需要事务和行级锁、数据可靠性要求高
- **选 MyISAM：** 读多写少的小型项目、日志表、只做计数功能的表（已被 InnoDB 替代）

### 迁移注意点

```sql
-- 查看存储引擎
SHOW TABLE STATUS LIKE 'orders';

-- 修改存储引擎
ALTER TABLE orders ENGINE = InnoDB;
```

**来源：** MySQL 8.0 Reference Manual、字节跳动技术博客

---

## 题目 4：PostgreSQL 中如何进行查询调优？有哪些关键手段？

### 参考答案

### 1. 使用 EXPLAIN ANALYZE 分析执行计划

```sql
EXPLAIN (ANALYZE, BUFFERS, FORMAT TEXT) 
SELECT * FROM orders WHERE user_id = 123;
-- 重点看：
-- Seq Scan: 是否全表扫描
-- Index Scan / Index Only Scan: 是否用到索引
-- Bitmap Heap Scan: 大数据量时的索引策略
-- Hash Join / Nested Loop: 关联策略是否合理
```

### 2. 索引优化

```sql
-- 创建合适的索引
CREATE INDEX idx_orders_user_id ON orders(user_id);

-- 表达式索引（解决函数导致索引失效）
CREATE INDEX idx_orders_date ON orders(DATE(created_at));

-- 局部索引（只索引满足条件的行）
CREATE INDEX idx_orders_active ON orders(user_id) WHERE status = 'active';

-- 覆盖索引（避免回表）
CREATE INDEX idx_orders_cover ON orders(user_id) INCLUDE (amount, created_at);
```

### 3. 统计信息与计划器

```sql
-- 更新统计信息（数据变更后必须执行）
ANALYZE orders;

-- 调整计划器成本参数
SET random_page_cost = 1.1;  -- SSD 设为接近 seq_page_cost
```

### 4. 分区表（Partitioning）

```sql
-- 按时间分区，大表优化经典方案
CREATE TABLE orders (
    id BIGSERIAL,
    created_at TIMESTAMP NOT NULL
) PARTITION BY RANGE (created_at);

CREATE TABLE orders_2026 PARTITION OF orders
    FOR VALUES FROM ('2026-01-01') TO ('2027-01-01');
```

### 5. 连接池与缓存

```sql
-- 使用 PgBouncer 做连接池，减少连接开销
-- 配置 shared_buffers = 1/4 物理内存
-- 开启 huge_pages 提高大内存性能
```

### 6. 常见坑

- **JOIN 顺序：** PostgreSQL 自动优化，但复杂查询可手动指定
- ** LIKE '%xxx'：** 无法使用索引，改用全文检索（pg_trgm 或 GIN）
- **函数索引：** 对包含函数的列建立表达式索引

**来源：** PostgreSQL 16 Documentation、《PostgreSQL实战》

---

## 题目 5：数据库缓存策略如何设计？Redis + MySQL 如何配合使用？

### 参考答案

### 经典缓存模式：Cache-Aside

```sql
# 1. 读操作
def get_user(user_id):
    user = redis.get(f"user:{user_id}")
    if user:
        return json.loads(user)
    user = mysql.query("SELECT * FROM users WHERE id = %s", user_id)
    redis.setex(f"user:{user_id}", 3600, json.dumps(user))
    return user

# 2. 写操作（先写DB，再删缓存）
def update_user(user_id, data):
    mysql.execute("UPDATE users SET ... WHERE id = %s", user_id)
    redis.delete(f"user:{user_id}")  # 删除而非更新，避免并发问题
```

### 缓存问题及解决方案

**1. 缓存穿透**
- 布隆过滤器或空值缓存
```python
if bloom_filter.might_exist(key):
    cache.set(key, "NULL", expire=60)
```

**2. 缓存击穿（热点key失效）**
- 互斥锁（setnx）确保只有一人重建缓存
- 永不过期 + 异步更新

**3. 缓存雪崩**
- 过期时间加随机值
- 高可用架构（Redis Cluster）

**4. 数据一致性问题**
- 强一致：先删缓存再写DB（低并发）
- 最终一致：延迟双删 + 消息队列补偿
```sql
# 延迟双删
redis.delete(key)
mysql.execute("UPDATE ...")
sleep(0.5)
redis.delete(key)  # 延迟再删一次
```

### MySQL 查询缓存的问题

> MySQL 8.0 已移除查询缓存（Query Cache），因为它不适合高并发写入场景。

### 分层缓存建议

| 层级 | 工具 | 适用场景 |
|------|------|----------|
| 本地缓存 | Caffeine / Guava | 极热点、不变的数据 |
| 分布式缓存 | Redis / Memcached | 跨进程共享 |
| 数据库缓存 | InnoDB Buffer Pool | 热数据页 |

**核心原则：** 缓存是CAP中的AP，保证最终一致即可。

**来源：** 《Redis设计与实现》、美团技术团队博客

---

> 📌 提示：以上答案仅为参考，实际面试中需根据个人经验灵活展开。
> 
> 🔗 GitHub 仓库：https://github.com/qq286158530/interview-questions
> 
> 📅 本题集整理日期：2026-05-21