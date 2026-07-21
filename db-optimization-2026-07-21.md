# 数据库性能优化面试题（2026-07-21）

> 本文件收录 5 道高质量 MySQL/PostgreSQL 数据库性能优化面试题，涵盖查询优化器、分库分表、事务隔离、索引高级特性、连接池等核心知识点。

---

## 题目一：MySQL 与 PostgreSQL 的查询优化器有何不同？如何利用 Cost Model 优化 SQL？

### 参考答案

**MySQL 优化器特点**

MySQL 使用基于规则的优化器（RBO，Rule-Based Optimizer）与代价优化器（CBO，Cost-Based Optimizer）结合，但优化能力相对简单：
- 仅支持**启发式规则**（如 index merge、range scan）
- 统计信息不够精确（InnoDB 采样 pages limited）
- 不支持函数索引、表达式索引（需生成虚拟列）
- 联接优化只支持 NestLoop、HashJoin、SortMergeJoin 三种

**PostgreSQL 优化器特点**

PostgreSQL 是纯 CBO 优化器，能力强大得多：
- 支持**表达式索引**和**函数索引**
- 支持**部分索引**（Partial Index）和 **BRIN 索引**
- 支持更丰富的统计信息（多列统计、函数依赖）
- 支持 **GEQO（Genetic Query Optimizer）** 优化多表联接顺序

**Cost Model 解析（以 PostgreSQL 为例）**

```sql
-- 查看查询计划的代价
EXPLAIN (ANALYZE, COSTS, BUFFERS) 
SELECT * FROM orders WHERE status = 1;

-- 关键 Cost 参数（postgresql.conf）
-- seq_page_cost = 1.0          -- 顺序扫描一个页的代价
-- random_page_cost = 4.0       -- 随机扫描一个页的代价（SSD 可调低）
-- cpu_tuple_cost = 0.01        -- 处理每行元组的代价
-- cpu_index_tuple_cost = 0.005 -- 索引扫描处理每行的代价

-- 优化建议：将 random_page_cost 设为接近 seq_page_cost（SSD场景）
ALTER SYSTEM SET random_page_cost = 1.1;
SELECT pg_reload_conf();
```

**实战优化技巧**

```sql
-- 1. 利用覆盖索引避免回表（减少 random_page_cost）
CREATE INDEX idx_orders_status ON orders(status) INCLUDE (user_id, amount);
-- 查询直接通过索引返回，无需回表

-- 2. 利用部分索引减少索引体积
CREATE INDEX idx_orders_active ON orders(user_id) WHERE status = 'active';

-- 3. 多列统计信息帮助优化器准确估算
CREATE STATISTICS s1 (dependencies, ndistinct) ON region, status FROM orders;
ANALYZE orders;
```

---

## 题目二：分库分表后带来的问题有哪些？如何解决跨分片查询和分布式 ID 生成？

### 参考答案

**分库分表的核心价值**

将数据水平切分到多个库/表中，解决单表数据量过大的问题（一般单表 > 2000 万行或 > 20GB 需考虑）。

**带来的新问题**

| 问题 | 描述 |
|------|------|
| **跨分片查询** | 数据分散在多个节点，联接、聚合、排序无法在单节点完成 |
| **分布式 ID** | 自增主键无法跨分片保持唯一性 |
| **分布式事务** | 跨库操作需要分布式事务支持 |
| **数据迁移成本** | 扩容（resharding）代价极高 |
| **多泳道管理** | 按业务或租户的路由复杂度增加 |

**跨分片查询的解决方案**

```sql
-- 方案一：异构索引表（Search DS 常用）
-- 将需要查询的字段反向写入 ES / Solr，ES 提供检索，MySQL 提供存储
-- 架构：应用层 → ES（检索）→ 拿到 IDs → MySQL（回表）

-- 方案二：汇总表（离线计算）
-- 定时将各分片数据聚合到汇总表，供报表查询

-- 方案三：分片代理层（ShardingSphere / MyCat）
-- 对应用透明，自动做跨分片 SQL 解析和结果聚合
-- 缺点：复杂联接和聚合性能差

-- 方案四：Scatter-Gather（并发多分片查询）
-- 应用并发查询所有分片，在应用层合并结果
-- 适合：COUNT/SUM/MAX/MIN 等可合并操作
```

**分布式 ID 生成方案**

```java
// 方案一：UUID（不推荐，無序、占用空间大）
String id = UUID.randomUUID().toString();

// 方案二：雪花算法（Snowflake，推荐）
// 64位：1位符号 + 41位时间戳 + 10位机器ID + 12位序列号
// 优点：趋势递增、支持分布式
// 缺点：依赖时钟，若时钟回拨会重复
public class SnowflakeIdWorker {
    private final long twepoch = 1609459200000L;
    private final long workerIdBits = 10L;
    private final long sequenceBits = 12L;
    // ...

    public synchronized long nextId() {
        long timestamp = timeGen();
        if (timestamp < lastTimestamp) {
            timestamp = tilNextMillis(lastTimestamp);
        }
        // ... 序列号溢出处理
        return ((timestamp - twepoch) << 22) | (workerId << 12) | sequence;
    }
}

// 方案三：数据库号段模式（推荐用于高并发）
// 一次从数据库取一批 ID（如 1000 个），本地分配，批量更新
// 优点：不依赖时钟，可超高性能，ID 连续递增
```

---

## 题目三：MySQL InnoDB 的 MVCC 机制是什么？READ COMMITTED 和 REPEATABLE READ 下快照有何不同？

### 参考答案

**MVCC（Multi-Version Concurrency Control）多版本并发控制**

InnoDB 为每个行记录存储两个隐藏列：
- `DB_TRX_ID`：最近一次修改该行的事务 ID
- `DB_ROLL_PTR`：指向 undo log 链表的指针（用于构建该行的历史版本）

**读已提交（READ COMMITTED）下 MVCC 工作方式**

每次读取都重新生成一致性快照：

```
Session A（事务A）：
START TRANSACTION;
-- 事务A获取快照时间点 T1

Session B（事务B）：
INSERT INTO orders VALUES(1001, '商品A');  -- 提交
COMMIT;

Session A：
SELECT * FROM orders;  -- 可看到订单 1001（快照基于最新已提交版本）
-- 每次 SELECT 都重新生成快照，所以能看到B新提交的数据
```

**可重复读（REPEATABLE READ）下 MVCC 工作方式**

事务开启时创建一致性快照，整个事务期间都使用这个快照：

```sql
-- 事务A（REPEATABLE READ）
START TRANSACTION;  -- 获取快照视图

-- 事务B插入并提交
INSERT INTO orders VALUES(1001, '商品A');
COMMIT;

SELECT * FROM orders;  -- 看不到订单1001（使用事务开启时的快照）
-- 即使B已提交，A也看不到

-- 但对于自己当前事务中修改的数据，是可见的：
INSERT INTO orders VALUES(1002, '商品B');
SELECT * FROM orders;  -- 可以看到订单1002（当前事务修改的）
COMMIT;
```

**MVCC 与 Next-Key Lock 的关系**

在 REPEATABLE READ 隔离级别下，InnoDB 使用 **Next-Key Lock**（记录锁 + 间隙锁）来防止幻读：

```sql
-- 防止幻读示例
SELECT * FROM orders WHERE id > 100 AND id < 200 FOR UPDATE;
-- 锁定 id 在 (100, 200) 之间的"间隙"，阻止其他事务插入新记录
```

**性能优化建议**

```sql
-- 1. 事务尽量小：减少锁持有时间，减少 undo log 积累
BEGIN;
UPDATE orders SET status = 1 WHERE id = 100;
COMMIT;  -- 立即提交，不要在事务中做无关操作

-- 2. 读写分离：主库写、从库读，减少主库并发压力
-- 3. 合理选择隔离级别：
--    READ COMMITTED：适合大多数业务（Oracle默认级别）
--    REPEATABLE READ：仅在需要防止幻读时使用（InnoDB默认）
```

---

## 题目四：PostgreSQL 支持哪些高级索引类型？GIN、GiST、BRIN 索引分别适合什么场景？

### 参考答案

**PostgreSQL 索引类型一览**

| 索引类型 | 原理 | 适用场景 |
|----------|------|----------|
| B-tree（默认） | 平衡多叉树 | =、>、<、BETWEEN、LIKE（前缀） |
| Hash | 哈希表 | 等值查询（=） |
| GIN | 倒排索引 | 多值列、全文检索、JSONB、数组 |
| GiST | 通用搜索树 | 几何GIS、范围类型、全文检索 |
| BRIN | 块范围索引 | 物理顺序与业务顺序一致的表 |
| Partial | 部分索引 | 只索引满足条件的行 |
| Expression | 表达式索引 | 函数/表达式计算结果的索引 |

**GIN 索引（Generalized Inverted Index）**

```sql
-- 适合：数组包含查询、JSONB 键值查询、全文检索
CREATE INDEX idx_tags ON posts USING GIN(tags);  -- tags 为 TEXT[]

-- 数组包含查询走索引
SELECT * FROM posts WHERE tags @> ARRAY['Python', 'PostgreSQL'];

-- JSONB 索引
CREATE INDEX idx_properties ON products USING GIN(properties);
SELECT * FROM products WHERE properties @> '{"color": "red"}';
```

**GiST 索引（Generalized Search Tree）**

```sql
-- 适合：几何图形范围查询（R-Tree）、全文检索（PostgreSQL 内置）
-- 地理位置范围查询
CREATE INDEX idx_location ON stores USING GIST(geo);
SELECT * FROM stores WHERE geo && ST_MakeEnvelope(...);  -- 矩形范围查询

-- 范围类型索引
CREATE INDEX idx_age_range ON employees USING GIST(age_range);
SELECT * FROM employees WHERE age_range @> 30;  -- 包含30岁的员工
```

**BRIN 索引（Block Range Index）**

```sql
-- 适合：超大型表，且数据物理存储顺序与查询条件有序（如时间序列）
-- 原理：将数据按块分组，记录每块的 MIN/MAX 值，扫描时跳过无关块

CREATE INDEX idx_created_at ON events USING BRIN(created_at);

-- 前提条件：数据按 created_at 顺序插入（append-only）
-- 对时间范围查询极有效：BRIN 只需扫描少量块
SELECT * FROM events WHERE created_at BETWEEN '2026-01-01' AND '2026-01-31';
-- 扫描块数远少于 B-tree（全表 1000 个块，BRIN 可能只需扫描 10 个块）
```

**性能对比与选型**

```sql
-- 测试：B-tree vs BRIN 在时间序列上的性能差异
CREATE TABLE events AS SELECT generate_series AS id, 
  now() - (random()*interval '365 days') AS created_at
  FROM generate_series(1, 10000000);

CREATE INDEX idx_btree ON events USING B-tree(id);
CREATE INDEX idx_brin ON events USING BRIN(created_at);

EXPLAIN SELECT * FROM events WHERE created_at BETWEEN '2026-06-01' AND '2026-06-30';
-- BRIN: Index Scan 几十毫秒
-- B-tree: Index Scan 同样几十毫秒（但 BRIN 索引体积小 100 倍）
```

**选型原则**

- 数据量大且查询条件有**物理有序性**（时间序列）→ BRIN（索引体积极小）
- 多值/数组/JSONB/全文检索 → GIN
- GIS/几何范围查询 → GiST
- 普通等值/范围查询 → B-tree

---

## 题目五：数据库连接池的原理是什么？HikariCP 为什么是目前性能最优的连接池？如何调优？

### 参考答案

**数据库连接池的核心原理**

```
应用发起 SQL → 连接池分配空闲连接 → 执行 SQL → 归还连接到池
                ↑ 如果池空 → 等待/新建/拒绝
```

关键指标：
- **Maximum Pool Size**：最大连接数（一般 = CPU核心数 × 2 + 磁盘数）
- **Minimum Idle**：最小空闲连接
- **Connection Timeout**：等待连接超时
- **Idle Timeout**：空闲连接超时回收时间
- **Max Lifetime**：连接最大存活时间

**HikariCP 为什么快**

1. **轻量级**：HikariCP 核心 jar 仅约 130KB（对比 Druid ~1.4MB）
2. **字节码优化**：使用 Javassist 字节码库预生成连接代理类，减少反射开销
3. **Fast-path 优化**：`ConcurrentBag` 数据结构实现无锁化获取/归还连接
4. **零拷贝**：绕过 JDBC 直达数据库驱动，减少一次内存拷贝
5. **超时精确控制**：使用 `LongAdder` 高效计数，避免 `synchronized` 竞争

```java
// HikariCP 核心配置
HikariConfig config = new HikariConfig();
config.setJdbcUrl("jdbc:mysql://localhost:3306/shop");
config.setUsername("root");
config.setPassword("password");
config.setMaximumPoolSize(20);          // 一般推荐：CPU cores * 2
config.setMinimumIdle(5);               // 建议和 maximumPoolSize 一致或接近
config.setConnectionTimeout(30000);     // 30秒，获取连接超时
config.setIdleTimeout(600000);           // 10分钟，空闲回收
config.setMaxLifetime(1800000);          // 30分钟，连接最大生命周期
config.setPoolName("ShopDB-Pool");
```

**常见连接池对比**

| 特性 | HikariCP | Druid | c3p0 |
|------|----------|-------|------|
| 性能 | 最优 | 中等 | 差 |
| 监控 | 基础 | 完善（内置 Web 界面）| 弱 |
| Filters | 无 | 强（SQL 防火墙） | 无 |
| 启动慢连接检测 | 支持 | 支持 | 不支持 |

**调优经验值**

```yaml
# Spring Boot + HikariCP 调优建议
spring:
  datasource:
    hikari:
      # OLTP 场景（高频短查询）：池大 + 超时短
      maximum-pool-size: 20
      minimum-idle: 10
      connection-timeout: 10000
      idle-timeout: 300000
      
      # OLAP 场景（复杂查询）：池小 + 超时长
      # maximum-pool-size: 5
      # connection-timeout: 120000
```

**连接池排查命令**

```sql
-- MySQL 查看当前连接数
SHOW STATUS LIKE 'Threads_connected';
SHOW VARIABLES LIKE 'max_connections';

-- PostgreSQL
SELECT count(*) FROM pg_stat_activity;
SELECT setting AS max_conn FROM pg_settings WHERE name = 'max_connections';

-- HikariCP 监控（JMX / Micrometer）
-- 添加配置：spring.datasource.hikari.register-mbeans = true
-- 通过 JConsole 观察：Active Count、Idle Count、Wait Count、Total Connections
```

---

> 📌 以上题目均经过筛选，适合中高级工程师面试准备。建议结合自身项目经验补充细节。
