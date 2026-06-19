# 数据库性能优化面试题

> 📅 2026-06-19 | 每天进步一点点

---

## 题目一：MySQL 索引优化——为何 SQL 明明有索引却没走索引？

### 题目
有一条查询语句 `SELECT * FROM orders WHERE YEAR(create_time)=2024 AND status=1`，`create_time` 和 `status` 上都有单独索引，为什么查询很慢且没有走索引？

### 答案

**原因分析：**

对索引列进行函数运算或表达式计算，会导致索引失效，MySQL 无法使用索引树进行范围扫描。

**具体原因：**

- `YEAR(create_time)=2024` 对 `create_time` 列使用了 YEAR() 函数，导致索引失效
- MySQL 必须对每一行数据执行 YEAR() 计算后才能过滤

**优化方案：**

```sql
-- 方案一：改为范围查询
SELECT * FROM orders 
WHERE create_time >= '2024-01-01' AND create_time < '2025-01-01' AND status=1;

-- 方案二：使用索引覆盖（将函数计算前置到常量）
-- 利用 create_time 的范围索引 + status 索引的组合
```

**总结：** 避免在 WHERE 条件中对索引列做计算、函数、类型转换。  
📎 来源：https://dev.mysql.com/doc/refman/8.0/en/index-brief-explanation.html

---

## 题目二：MySQL 查询优化——分页深度分页问题如何解决？

### 题目
`SELECT * FROM products ORDER BY id LIMIT 1000000, 10` 为何越往后翻页越慢？如何优化？

### 答案

**问题本质：**

LIMIT m, n 语法的执行过程是：MySQL 先定位到第 m 条记录，然后读取后面的 n 条记录。  
当 m 很大时，需要扫描并跳过前面大量的数据行，效率极低。

**优化方案：**

```sql
-- 方案一：游标分页（基于上一页最大 ID）
SELECT * FROM products 
WHERE id > 1000000 ORDER BY id LIMIT 10;

-- 方案二：延迟关联（先定位 ID，再连表查详情）
SELECT p.* FROM products p
INNER JOIN (SELECT id FROM products ORDER BY id LIMIT 1000000, 10) AS t
ON p.id = t.id;

-- 方案三：记录总页数时用 SQL_CALC_FOUND_ROWS（不推荐大数据量）
```

**核心思想：** 用主键或唯一索引的区间代替 OFFSET，避免大偏移量扫描。

**性能对比：**
- 偏移量 1000 万行时，游标分页耗时 < 10ms，OFFSET 分页耗时 > 5s  
📎 来源：https://dev.mysql.com/doc/refman/8.0/en/limit-optimization.html

---

## 题目三：PostgreSQL 缓存策略——如何利用缓存提升查询性能？

### 题目
PostgreSQL 的 shared_buffers、effective_cache_size、work_mem 分别起什么作用？如何根据服务器配置进行调优？

### 答案

**三个核心参数解析：**

| 参数 | 作用 | 推荐值 |
|------|------|--------|
| `shared_buffers` | 数据页缓存，存放热数据页 | 机器内存的 25%（虚拟机可降至 20%） |
| `effective_cache_size` | 优化器估算缓存可用大小 | 机器内存的 75% |
| `work_mem` | 排序/哈希操作的内存上限 | 机器内存 / 最大并发连接数 / 4 |

**调优思路：**

```sql
-- 查看当前配置
SHOW shared_buffers;
SHOW effective_cache_size;

-- 查看缓存命中率
SELECT 
    sum(heap_blks_read) as heap_read,
    sum(heap_blks_hit) as heap_hit,
    round(sum(heap_blks_hit) / (sum(heap_blks_hit) + sum(heap_blks_read)) * 100, 2) as cache_hit_ratio
FROM pg_statio_user_tables;
```

**实战建议：**
- `cache_hit_ratio` 低于 99% 需关注，可能需要增加 shared_buffers
- 复杂查询（多表 JOIN、大量排序）可适当增大 work_mem，但要注意并发下的内存压力
- effective_cache_size 只是估算值，不影响实际内存分配，但影响执行计划选择

📎 来源：https://www.postgresql.org/docs/current/runtime-config-resource.html

---

## 题目四：MySQL 存储引擎——InnoDB 与 MyISAM 的核心区别及选型

### 题目
项目要存储订单数据，应该选择 InnoDB 还是 MyISAM？为什么？

### 答案

**核心区别：**

| 特性 | InnoDB | MyISAM |
|------|--------|--------|
| 事务支持 | ✅ ACID 事务 | ❌ 不支持 |
| 行锁 | ✅ 支持 | ❌ 表锁 |
| 外键 | ✅ 支持 | ❌ 不支持 |
| 崩溃恢复 | ✅ 自动恢复 | ❌ 需手动修复 |
| 全文索引 | ✅ 5.6+ 支持 | ✅ 原生支持 |
| 并发写入 | ✅ 高 | ❌ 低 |

**订单表选型分析：**

**必须选 InnoDB 的理由：**
1. 订单涉及资金交易，需要事务保证（支付→发货→收货 全流程一致性）
2. 高并发写入需要行锁，避免表锁阻塞
3. 崩溃后自动恢复，不丢数据
4. 支持外键约束订单明细表关联

```sql
CREATE TABLE orders (
    id BIGINT PRIMARY KEY AUTO_INCREMENT,
    user_id BIGINT NOT NULL,
    amount DECIMAL(10,2) NOT NULL,
    status TINYINT DEFAULT 0,
    create_time DATETIME DEFAULT CURRENT_TIMESTAMP,
    INDEX idx_user_id (user_id),
    INDEX idx_status_time (status, create_time)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4;
```

**唯一选 MyISAM 的场景：** 只读的全文搜索、日志表（历史数据不修改）  
📎 来源：https://dev.mysql.com/doc/refman/8.0/en/innodb-introduction.html

---

## 题目五：MySQL + Redis 缓存一致性问题——Cache Aside 模式详解

### 题目
使用 Redis 缓存 MySQL 数据时，更新数据时应该先删缓存还是先更新数据库？为什么？如何解决缓存击穿、穿透、雪崩？

### 答案

**Cache Aside（旁路缓存）模式：**

```
读：Cache 命中 → 直接返回
    Cache 未命中 → 查 DB → 写 Cache → 返回

写：更新 DB → 删除 Cache（不是更新 Cache）
```

**为什么是"删除"而不是"更新"？**

更新缓存可能产生数据不一致（并发下旧值覆盖新值），且增加一次无意义的写操作。

**异常处理（延迟双删）：**
```sql
-- 伪代码
redis.del("product:123")  -- 第一步删缓存
db.update("product:123")  -- 第二步更新数据库
sleep(100ms)              -- 第三步等读操作完成
redis.del("product:123")  -- 第四步再删一次
```

**三大问题及解决方案：**

| 问题 | 现象 | 解决方案 |
|------|------|----------|
| 缓存击穿 | 热点 key 过期，瞬间大量请求打到 DB | 互斥锁 / 永不过期 + 异步更新 / 热点数据永不过期 |
| 缓存穿透 | 查询不存在的数据穿透到 DB | 布隆过滤器 / 空值缓存 / 参数校验 |
| 缓存雪崩 | 大量 key 同时过期 | 过期时间加随机值 / 多级缓存 / 熔断限流 |

📎 来源：https://learn.lianglianglee.com/专栏/Redis 核心技术与实战/06 | Redis 缓存满了怎么办？—— 深入 Docker 镜像/10 讲吃透 Redis 源码/05 | 缓存实战）

---

> 🤖 每天进步一点点 | 面试题持续更新中