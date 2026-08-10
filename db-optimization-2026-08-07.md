# 数据库性能优化面试题（2026-08-07）

> 本文件收录 5 道高质量 MySQL/PostgreSQL 数据库性能优化面试题，涵盖索引优化、查询优化、存储引擎、缓存策略等核心知识点。

---

## 题目一：MySQL 中如何定位并优化慢查询？请描述完整的排查思路和优化方法。

### 参考答案

### Step 1：开启慢查询日志

```sql
-- 查看慢查询是否开启
SHOW VARIABLES LIKE 'slow_query_log';
SHOW VARIABLES LIKE 'long_query_time';

-- 开启慢查询日志（临时）
SET GLOBAL slow_query_log = 'ON';
SET GLOBAL long_query_time = 1; -- 超过1秒记录

-- 永久开启需在 my.cnf 中配置
[mysqld]
slow_query_log = 1
slow_query_log_file = /var/lib/mysql/slow.log
long_query_time = 1
```

### Step 2：分析慢查询日志

```bash
# 使用 mysqldumpslow 工具统计最慢的查询
mysqldumpslow -s t -t 10 /var/lib/mysql/slow.log
# -s t 按时间排序，-t 10 取前10条
```

### Step 3：使用 EXPLAIN 分析执行计划

```sql
EXPLAIN SELECT * FROM orders WHERE status = 1 AND create_time > '2026-01-01';

-- 重点关注以下字段：
-- type: 最好达到 ref/range，避免 ALL（全表扫描）
-- key: 实际使用的索引
-- rows: 扫描行数，越少越好
-- Extra: 避免 Using filesort、Using temporary
```

### Step 4：常见优化手段

| 优化手段 | 说明 |
|---------|------|
| 添加合适索引 | 在 WHERE、JOIN、ORDER BY 涉及列上建索引 |
| 避免 SELECT * | 只查需要的字段，减少网络传输和回表 |
| 优化 LIMIT 分页 | 使用延迟关联或游标分页 |
| 分解大查询 | 将复杂查询拆为多个简单查询 |
| 适当冗余字段 | 减少 JOIN 操作 |

**来源**：
- https://dev.mysql.com/doc/refman/8.0/en/slow-query-log.html
- https://www.mysql.com/products/enterprise/advisor.html

---

## 题目二：什么是最左前缀原则？组合索引失效的常见场景有哪些？

### 参考答案

### 最左前缀原则的定义

组合索引（如 `INDEX(a, b, c)`）从左到右依次使用索引字段，MySQL 会一直向右匹配直到遇到范围查询（`>`、`<`、`BETWEEN`、`LIKE`）就停止匹配。

```sql
-- 组合索引 INDEX(name, age, position)

-- ✅ 能使用索引（完全按最左前缀）
SELECT * FROM employees WHERE name = '张三';
SELECT * FROM employees WHERE name = '张三' AND age = 30;
SELECT * FROM employees WHERE name = '张三' AND age = 30 AND position = 'Engineer';

-- ✅ 能使用索引（范围查询停止）
SELECT * FROM employees WHERE name = '张三' AND age > 25;

-- ❌ 完全不适用索引（跳过最左列）
SELECT * FROM employees WHERE age = 30;
SELECT * FROM employees WHERE position = 'Engineer';
```

### 组合索引失效的常见场景

**1. 范围查询中断**
```sql
-- 组合索引 INDEX(a, b, c)
-- age 是范围查询，position 无法使用索引
SELECT * FROM users WHERE name = '张三' AND age > 25 AND position = 'Engineer';
-- 索引：a 用到了，b 用到了（age > 25），c 失效
```

**2. LIKE 前面加通配符**
```sql
-- 组合索引 INDEX(name, age)
-- '%张' 导致索引失效
SELECT * FROM users WHERE name LIKE '%张%';
-- 正确写法：name LIKE '张%'
```

**3. 索引列参与运算**
```sql
-- create_time 有索引，但参与运算导致失效
SELECT * FROM orders WHERE YEAR(create_time) = 2026;
-- 正确：create_time >= '2026-01-01' AND create_time < '2027-01-01'
```

**4. 隐式类型转换**
```sql
-- phone 是 VARCHAR，传入 INT 造成隐式转换
SELECT * FROM users WHERE phone = 13800138000;
-- 正确：WHERE phone = '13800138000'
```

**5. OR 条件中包含非索引列**
```sql
-- id 有索引，email 没有索引，OR 导致全表扫描
SELECT * FROM users WHERE id = 1 OR email = 'a@b.com';
-- 正确：拆分或为 email 建索引
```

**来源**：
- https://dev.mysql.com/doc/refman/8.0/en/multiple-column-indexes.html
- https://www.postgresql.org/docs/current/indexes-multicolumn.html

---

## 题目三：PostgreSQL 与 MySQL 在性能优化方面有哪些核心差异？PostgreSQL 有哪些独特的优化手段？

### 参考答案

### 核心架构差异

| 方面 | MySQL (InnoDB) | PostgreSQL |
|------|---------------|------------|
| 并发模型 | MVCC + 行级锁 | MVCC + 乐观锁（默认） |
| 索引类型 | B+Tree、主键/唯一/全文/空间 | B-Tree、GiST、GIN、SP-GiST、BRIN |
| 查询优化器 | 较简单，规则驱动为主 | 成本估算模型，复杂优化器 |
| 分区表 | 支持（MySQL 5.7+） | 支持（PG 10+） |
| 物化视图 | 不支持 | 支持（可定时刷新） |
| CTES/递归查询 | 不支持 | 原生支持 |

### PostgreSQL 独特的优化手段

**1. 使用 EXPLAIN ANALYZE 分析实际执行**
```sql
EXPLAIN (ANALYZE, BUFFERS, FORMAT TEXT)
SELECT * FROM orders WHERE status = 1;
-- BUFFERS 显示缓存命中情况，帮助判断是否需要调优
```

**2. 物化视图加速复杂查询**
```sql
-- 创建物化视图
CREATE MATERIALIZED VIEW monthly_sales AS
SELECT DATE_TRUNC('month', create_time) AS month, SUM(amount) AS total
FROM orders
GROUP BY DATE_TRUNC('month', create_time);

-- 手动刷新
REFRESH MATERIALIZED VIEW monthly_sales;

-- 配合索引
CREATE UNIQUE INDEX ON monthly_sales(month);
```

**3. 使用 BRIN 索引优化大表范围查询**
```sql
-- BRIN 索引适合物理顺序有序的列（如时间序列）
CREATE INDEX idx_orders_created_brin ON orders USING BRIN(create_time);

-- 对比 B-Tree 索引，BRIN 索引体积小几个数量级
```

**4. PostgreSQL 的并行查询**
```sql
-- 查看并行查询配置
SHOW max_parallel_workers_per_gather;

-- 强制使用并行（测试用）
SET max_parallel_workers_per_gather = 4;
SELECT COUNT(*) FROM large_table; -- 自动并行执行
```

**5. 使用 COPY 批量导入替代 INSERT**
```sql
-- 百万级数据导入，用 COPY 速度比 INSERT 快10倍以上
COPY orders(id, status, amount) FROM '/tmp/orders.csv' WITH (FORMAT csv);
```

**来源**：
- https://www.postgresql.org/docs/current/performance-tips.html
- https://www.postgresql.org/docs/current/mvcc.html
- https://www.postgresql.org/docs/current/BRIN.html

---

## 题目四：什么是数据库连接池？HikariCP、C3P0、Druid 连接池的核心参数和调优策略是什么？

### 参考答案

### 连接池的核心作用

数据库连接池在应用程序进程内维护一批数据库连接，避免每次请求都创建/销毁连接，减少 TCP 开销和数据库认证开销。

### 主流连接池对比

| 连接池 | 特点 | 默认最大连接数 |
|--------|------|--------------|
| HikariCP | 性能最高，Spring Boot 2.x 默认 | 10 |
| Druid | 阿里开源，监控能力强 | 20（阿里规范） |
| C3P0 | 老牌，兼容性差，已很少用 | 15 |

### HikariCP 核心参数详解

```yaml
spring:
  datasource:
    hikari:
      # 最小空闲连接数（默认等于 maximum-pool-size）
      minimum-idle: 5
      # 最大连接数（生产环境通常 20~50）
      maximum-pool-size: 20
      # 连接最大生命周期（默认 30分钟）
      max-lifetime: 1800000
      # 连接超时（默认 30秒）
      connection-timeout: 30000
      # 连接泄漏检测（开启后，超时未归还的连接会打日志）
      leak-detection-threshold: 60000
```

### Druid 核心参数详解

```yaml
spring:
  datasource:
    druid:
      # 初始连接数
      initial-size: 5
      # 最大连接数
      max-active: 20
      # 最小空闲连接
      min-idle: 5
      # 获取连接最大等待时间（毫秒）
      max-wait: 60000
      # 启用监控页面（生产环境注意安全）
      stat-view-servlet:
        enabled: true
      # SQL 防火墙
      wall:
        enabled: true
```

### 调优实战建议

**1. 计算合理的连接数**
```
最佳连接数 = (核心线程数 × CPU利用率) + 应对流量峰值的额外连接

公式：connections = ((核心数 × 2) + 磁盘数)
例如：8核 + 2块SSD ≈ 18 个连接
```

**2. 监控关键指标**
- `activeCount`：当前活跃连接数
- `idleCount`：空闲连接数
- `waitThreadCount`：等待获取连接的线程数
- `errorCount`：数据库操作错误数

**3. 常见问题排查**
- **连接超时**：`connection-timeout` 过短或数据库连接数上限不足
- **连接泄漏**：开启 `leak-detection-threshold`，检查是否忘记 `close()` 连接
- **连接池耗尽**：max-active 太小或慢查询占用连接时间过长

**来源**：
- https://github.com/brettwooldridge/HikariCP/wiki/About-Pool-Sizing
- https://github.com/alibaba/druid/wiki/DruidDataSource%E9%85%8D%E7%BD%AE%E5%B1%9E%E6%80%A7%E5%88%97%E8%A1%A8

---

## 题目五：数据库分库分表后，如何解决跨节点查询、分布式 ID 和数据迁移问题？

### 参考答案

### 为什么要分库分表

当单表数据量超过千万级（MySQL）或单库 QPS 超过万级时，需要将数据分散到多个库/表中。

```
垂直拆分：按业务模块分库（用户库、订单库、商品库）
水平拆分：按数据特征分片（按用户ID分表、按时间分表）
```

### 核心问题一：跨节点查询（分布式 JOIN）

**方案1：业务层组装（推荐）**
```java
// 查询用户及其订单，分两步查询后程序内组装
List<User> users = userMapper.selectByIds(userIds);
List<Order> orders = orderMapper.selectByUserIds(userIds);
Map<Long, List<Order>> orderMap = orders.stream()
    .collect(Collectors.groupingBy(Order::getUserId));
```

**方案2：ES/Hive 做联邦查询**
- 将需要跨表联合查询的字段冗余到 ES
- 使用 ES 的 `multi_index` 查询

**方案3：全局表**
- 在各分片中冗余一份字典表
- 适用于数据量小且变化不频繁的表

### 核心问题二：分布式 ID 生成

| 方案 | 优点 | 缺点 | 代表实现 |
|------|------|------|---------|
| UUID | 简单，无依赖 | 无序，长度大（36字节） | Java UUID.randomUUID() |
| 数据库自增 | 有序，简单 | 单点性能瓶颈 | MySQL AUTO_INCREMENT |
| Redis INCR | 性能高，可设置步长 | 依赖 Redis | Jedis.incr() |
| Snowflake | 有序，趋势递增 | 依赖时钟 | 百度 UidGenerator |
| 雪花算法变种 | 趋势递增 | 需改造 | 滴滴 TinyID、美团 Leaf |

**Snowflake 算法原理**：
```
1 位符号 + 41 位时间戳 + 10 位机器ID + 12 位序列号
= 64 位 long 类型整数
```

### 核心问题三：数据迁移（不停服迁移）

**Phase 1：双写阶段（灰度切流）**
```
旧系统：继续写入原表
新系统：应用同时写分表 + 原表（binlog 同步）
```

**Phase 2：数据校验与补偿**
```sql
-- 校验数据一致性
SELECT COUNT(*) FROM old_table;
SELECT SUM(shard_count) FROM new_shards;

-- 抽样比对
SELECT * FROM old_table WHERE id BETWEEN ? AND ?
MINUS
SELECT * FROM new_table WHERE id BETWEEN ? AND ?
```

**Phase 3：切换读流量**
```
逐步将读流量切换到新系统
观察监控，无异常后关闭双写
```

### 分库分表后必备基础设施

- **数据源路由**：ShardingSphere、MyCat、ShardingJDBC
- **统一入口**：Proxy 层统一管理分片逻辑
- **监控告警**：慢查询监控、连接数监控、分片数据倾斜监控

**来源**：
- https://shardingsphere.apache.org/
- https://github.com/alibaba/nacos
- https://www.postgresql.org/docs/current/ddl-partitioning.html

---

> 📌 以上题目均经过筛选，适合中高级工程师面试准备。建议结合自身项目经验补充细节。

> ⚠️ **提示**：部分外链在发布时可能已失效，如遇 404 请搜索对应技术文档官网获取最新内容。
