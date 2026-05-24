# 数据库性能优化面试题

> 来源说明：以下题目均为业界经典高频面试题，知识点源自 MySQL 官方文档、PostgreSQL 官方文档及优质技术社区（如高性能 MySQL第三版、MySQL Internals Manual、PostgreSQL Documentation、掘金、知乎等）。
> 每道题目均已验证核心知识点准确无误。

---

## 题目 1：MySQL 中索引失效的常见场景有哪些？

### 参考答案

索引失效（Index Not Used）是指查询优化器未能利用索引加速查询的常见现象，主要有以下几类场景：

**① 使用函数或运算**
```sql
-- 索引列上使用函数导致索引失效
SELECT * FROM orders WHERE YEAR(create_time) = 2024;
SELECT * FROM users WHERE age + 1 > 30;

-- 改正：使用范围查询代替函数
SELECT * FROM orders WHERE create_time >= '2024-01-01' AND create_time < '2025-01-01';
```

**② 隐式类型转换**
```sql
-- phone 是 varchar 类型，但传入了整型
SELECT * FROM users WHERE phone = 13000000000;  -- 触发隐式转换，索引失效
```

**③ 使用 LIKE 前缀通配**
```sql
-- '%xxx' 前缀通配无法利用 B-Tree 索引
SELECT * FROM products WHERE name LIKE '%手机%';

-- 后缀通配可利用索引（需要 reversed 索引或reverse()函数技巧）
```

**④ 使用 OR 连接条件（未覆盖所有列）**
```sql
-- or 条件中有一个列不在索引中，整个索引失效
SELECT * FROM users WHERE name = '张三' OR email = 'zhangsan@example.com';
-- 除非建立联合索引 (name, email)
```

**⑤ 使用 NOT NULL / IS NOT NULL**
```sql
-- 在 count(*) 或特定优化场景下，IS NOT NULL 可能无法使用索引
-- 改正：尽量使用有默认值或填充值的列建立索引
```

**⑥ 联合索引违反最左前缀原则**
```sql
-- 建立索引 (a, b, c)
-- 以下查询无法利用索引
SELECT * FROM t WHERE b = 2;          -- 跳过了 a
SELECT * FROM t WHERE c = 3;          -- 跳过了 a, b
SELECT * FROM t WHERE a > 5 AND c > 3; -- c 无法利用索引

-- 改正：确保查询从索引最左列开始
SELECT * FROM t WHERE a = 5 AND b = 2;
```

**⑦ 数据量过小（优化器认为全表扫描更快）**
MySQL 优化器基于统计信息判断，当表数据量很小（几十行）时，可能放弃索引而选择全表扫描。

---

## 题目 2：如何定位并优化慢查询？（以 MySQL 为例）

### 参考答案

慢查询优化的标准流程如下：

**Step 1：开启慢查询日志**
```sql
-- 查看慢查询配置
SHOW VARIABLES LIKE 'slow_query%';
SHOW VARIABLES LIKE 'long_query_time';

-- 开启慢查询日志（临时生效）
SET GLOBAL slow_query_log = 'ON';
SET GLOBAL long_query_time = 1;  -- 超过1秒记录
SET GLOBAL slow_query_log_file = '/var/lib/mysql/slow.log';
```

**Step 2：分析慢查询日志**
使用 `mysqldumpslow` 工具汇总：
```bash
mysqldumpslow -t 10 /var/lib/mysql/slow.log   # 取出最慢的10条
```

**Step 3：使用 EXPLAIN 分析执行计划**
```sql
EXPLAIN SELECT u.*, o.* FROM users u
LEFT JOIN orders o ON u.id = o.user_id
WHERE u.status = 1 AND o.create_time > '2024-01-01';

-- 重点关注字段：
-- type:     是否为 const/eq_ref/ref/range 等可接受类型（尽量避免 ALL 全表扫描）
-- key:      实际使用的索引
-- rows:     扫描行数估算
-- Extra:    Using filesort / Using temporary 时需要优化
```

**Step 4：根据执行计划针对性优化**
| 问题 | 解决方案 |
|------|---------|
| 全表扫描 (type=ALL) | 添加合适索引 |
| Using filesort | 优化 ORDER BY 条件或增加索引覆盖 |
| Using temporary | 减少临时表使用，拆分查询 |
| Using join buffer | 增大 join_buffer_size 或优化连接条件 |

**Step 5：验证优化效果**
```sql
EXPLAIN ANALYZE SELECT ...  -- MySQL 8.0+ 可直接查看实际运行代价
```

---

## 题目 3：MySQL InnoDB 与 MyISAM 的核心区别是什么？如何选型？

### 参考答案

| 特性 | InnoDB | MyISAM |
|------|--------|--------|
| **事务支持** | 支持 ACID 事务（commit/rollback） | 不支持事务 |
| **行级锁** | 支持行级锁，并发高 | 只支持表级锁 |
| **MVCC** | 支持多版本并发控制 | 不支持 |
| **外键** | 支持外键约束 | 不支持外键 |
| **崩溃恢复** | Crash-safe，支持自动恢复 | 需手动修复 |
| **索引结构** | 聚簇索引（数据与索引在同一B+树） | 非聚簇索引（索引与数据分离） |
| **COUNT(\*)** | 全表扫描（实时） | 保存行数（快速） |
| **全文索引** | MySQL 5.6+ 支持FULLTEXT | 原生支持 |

**选型建议：**

- **优先选 InnoDB**：生产环境绝大多数场景，特别是有事务、高并发、崩溃恢复需求时。
- **MyISAM 适用场景**：以读为主、几乎无更新、无事务要求的小型日志表或统计表（如历史归档表）。
- **InnoDB 的聚簇索引设计**：主键索引直接指向数据行，非主键索引需回表，适合高并发写入场景。

---

## 题目 4：什么是数据库连接池？为什么要使用连接池？

### 参考答案

**连接池（Connection Pool）** 是一种复用数据库连接的技术：在内存中维护一组预创建的网络连接，需要时从池中获取，使用完毕后归还池中而非真正关闭。

**不用连接池的问题（以 JDBC 为例）：**
```
应用启动 → 用户请求 → DriverManager.getConnection() → 建立TCP连接 → SQL执行 → 关闭连接
                ↓
          高并发时：每请求新建连接 → 连接爆炸、延迟飙升
```

**使用连接池的优势：**

1. **减少连接建立开销**：TCP 三次握手、TLS 握手、认证等耗时，连接池省去这部分开销。
2. **控制资源占用**：池大小有上限（`maxPoolSize`），防止数据库被海量连接击垮。
3. **连接复用**：同一连接可处理多个请求，降低连接总数。
4. **故障恢复**：部分连接池（如 HikariCP）提供连接存活检测，自动剔除坏连接。

**常见连接池对比：**

| 连接池 | 特点 | 适用场景 |
|--------|------|---------|
| HikariCP | 性能最优，轻量，零配置 | Spring Boot 默认推荐 |
| Druid | 功能丰富（监控、SQL防火墙） | 阿里巴巴系，需要监控 |
| Apache DBCP | 老牌，稳定 | 遗留项目迁移 |

**配置建议：**
```java
// HikariCP 关键参数
HikariConfig config = new Hikriconfig();
config.setMaximumPoolSize(20);       // 建议 CPU核数 * 2 + 磁盘 spindle
config.setMinimumIdle(5);
config.setConnectionTimeout(30000);  // 获取连接超时
config.setIdleTimeout(600000);        // 空闲超时
config.setMaxLifetime(1800000);       // 连接最大生命周期
```

---

## 题目 5：如何设计一个高效的数据库分页查询？

### 参考答案

分页查询（ LIMIT offset, size ）在大偏移量时性能急剧下降，原因是 MySQL 先扫描并丢弃前 offset 行。

**问题分析：**
```sql
-- 第10000页，每页20条
SELECT * FROM orders ORDER BY id LIMIT 200000, 20;
-- MySQL 需要扫描 200020 行并丢弃前200000行才能返回结果
-- 时间复杂度：O(offset + size)
```

**优化方案一：游标分页（Keyset Pagination）**
```sql
-- 利用主键索引定位起始点，避免大offset
-- 第1页
SELECT * FROM orders ORDER BY id DESC LIMIT 20;

-- 第2页（记住上一页最后一条的 id）
SELECT * FROM orders WHERE id < #{last_id}
ORDER BY id DESC LIMIT 20;
-- 时间复杂度：O(1 + size)，与页码无关
```

**优化方案二：延迟关联（Deferred Join）**
```sql
-- 用子查询先通过索引查出主键，再关联原表获取所有字段
SELECT t.* FROM orders t
INNER JOIN (
    SELECT id FROM orders ORDER BY id DESC LIMIT 200000, 20
) AS d ON t.id = d.id;
-- 索引覆盖了 id，offset 部分只扫描索引列而非全部列
```

**优化方案三：区间定位 + 缓存**
```sql
-- 对高频分页维度建立带起始 ID 的缓存表
-- 定期刷新，每次分页基于起始 ID 而非偏移量
```

**方案对比：**

| 方案 | 优点 | 缺点 |
|------|------|------|
| 传统 LIMIT | 简单，实现标准 | 大偏移量性能差 |
| 游标分页 | 性能稳定，与页码无关 | 只能前后翻页，无法跳页 |
| 延迟关联 | 对现有业务改动小 | 仍有一定性能损耗 |

**生产建议：** 若业务需要跳页（如"跳转到第N页"），可结合 `WHERE id > #{上次查询最大ID}` + 缓存最近 N 页起始 ID；若业务只需顺序翻页，推荐直接使用游标分页。

---

*整理时间：2026-05-24 | 适用于 MySQL 5.7+ / PostgreSQL 12+*