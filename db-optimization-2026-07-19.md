# 数据库性能优化面试题

> 📅 日期：2026-07-19
> 🏷️ 标签：MySQL | PostgreSQL | 性能优化 | 索引 | 查询优化

---

## 题目一：如何定位并优化慢查询？

### 📌 考察点
索引优化、查询分析、EXPLAIN 使用

### ✅ 参考答案

**1. 开启慢查询日志**

```sql
-- MySQL 临时开启
SET GLOBAL slow_query_log = 'ON';
SET GLOBAL long_query_time = 1;  -- 超过1秒记录
SET GLOBAL slow_query_log_file = '/var/log/mysql/slow.log';

-- PostgreSQL
ALTER SYSTEM SET log_min_duration_statement = 1000;  -- 超过1秒记录
SELECT pg_reload_conf();
```

**2. 使用 EXPLAIN 分析执行计划**

```sql
EXPLAIN SELECT * FROM orders WHERE user_id = 100 AND status = 'paid';

-- 关键字段解读：
-- type: const > eq_ref > ref > range > index > ALL（尽量避免 ALL）
-- key: 实际使用的索引
-- rows: 扫描行数，越少越好
-- Extra: Using filesort / Using temporary（需优化）
```

**3. 常见优化手段**

| 场景 | 优化方案 |
|------|----------|
| 索引失效 | 避免函数/运算、隐式类型转换、`LIKE '%xx'` 前缀通配 |
| 回表过多 | 覆盖索引、联合索引调整列顺序 |
| 文件排序 | 减少排序数据量、增加合适索引 |
| 临时表 | 优化 SQL 结构、拆分为简单查询 |

**4. 线上调优实战**

```sql
-- 查看当前连接和耗时查询
SHOW PROCESSLIST;  -- MySQL
SELECT * FROM pg_stat_activity WHERE state != 'idle';  -- PostgreSQL

-- 强制使用索引（MySQL）
SELECT * FROM orders USE INDEX (idx_user_status) WHERE user_id = 100;
```

**📚 来源**
- MySQL 慢查询优化：[MySQL 官方文档](https://dev.mysql.com/doc/refman/8.0/en/slow-query-log.html)
- PostgreSQL 性能调优：[PostgreSQL 官方文档](https://www.postgresql.org/docs/current/performance-tips.html)

---

## 题目二：InnoDB 与 MyISAM 的核心区别？如何选择？

### 📌 考察点
存储引擎特性、事务支持、锁机制、场景选型

### ✅ 参考答案

| 特性 | InnoDB | MyISAM |
|------|--------|--------|
| 事务支持 | ✅ 支持 ACID | ❌ 不支持 |
| 锁粒度 | 行级锁 + 间隙锁 | 表级锁 |
| 外键 | ✅ 支持 | ❌ 不支持 |
| 全文索引 | ✅（5.6+） | ✅ 原生支持 |
| 崩溃恢复 | ✅ 自动修复 | ❌ 需手动修复 |
| 存储结构 | 表空间 + 共享表空间 | 三个独立文件 |
| 索引 | Clustered Index | Heap Table |

**InnoDB 为什么是默认存储引擎？**

1. **事务安全**：支持 COMMIT、ROLLBACK，保证数据一致性
2. **并发友好**：行级锁减少锁冲突，高并发下性能更优
3. **外键约束**：维护表间数据完整性
4. **崩溃恢复**：redo log + undo log 自动恢复

**选型建议**

```
选择 InnoDB ✅：
- 需要事务（订单、支付、用户数据）
- 高并发写入/更新
- 数据安全要求高
- 主从架构需要 MVCC

选择 MyISAM ⚠️：
- 只读/很少更新的数据（配置表、日志表）
- 全文搜索场景（5.7+ 也可用 InnoDB FTS）
- 极端写入性能优先且不需要事务
```

**📚 来源**
- InnoDB 架构：[MySQL Internals Manual](https://dev.mysql.com/doc/internals/en/innodb.html)
- 存储引擎对比：[MySQL 官方文档](https://dev.mysql.com/doc/refman/8.0/en/storage-engines.html)

---

## 题目三：什么是索引下推（Index Condition Pushdown）？如何利用它优化查询？

### 📌 考察点
索引原理、ICP 优化机制、联合索引设计

### ✅ 参考答案

**ICP 原理**

索引下推是 MySQL 5.6+ 引入的优化策略，**在索引遍历过程中，在引擎层直接过滤索引列条件，减少回表次数**。

**无 ICP vs 有 ICP 对比**

```sql
-- 表结构
CREATE TABLE users (
  id INT PRIMARY KEY,
  name VARCHAR(20),
  age INT,
  city VARCHAR(20),
  INDEX idx_name_age_city (name, age, city)
);

-- 查询条件
SELECT * FROM users WHERE name = '张三' AND age > 25;
```

```
❌ 无 ICP（MySQL 5.6之前）：
1. 使用索引找到 name='张三' 的记录 → 获得主键
2. 回表获取完整行数据
3. 在 Server 层过滤 age > 25

✅ 有 ICP（MySQL 5.6+）：
1. 使用索引找到 name='张三' 的记录
2. 在引擎层直接过滤 age > 25（利用索引中 age 列）
3. 只将满足条件的记录回表
```

**ICP 开启状态**

```sql
-- 查看优化器开关
SHOW VARIABLES LIKE 'optimizer_switch';
-- index_condition_pushup=on 表示开启

-- 关闭 ICP（仅测试用）
SET optimizer_switch = 'index_condition_pushoff=on';
```

**ICP 生效条件**

1. 必须是索引列条件（列在索引中且无法使用索引范围扫描）
2. 需要访问表的所有列都在索引中才能下推（覆盖索引效果最佳）
3. 条件不能是常量（常量可在索引树快速比较）

**联合索引设计原则**

```
查询：WHERE a = ? AND b > ? AND c = ?
最优索引：(a, b, c) 或 (a, c, b)

原因：
- 等值条件放前面（a, c）
- 范围条件放最后（b > ?），范围后的列无法利用索引
- 避免回表：将 WHERE 和 SELECT 中的列都纳入索引
```

**📚 来源**
- ICP 官方文档：[MySQL 5.6 Release Notes](https://dev.mysql.com/doc/refman/5.6/en/index-condition-pushdown-optimization.html)
- 联合索引设计：[MySQL 索引原理详解](https://dev.mysql.com/doc/refman/8.0/en/multiple-column-indexes.html)

---

## 题目四：如何设计分库分表方案？说说读写分离的典型架构。

### 📌 考察点
数据库架构设计、分布式概念、数据迁移

### ✅ 参考答案

**分库分表适用场景**

```
单表数据量超过 1000 万行
单库 QPS 超过 1 万
业务增长快，水平扩展需求强烈
```

**垂直拆分 vs 水平拆分**

| 类型 | 拆分维度 | 适用场景 |
|------|----------|----------|
| 垂直分库 | 按业务模块 | 微服务架构、不同模块访问量差异大 |
| 垂直分表 | 按冷热字段 | 大文本字段、配置信息独立管理 |
| 水平分库 | 按数据行分到不同库 | 单库性能瓶颈、数据量大 |
| 水平分表 | 按数据行分到同库不同表 | 单表数据量爆炸 |

**水平分片策略**

```sql
-- 取模分片（数据均匀，但扩容困难）
shard_key = user_id % 4

-- 范围分片（扩展方便，但可能热点不均）
shard_key = user_id / 1000  -- 按ID区间

-- 哈希分片（一致性哈希，扩容迁移少）
shard_key = hash(user_id) % 4
```

**读写分离架构**

```
                 ┌─────────────┐
                 │   应用层    │
                 └──────┬──────┘
                        │
        ┌───────────────┼───────────────┐
        ▼               ▼               ▼
   ┌─────────┐    ┌─────────┐    ┌─────────┐
   │ 主库    │    │ 从库1   │    │ 从库2   │
   │ (写)    │◄───│ (读)    │◄───│ (读)    │
   └─────────┘    └─────────┘    └─────────┘
```

**MySQL 主从复制配置**

```sql
-- 主库配置（my.cnf）
server-id = 1
log-bin = mysql-bin
binlog-do-db = myapp

-- 从库配置
server-id = 2
relay-log = relay-bin
read-only = 1  -- 只读（需注意 super_read_only）

-- 从库执行
CHANGE MASTER TO
  MASTER_HOST = '主库IP',
  MASTER_USER = 'repl',
  MASTER_PASSWORD = '密码',
  MASTER_LOG_FILE = 'mysql-bin.001',
  MASTER_LOG_POS = 4;

START SLAVE;
SHOW SLAVE STATUS\G;
```

**常见问题及解决**

| 问题 | 解决方案 |
|------|----------|
| 主从延迟 | 加缓存、读从库失败时读主库 |
| 数据一致性 | 半同步复制、增强半同步 |
| 扩容迁移 | 使用一致性哈希环，最小化数据迁移 |
| 跨分片查询 | 多次查询 + 内存聚合，或引入 ES |

**📚 来源**
- MySQL 主从复制：[MySQL 官方文档 - Replication](https://dev.mysql.com/doc/refman/8.0/en/replication.html)
- 分库分表中间件：[ShardingSphere 官方文档](https://shardingsphere.apache.org/)

---

## 题目五：Redis 与 MySQL 如何配合使用？缓存一致性问题如何解决？

### 📌 考察点
缓存策略、分布式系统一致性、CAP 理论

### ✅ 参考答案

**缓存读写模式**

```
模式一：Cache-Aside（旁路缓存）⭐最常用
┌────────────────────────────────────────────┐
│ 读：先读缓存 → 缓存未命中 → 读DB → 写缓存  │
│ 写：先写DB → 删除缓存（而非更新）          │
└────────────────────────────────────────────┘

模式二：Read-Through（读穿透）
应用只读缓存，缓存负责加载数据库

模式三：Write-Through（写穿透）
写缓存时同步写数据库
```

**为什么写操作是删除缓存而非更新？**

```
场景：同时有线程A（写）和线程B（读）

错误流程（更新缓存）：
1. 线程A更新数据库 age=26
2. 线程B读取，发现缓存旧值 age=25
3. 线程A更新缓存 age=26
4. 线程B拿到 age=26（看似没问题，但时序错乱会导致数据不一致）

正确流程（删除缓存）：
1. 线程A删除缓存
2. 线程B读取，缓存未命中，读DB age=26，写入缓存
3. 后续读取都是 age=26
```

**缓存一致性方案**

```sql
-- 方案1：延迟双删（解决主从延迟问题）
1. 先删除缓存
2. 写数据库
3. 延迟 N 毫秒（如500ms）再删除缓存

-- 方案2：订阅 binlog + 异步更新缓存
使用 Canal（MySQL）或其他工具订阅数据库变更
变更后写入 Redis，保证最终一致

-- 方案3：分布式锁
SET lock_key product_1 NX PX 3000
-- 获取锁后写DB，再更新缓存，最后释放锁
```

**缓存问题应对**

| 问题 | 表现 | 解决方案 |
|------|------|----------|
| 缓存穿透 | 查询不存在数据，每次都打DB | 布隆过滤器 / 缓存空值 |
| 缓存击穿 | 热点key过期，瞬间打爆DB | 互斥锁 / 永不过期+异步更新 |
| 缓存雪崩 | 大量key同时过期 | 过期时间 + 随机值 / 多级缓存 |

**Redis 集群高可用**

```
Redis Sentinel（哨兵模式）：
- 监控主从切换
- 自动故障转移
- 客户端自动感知新主库

Redis Cluster：
- 数据分片（16384个槽）
- 去中心化
- 支持高并发写入
```

**📚 来源**
- Redis 缓存策略：[Redis 官方文档](https://redis.io/docs/management/optimization/)
- 数据库缓存一致性：[缓存架构设计指南](https://docs.microsoft.com/en-us/azure/architecture/best-practices/caching)

---

## 📖 更多学习资源

- [MySQL 8.0 Reference Manual](https://dev.mysql.com/doc/refman/8.0/en/)
- [PostgreSQL 16 Documentation](https://www.postgresql.org/docs/16/)
- [Redis 官方文档](https://redis.io/documentation)
- [ShardingSphere 官方文档](https://shardingsphere.apache.org/)

---

> 💡 提示：面试中不仅要看答案，更要理解原理，能够举一反三！
