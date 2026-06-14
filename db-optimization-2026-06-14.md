# 数据库性能优化面试题

> 本文件由 AI 自动生成，涵盖 MySQL 和 PostgreSQL 核心性能优化知识点。
> 日期：2026-06-14

---

## 题目一：MySQL 慢查询优化的基本步骤是什么？

### 参考答案

MySQL 慢查询优化通常遵循以下步骤：

**第一步：开启慢查询日志，定位慢 SQL**

```bash
# 在 my.cnf 中配置
[mysqld]
slow_query_log = ON
slow_query_log_file = /var/lib/mysql/hostname-slow.log
long_query_time = 3  # 超过3秒记录
```

然后使用 `mysqldumpslow` 工具分析日志：

```bash
mysqldumpslow -s t -t 10 /var/lib/mysql/hostname-slow.log
```

**第二步：用 EXPLAIN 分析执行计划**

```sql
EXPLAIN SELECT * FROM users WHERE email = 'test@example.com';
```

重点关注字段：
- `type`：访问类型，const/eq_ref/ref/range/index/ALL（尽量避免 ALL）
- `key`：实际使用的索引
- `rows`：扫描行数，越少越好
- `Extra`：Filesort、Using temporary 等是需要优化的信号

**第三步：使用 SHOW PROFILE 细粒度分析**

```sql
SET profiling = 1;
SHOW VARIABLES LIKE '%pro%';  -- 查看是否开启
SHOW PROFILES;                 -- 查看最近15条SQL
SHOW PROFILE FOR QUERY 1;      -- 分析单条SQL资源消耗
```

**第四步：针对性优化**

- 单表查询：优先锁定最小返回记录表，字段区分度最高的优先查
- `ORDER BY LIMIT` 优先处理排序表
- 参照建索引原则添加适当索引
- 观察结果，不符合预期则重新分析

---

**来源**：[MySQL性能优化面试题_MySQL优化面试题及答案 - 动力节点](https://www.bjpowernode.com/mysqlmst/xnyh.html)

---

## 题目二：MySQL 索引优化的核心原则有哪些？

### 参考答案

**① 优先使用复合索引而非单列索引组合**

复合索引 (a, b, c) 可以同时支持 `a`、`a+b`、`a+b+c` 的查询，而多个单列索引无法合并使用。

**② 利用覆盖索引避免回表**

覆盖索引（包含查询所需全部列）可直接在索引树返回结果，无需再查主表：

```sql
-- 覆盖索引示例
CREATE INDEX idx_user_name ON users(name);

-- 查询时利用覆盖索引，无需回表
SELECT name, age FROM users WHERE name = '张三';
```

**③ 自增主键避免页分裂**

InnoDB 采用 B+ 树存储，自增主键保证新记录顺序插入叶子节点，避免频繁页分裂和索引重建。

**④ 避免使用 SELECT \***

```sql
-- 低效：大量回表
SELECT * FROM users WHERE id > 100;

-- 高效：覆盖索引直接返回
SELECT name, email FROM users WHERE id > 100;
```

**⑤ 减少子查询，使用外连接替代**

```sql
-- 低效：子查询产生笛卡尔积
SELECT * FROM users WHERE id IN (SELECT user_id FROM orders WHERE total > 100);

-- 高效：外连接
SELECT u.* FROM users u JOIN orders o ON u.id = o.user_id WHERE o.total > 100;
```

**⑥ 优先使用短索引**

非叶子节点存储更多索引列，降低树高，减少空间开销：

```sql
-- 电话号码使用 CHAR(11) 而非 TEXT
CREATE INDEX idx_phone ON users(phone(11));
```

**⑦ 避免在索引列上使用函数或运算**

```sql
-- 低效：索引失效
SELECT * FROM users WHERE YEAR(created_at) = 2024;

-- 高效：索引正常生效
SELECT * FROM users WHERE created_at BETWEEN '2024-01-01' AND '2024-12-31';
```

---

**来源**：[MySQL性能优化面试题_MySQL优化面试题及答案 - 动力节点](https://www.bjpowernode.com/mysqlmst/xnyh.html)

---

## 题目三：MySQL 性能优化可以从哪几个层面入手？请详细说明。

### 参考答案

MySQL 性能优化分为四个层面：

**① 硬件及操作系统层面**

- CPU、内存大小、磁盘读写速度、网络带宽
- 操作系统文件句柄数、网络配置
- 通常由 DBA 或运维工程师完成

**② 架构设计层面**

- **主从集群**：保证高可用，单点故障不影响服务
- **读写分离**：读多写少场景下避免读写冲突
- **分库分表**：分库降低 IO 压力，分表降低单表数据量
- **引入缓存**：Redis/MongoDB 等分布式数据库缓解 MySQL 压力

**③ MySQL 程序配置优化（my.cnf）**

- 最大连接数 `max_connections`
- `binlog` 日志开启
- `innodb_buffer_pool_size` 缓冲池大小
- 注意全局与会话级别的区别，以及是否支持热加载

**④ SQL 优化（最核心）**

慢 SQL 定位 → EXPLAIN 执行计划分析 → SHOW PROFILE 资源消耗分析

常见 SQL 优化规则：
- SQL 查询基于索引扫描
- 避免索引列上使用函数或运算
- `LIKE %` 放在右边：`WHERE name LIKE '张%'`
- 联合索引从左往右命中
- 避免文件排序，使用索引覆盖排序
- 查询有效列，少用 `SELECT *`
- 小结果集驱动大表（`JOIN` 时小表放左边）

---

**来源**：[面试题：谈一谈你对MySQL 性能优化的理解 - 腾讯云](https://cloud.tencent.com/developer/article/2478179)

---

## 题目四：PostgreSQL 的缓存策略和关键配置参数有哪些？

### 参考答案

**核心内存参数**

| 参数 | 说明 | 建议值 |
|------|------|--------|
| `shared_buffers` | 缓存表数据的共享内存区域 | 物理内存的 25%-40% |
| `work_mem` | 每个查询操作（排序/哈希）使用的内存 | 10MB-100MB |
| `maintenance_work_mem` | 维护操作（创建索引/VACUUM）内存 | 1GB+ |
| `effective_cache_size` | 优化器判断可用文件缓存大小 | 物理内存的 50%-75% |

**配置示例**

```ini
shared_buffers = 4GB
work_mem = 64MB
maintenance_work_mem = 1GB
effective_cache_size = 12GB
max_connections = 300
```

**WAL 相关优化**

```ini
wal_buffers = 16MB                    # shared_buffers 的 1/32
checkpoint_completion_target = 0.9     # 平滑 WAL 写入压力
```

**缓存策略的核心思想**

- `shared_buffers`：减少磁盘 IO，数据页缓存于此
- `work_mem`：每个连接独立分配，需根据并发量合理设置
- `effective_cache_size`：帮助优化器选择索引扫描还是全表扫描
- 引入外部缓存（Redis）可进一步提升读性能

**日常维护**

```sql
VACUUM ANALYZE;           -- 清理死亡元组，更新统计信息
REINDEX INDEX idx_name;   -- 重建索引，减少碎片
```

---

**来源**：[PostgreSQL 性能优化全方位指南：深度提升数据库效率 - 腾讯云](https://cloud.tencent.com/developer/article/2476855)

---

## 题目五：百万级数据如何删除？如何设计分库分表策略？

### 参考答案

**百万级数据删除策略**

直接删除百万级数据会产生大量索引维护成本，速度与索引数量成正比。推荐分步操作：

1. **先删除索引**（耗时约 3 分钟）
2. **再删除无用数据**（耗时约 2 分钟）
3. **最后重建索引**（数据少则创建快，约 10 分钟）

```sql
-- 分批删除示例
DELETE FROM logs WHERE created_at < '2023-01-01' LIMIT 1000;
-- 循环执行直到删除完毕
```

**分库分表策略**

水平分表：将大表按规则拆分到多个表中（按 ID 范围、按时间等）。

```sql
-- 按时间分表示例
CREATE TABLE orders_2024 PARTITION OF orders
FOR VALUES FROM ('2024-01-01') TO ('2025-01-01');
```

**架构设计层面的优化思路**

| 策略 | 适用场景 | 效果 |
|------|----------|------|
| 主从集群 | 高可用需求 | 单点故障不影响服务 |
| 读写分离 | 读多写少 | 避免读写冲突 |
| 分库 | IO 压力大 | 分散单服务器压力 |
| 分表 | 单表数据量大 | 提升查询效率 |
| 引入 Redis | 热点数据访问 | 缓解数据库压力 |

**分表原则**
- 单表数据量超过 500 万时考虑分表（但非绝对，与硬件和数据特性相关）
- 选择有区分度的字段作为分片键（如用户 ID、时间）
- 避免跨分片查询或使用路由中间件

---

**来源**：[MySQL数据库优化面试题 - 知乎专栏](https://zhuanlan.zhihu.com/p/346064689) | [MySQL高性能优化规范建议总结 | JavaGuide](https://javaguide.cn/database/mysql/mysql-high-performance-optimization-specification-recommendations.html)

---

> 📌 以上题目涵盖索引优化、查询优化、缓存策略、存储引擎/配置优化、架构设计等数据库性能优化核心知识点。建议结合实际项目经验准备面试。