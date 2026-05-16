# 数据库性能优化面试题 · 2026-05-16

本文档整理了 5 道高质量数据库性能优化面试题，涵盖索引优化、查询优化、存储引擎、缓存策略、主从复制等核心知识点。

---

## 一、MySQL 为什么默认采用 InnoDB 存储引擎？与 MyISAM 的核心区别是什么？

### 答案

**InnoDB 是 MySQL 5.5+ 的默认存储引擎**，取代了之前的 MyISAM，主要区别如下：

### 核心区别对比

| 特性 | InnoDB | MyISAM |
|-----|--------|--------|
| **事务支持** | ✅ ACID 事务 | ❌ 不支持 |
| **行锁** | ✅ 支持行级锁 | ❌ 只有表级锁 |
| **外键** | ✅ 支持 | ❌ 不支持 |
| **MVCC** | ✅ 支持 | ❌ 不支持 |
| **崩溃恢复** | ✅ 自动恢复 | ❌ 需手动修复 |
| **全文索引** | ✅ 5.6+ 支持 | ✅ 原生支持 |
| **存储空间** | 约 2 倍 MyISAM | 相对较小 |
| **适用场景** | 高并发、事务场景 | 读多写少、日志 |

### InnoDB 关键技术点

**1. MVCC（多版本并发控制）**
- 每行数据隐藏两个版本列：`DB_TRX_ID`（事务ID）和 `DB_ROLL_PTR`（回滚指针）
- 读操作不加锁，通过版本链实现快照读
- 配合 `Read Committed` 和 `Repeatable Read` 隔离级别

**2. 行级锁与意向锁**
- 加锁时先加意向锁（表级），再在记录上加行锁
- 避免全表扫描检查锁

**3. 两次写（Doublewrite Buffer）**
- 页面写入磁盘前，先写入 doublewrite buffer，再写入表空间
- 防止 partial page write（页写入只完成一半的崩溃）

**4. 自适应哈希索引**
- InnoDB 自动监控表访问，为热点数据建立哈希索引
- `SHOW ENGINE INNODB STATUS` 可查看自适应哈希索引状态

### 面试加分点

> 为什么 InnoDB 能支持高并发？
> - 普通的 SELECT 不加锁（MVCC）
> - 行级锁减少锁冲突
> - 插入缓冲（Insert Buffer）优化随机插入
> - 异步 I/O + 刷新邻接页（Flush Neighbor Thread）

来源参考： [InnoDB 存储引擎官方文档](https://dev.mysql.com/doc/refman/8.0/en/innodb-storage-engine.html) | [InnoDB vs MyISAM 对比](https://www.mysql.com/innodb/)

---

## 二、如何通过 EXPLAIN 分析 SQL 查询性能？有哪些关键字段必须关注？

### 答案

`EXPLAIN` 是 MySQL 查询优化的神器，通过它可以判断索引使用情况、扫描行数、排序方式等。

### 基础用法

```sql
EXPLAIN SELECT * FROM users WHERE name = '张三' AND status = 1;
```

### 必须关注的 6 个关键字段

#### 1. type（连接类型）— 最重要

| type 值 | 含义 | 推荐程度 |
|---------|------|---------|
| `const` | 主键/唯一索引等值查询，最多 1 行 | ⭐⭐⭐ 最优 |
| `eq_ref` | 被驱动表JOIN时，使用主键/唯一索引 | ⭐⭐⭐ |
| `ref` | 非唯一索引等值查询 | ⭐⭐ |
| `range` | 索引范围扫描 | ⭐ |
| `index` | 全索引扫描 | ❌ |
| `ALL` | 全表扫描 | ❌ 最差 |

> 🚨 type = ALL 是性能杀手，必须优化！

#### 2. key（实际使用的索引）

```sql
key: idx_user_name  -- 使用了哪个索引
key_len: 98         -- 索引字段长度，越小越好
```

#### 3. rows（预计扫描行数）

```sql
rows: 1000  -- 预计扫描 1000 行，越少越好
```

#### 4. Extra（附加信息）— 重要！

| Extra 值 | 含义 | 优化建议 |
|---------|------|---------|
| `Using filesort` | 文件排序，需额外文件 | ❌ 用索引排序替代 |
| `Using temporary` | 临时表 | ❌ 需优化查询 |
| `Using index` | 索引覆盖，不用回表 | ⭐ 最优 |
| `Using index condition` | 索引下推 | ⭐ 优化 |
| `Using where` | 服务端过滤 | ⚠️ 需结合 rows 评估 |
| `Impossible WHERE` | WHERE 条件永假 | ❌ 检查 SQL |

#### 5. possible_keys（可用索引）

显示查询可能使用的索引，不一定真正使用。

#### 6. key_len（索引长度）

判断复合索引使用了多少列，越小越好。

### 实战案例

```sql
EXPLAIN SELECT id, name FROM users
WHERE status = 1 AND created_at > '2024-01-01'
ORDER BY created_at DESC;
```

输出分析：
```
type: range        → 使用了范围查询（可接受）
key: idx_status_created  → 使用了复合索引
rows: 50           → 只扫描 50 行（优秀）
Extra: Using index condition; Using filesort → 有排序需优化
```

### 优化建议

- **Using filesort 优化：** 添加 `created_at` 索引，利用索引本身有序避免文件排序
- **Using temporary 优化：** 避免使用 DISTINCT、GROUP BY（无索引）、ORDER BY（多字段）等

来源参考： [MySQL EXPLAIN 官方文档](https://dev.mysql.com/doc/refman/8.0/en/explain-output.html) | [EXPLAIN 详解](https://dev.mysql.com/blog-category/explain/)

---

## 三、MySQL 慢查询如何优化？有哪些具体的优化思路和手段？

### 答案

慢查询优化是数据库面试的重点，一般按照以下思路进行：

### 优化思路总览

```
慢查询优化五步法：
1. 定位慢查询（慢日志/监控）
2. EXPLAIN 分析执行计划
3. 检查索引（是否失效/缺失）
4. 分析业务逻辑
5. 改写 SQL 或调整架构
```

### Step 1：定位慢查询

```bash
# 开启慢查询日志
slow_query_log = 1
slow_query_log_file = /var/lib/mysql/slow.log
long_query_time = 1  # 超过 1 秒记录

# 查看慢查询
mysqldumpslow -t 5 /var/lib/mysql/slow.log   # Top 5 慢查询
```

```sql
-- 查看最近慢查询
SELECT * FROM mysql.slow_log ORDER BY start_time DESC LIMIT 10;
```

### Step 2：EXPLAIN 分析执行计划（见第二题）

### Step 3：索引优化

**1. 复合索引最左前缀原则**

```sql
-- 创建复合索引
CREATE INDEX idx_a_b_c ON orders(a, b, c);

-- 以下查询可以命中索引：
WHERE a = 1            -- ✅ 只用 a
WHERE a = 1 AND b = 2  -- ✅ 用 a + b
WHERE a = 1 AND b = 2 AND c = 3  -- ✅ 全部使用

-- 以下查询不能命中索引：
WHERE b = 2            -- ❌ 跳过 a
WHERE c = 3            -- ❌ 跳过 a 和 b
```

**2. 索引失效的常见场景**

```sql
-- ❌ 这些情况会导致索引失效
WHERE name = '张三' AND LEFT(name, 2) = '张'  -- 函数在索引列上
WHERE age + 1 = 10                               -- 运算在索引列上
WHERE status = 1 OR name = '李四'                -- OR 导致索引中断（可改 UNION）
WHERE name LIKE '%四%'                          -- %开头无法使用索引
WHERE id = '123'                                 -- 类型转换（整型字段用字符串）
```

**3. 覆盖索引避免回表**

```sql
-- 索引覆盖示例
CREATE INDEX idx_id_name ON users(id, name);

-- 这条查询可以直接从索引中获取数据，无需回表
SELECT id, name FROM users WHERE id = 1;  -- Using index
```

### Step 4：SQL 改写优化

**1. 大批量插入优化**

```sql
-- ❌ 低效：逐条插入
INSERT INTO orders VALUES(1, 'A');
INSERT INTO orders VALUES(2, 'B');

-- ✅ 高效：批量插入
INSERT INTO orders VALUES(1, 'A'), (2, 'B'), (3, 'C');

-- ✅ 更大批量：LOAD DATA
LOAD DATA INFILE '/tmp/orders.csv' INTO TABLE orders;
```

**2. 分页查询优化**

```sql
-- ❌ 深度分页（off过大时效率低）
SELECT * FROM orders ORDER BY id LIMIT 100000, 20;

-- ✅ 优化：基于主键游标
SELECT * FROM orders WHERE id > 100000 ORDER BY id LIMIT 20;

-- ✅ 优化：子查询先定位主键
SELECT * FROM orders o
INNER JOIN (
    SELECT id FROM orders ORDER BY id LIMIT 100000, 20
) t ON o.id = t.id;
```

**3. JOIN 优化**

```sql
-- ✅ 小表驱动大表（EXPLAIN 中前面是小表）
SELECT * FROM users u INNER JOIN orders o ON u.id = o.user_id;

-- ✅ 添加合适索引
CREATE INDEX idx_orders_user_id ON orders(user_id);
```

### Step 5：架构层优化

| 手段 | 说明 |
|-----|------|
| **读写分离** | 主从复制，将读请求分散到从库 |
| **分库分表** | 数据量大时水平拆分（Sharding） |
| **缓存前置** | Redis 缓存热点数据，减少 DB 压力 |
| **冷热分离** | 将历史数据归档到归档库 |

来源参考： [MySQL 慢查询优化官方指南](https://dev.mysql.com/doc/refman/8.0/en/slow-query-log.html) | [MySQL 性能优化](https://www.mysql.com/performance/)

---

## 四、什么是数据库主从复制？主从复制有哪些模式？如何解决主从延迟问题？

### 答案

主从复制（Master-Slave Replication）是数据库高可用和读写分离的基础架构。

### 主从复制原理

```
┌─────────────┐                    ┌─────────────┐
│   Master    │ ←── Binlog ────→  │   Slave     │
│  (主库)     │    事件传输        │  (从库)     │
└─────────────┘                    └─────────────┘
     │                                    │
  W/R                                 R Only
```

**复制流程：**

1. 主库将所有写操作记录到 Binlog（二进制日志）
2. 从库通过 IO Thread 连接主库，请求 Binlog 中的新事件
3. 主库通过 Dump Thread 将 Binlog 内容发送给从库
4. 从库的 IO Thread 将事件写入 Relay Log（中继日志）
5. 从库的 SQL Thread 读取 Relay Log，执行其中的事件

### 主从复制模式

**模式一：异步复制（Async Replication）**
- 主库提交事务后，不等从库确认就返回
- ⚠️ 主库故障时可能丢失数据

**模式二：半同步复制（Semi-sync Replication）MySQL 5.5+**
- 主库等至少一个从库确认收到 Binlog 后才提交
- ✅ 性能略低，但数据更安全

```sql
-- 启用半同步复制（需安装插件）
INSTALL PLUGIN rpl_semi_sync_master SONAME 'semisync_master.so';
SET GLOBAL rpl_semi_sync_master_enabled = 1;
```

**模式三：全同步复制（Full Sync）**
- 所有从库确认后才提交，性能最差，实际很少用

### GTID 复制模式（MySQL 5.6+）

**GTID（Global Transaction ID）** 是全局事务标识符，解决传统复制位点管理的难题：

```sql
-- 开启 GTID 模式
gtid_mode = ON
enforce_gtid_consistency = ON

-- 自动定位复制位点，无需知道 binlog 文件和位置
CHANGE MASTER TO MASTER_AUTO_POSITION = 1;
```

### 主从延迟原因及解决方案

**常见延迟原因：**

| 原因 | 表现 | 解决方案 |
|-----|------|---------|
| 从库大事务 | 从库执行时间超过主库 | 拆分大事务为小批次 |
| 从库服务器性能差 | CPU/IO 瓶颈 | 升级从库硬件 |
| 网络延迟 | 跨机房复制 | 优化网络或同机房部署 |
| 从库查询阻塞 | 从库承担大量读请求 | 读写分离 + 从库扩展 |
| 复制延迟 | 大表 DDL 操作 | 使用 pt-online-schema-change |

**监控延迟：**

```sql
-- 查看从库延迟
SHOW SLAVE STATUS\G
Seconds_Behind_Master: 0  -- 延迟秒数，为 0 表示无延迟

-- 或监控 GTID 延迟
SHOW MASTER STATUS;
SHOW SLAVE STATUS;
```

**解决延迟的实践方案：**

1. **读写分离**：读操作分散到多个从库
2. **延迟扩容**：多个从库分担读压力
3. **中间件**：使用 ProxySQL 或 MySQL Router 做负载均衡
4. **应用层补偿**：写入后短暂延迟再读，或者强制读主库（关键数据）

来源参考： [MySQL 主从复制官方文档](https://dev.mysql.com/doc/refman/8.0/en/replication.html) | [MySQL GTID 复制](https://dev.mysql.com/doc/refman/8.0/en/replication-gtids.html)

---

## 五、PostgreSQL 与 MySQL 相比，在性能优化上有哪些独特优势？何时选择 PostgreSQL？

### 答案

PostgreSQL 和 MySQL 是最流行的两款开源关系型数据库，两者定位不同，性能优化思路也有差异。

### 核心区别与 PostgreSQL 优势

#### 1. MVCC 与并发控制

**MySQL InnoDB 的 MVCC：**
- 每次只保留两个版本（当前版本 + 历史版本）
- 通过 `undo log` 管理旧版本

**PostgreSQL 的 MVCC（版本链更长）：**
- 每个事务可见性判断通过 `xmin/xmax` 字段
- 旧版本存储在表文件中，VACUUM 清理
- 优势：事务隔离更严格，无"脏读"隐患

#### 2. 多版本并发控制（PostgreSQL 优势）

```sql
-- PostgreSQL 的隔离级别默认是 READ COMMITTED
-- 不同事务看到的数据版本不同

BEGIN;
SELECT * FROM orders WHERE id = 1;  -- 事务 A 看到 version=1
-- （另一个事务修改了数据）
SELECT * FROM orders WHERE id = 1;  -- 事务 A 可能看到不同数据（脏读风险低）
COMMIT;
```

#### 3. PostgreSQL 的查询优化器更智能

PostgreSQL 使用 **Cascades 优化器**，比 MySQL 的优化器更复杂：

```sql
-- PostgreSQL 会自动重写子查询为 JOIN
EXPLAIN SELECT * FROM users
WHERE user_id IN (SELECT id FROM orders WHERE amount > 1000);

-- MySQL 5.7 可能走嵌套循环（效率低）
-- PostgreSQL 自动优化为 HASH JOIN 或 MERGE JOIN
```

#### 4. 索引类型更丰富

| 索引类型 | PostgreSQL | MySQL |
|---------|-----------|-------|
| B-Tree | ✅ | ✅ |
| Hash | ✅ | ✅（Memory引擎） |
| GiST | ✅ | ❌ |
| GIN | ✅ | ❌ |
| BRIN | ✅ | ❌ |
| 表达式索引 | ✅ | ❌（5.7前） |
| 部分索引 | ✅ | ❌（5.7前） |

**PostgreSQL 特有索引场景：**

```sql
-- GIN 索引：适合数组、全文搜索
CREATE INDEX idx_tags ON posts USING GIN(tags);  -- 标签数组检索

-- BRIN 索引：适合有序物理存储的大表（时序数据）
CREATE INDEX idx_created_at ON logs USING BRIN(created_at);

-- 部分索引：只索引满足条件的行
CREATE INDEX idx_active_users ON users(created_at)
WHERE status = 'active';

-- 表达式索引：索引计算结果
CREATE INDEX idx_email_lower ON users(LOWER(email));
```

#### 5. 全表锁与 DDL 操作

**MySQL：** ALTER TABLE 会锁表（MySQL 5.6+ 支持 Online DDL）

**PostgreSQL：** 支持 `VACUUM` 和并发建索引：

```sql
-- PostgreSQL 并发创建索引（不影响写入）
CREATE INDEX CONCURRENTLY idx_name ON table_name(column);

-- 配合 pg_repack 工具在线修改表结构
```

#### 6. 性能优化工具对比

| 工具 | PostgreSQL | MySQL |
|-----|-----------|-------|
| 慢查询分析 | `pg_stat_statements` | `slow_query_log` |
| 实时监控 | `pg_stat_activity` | `SHOW PROCESSLIST` |
| 执行计划 | `EXPLAIN ANALYZE` | `EXPLAIN ANALYZE` |
| 连接池 | PgBouncer | ProxySQL |
| 分区表 | 原生分区表 | 分区表（MySQL 5.7+） |

**PostgreSQL 常用性能分析：**

```sql
-- 开启性能统计
CREATE EXTENSION pg_stat_statements;

-- 查看最慢查询
SELECT query, calls, mean_time, total_time
FROM pg_stat_statements
ORDER BY mean_time DESC
LIMIT 10;

-- 查看实时活动
SELECT * FROM pg_stat_activity WHERE state != 'idle';

-- 分析表膨胀
SELECT relname, n_dead_tup, n_live_tup, last_vacuum
FROM pg_stat_user_tables;
```

### 何时选择 PostgreSQL？

| 场景 | 推荐 |
|-----|------|
| 复杂查询、OLAP 分析 | ✅ PostgreSQL |
| JSON/FJSON 数据处理 | ✅ PostgreSQL（JSONB 类型） |
| GIS 地理信息系统 | ✅ PostGIS 扩展 |
| 全文搜索 | ✅ PostgreSQL GIN 索引 |
| 高并发 OLTP | ⚠️ MySQL（InnoDB 更好） |
| 超大数据量（>TB 级） | ✅ PostgreSQL 分区表 + BRIN |

### 一句话总结

> **MySQL 适合互联网高并发小事务场景；PostgreSQL 适合复杂分析、多样化数据类型和需要高级 SQL 特性的场景。**

来源参考： [PostgreSQL 官方文档](https://www.postgresql.org/docs/) | [PostgreSQL vs MySQL 对比](https://www.postgresql.org/docs/current/mvcc-intro.html) | [PostgreSQL 性能优化](https://www.postgresql.org/docs/current/using-explain.html)

---

## 📚 参考来源

- [MySQL 官方文档](https://dev.mysql.com/doc/refman/8.0/en/)
- [PostgreSQL 官方文档](https://www.postgresql.org/docs/)
- [MySQL 慢查询优化官方指南](https://dev.mysql.com/doc/refman/8.0/en/slow-query-log.html)
- [PostgreSQL pg_stat_statements](https://www.postgresql.org/docs/current/pgstatstatements.html)
- [InnoDB 存储引擎官方文档](https://dev.mysql.com/doc/refman/8.0/en/innodb-storage-engine.html)
- [MySQL 主从复制官方文档](https://dev.mysql.com/doc/refman/8.0/en/replication.html)

整理 by AI 助手 | 2026-05-16