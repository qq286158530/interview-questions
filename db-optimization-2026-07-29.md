# 数据库性能优化面试题

> 📅 更新时间：2026-07-29
> 🏷️ 标签：MySQL | PostgreSQL | 性能优化 | 索引 | 查询优化

---

## 题目一：MySQL 中 B+ 树索引的工作原理是什么？为什么比 B 树更适合数据库？

### ✅ 答案

**B+ 树 vs B 树结构差异**

```
B树：每个节点都存储数据
        [15] [25] [30]
       /     |      \
    [5,10] [15,20] [25,28] [30,40]

B+ 树：非叶子节点只存储索引，叶子节点存储所有数据且互联
           [25] [50] [75]
          /     |      \
    [5~20] - [25~45] - [50~70] - [75~100]
     ↓           ↓           ↓          ↓
   data        data        data       data
```

**B+ 树更适合数据库的 5 个原因**

| 特性 | B 树 | B+ 树 | 数据库优势 |
|------|------|-------|-----------|
| 树高 | 较高 | 较矮（相同数据） | I/O 次数更少 |
| 叶子节点 | 离散分布 | 链表互联 | 范围查询极快 |
| 非叶子节点 | 含数据 | 仅存索引 | 索引页能容纳更多条目 |
| 查询稳定性 | O(logN) ~ O(N) | O(logN) 稳定 | 查询时间可预测 |
| 全表扫描 | 需遍历所有节点 | 只需遍历叶子链表 | 扫描效率更高 |

**InnoDB B+ 树的具体特点**

```sql
-- 聚集索引（Clustered Index）：主键索引即 B+ 树，叶子节点存完整行数据
-- 辅助索引（Secondary Index）：叶子节点存主键值，需回表查询

EXPLAIN SELECT * FROM users WHERE id = 100;
-- id 为主键，直接通过聚集索引定位，效率最高

EXPLAIN SELECT * FROM users WHERE email = 'test@example.com';
-- email 为辅助索引，先查辅助索引拿到主键，再回表查聚集索引
```

**页（Page）与磁盘预读**

- InnoDB 默认页大小 16KB
- B+ 树每个节点为一个页，一次 I/O 读取一个页
- 磁盘预读：读取一个页时会预读相邻页到 Buffer Pool
- B+ 树高 3~4 层即可支撑千万级数据

> 📚 参考：[MySQL 索引底层：B+ 树详解](https://www.cnblogs.com/wuanlife/p/7041515.html)

---

## 题目二：PostgreSQL 的 MVCC 机制是如何工作的？如何避免 VACUUM 带来的性能问题？

### ✅ 答案

**MVCC（Multi-Version Concurrency Control）原理**

PostgreSQL 通过每行数据的两个隐藏列实现 MVCC：

```
xmin: 插入该行的事务 ID
xmax: 删除/更新该行的事务 ID（为0表示未被删除）
```

事务快照（Snapshot）决定"能看到哪些版本"：

```sql
-- 事务隔离级别为 READ COMMITTED
SELECT * FROM orders; -- 看到的是快照时的可见版本

-- 不同事务看到不同版本
Transaction A: INSERT INTO orders VALUES (1, 'A');  -- xmin=100
Transaction B: UPDATE orders SET name='B' WHERE id=1; -- 原行 xmax=101, 新行 xmin=101
-- 事务A仍能看到 name='A'，事务B看到 name='B'
```

**版本链与可见性判断**

```
事务100插入一行 → xmin=100, xmax=0
         ↓
事务101更新该行 → 原行标记 xmax=101，生成新行 xmin=101
         ↓
新事务判断可见性：
  - 原行：如果 xmax 已提交 且 快照中无此事务 → 不可见
  - 新行：如果 xmin 已提交 且 快照中有此事务 → 可见
```

**VACUUM 的作用**

```sql
-- VACUUM 清理死亡元组（dead tuples），释放磁盘空间
VACUUM orders;                    -- 普通VACUUM，不锁表
VACUUM FULL orders;               -- 压缩重建，需排他锁（生产禁用）
VACUUM ANALYZE orders;            -- VACUUM + 更新统计信息
```

**VACUUM 性能问题及优化**

❄️ **问题场景：大量更新/删除后表臃肿，VACUUM 耗时过长**

| 优化策略 | 说明 |
|---------|------|
| `autovacuum_vacuum_threshold = 50` | 超过50条死亡元组自动触发 |
| `autovacuum_vacuum_scale_factor = 0.1` | 超过表10%行数时触发 |
| `autovacuum_analyze_threshold = 50` | 分析阈值 |
| 调低 `autovacuum_vacuum_cost_delay` | 降低 VACUUM I/O 开销 |
| 分批删除 | 大批量 DELETE 时分批提交，减少死亡元组堆积 |
| `VACUUM` 而非 `DELETE` | 批量删除后立即 VACUUM |

```sql
-- 生产环境推荐配置（postgresql.conf）
autovacuum_max_workers = 4          -- 并行worker数
autovacuum_naptime = 20s            -- 检查间隔
autovacuum_vacuum_cost_delay = 10ms -- 降低I/O影响
autovacuum_vacuum_scale_factor = 0.05  -- 表5%时触发（更频繁但每次量小）
```

**监控 VACUUM 状态**

```sql
SELECT schemaname, relname, n_dead_tup, n_live_tup, last_vacuum, last_autovacuum
FROM pg_stat_user_tables
WHERE n_dead_tup > 10000
ORDER BY n_dead_tup DESC;
```

> 📚 参考：[PostgreSQL 官方文档 - MVCC](https://www.postgresql.org/docs/current/mvcc.html)

---

## 题目三：如何设计合理的索引能够提升查询性能？有哪些索引设计原则需要遵循？

### ✅ 答案

**索引设计核心原则**

| 原则 | 说明 | 反例 |
|------|------|------|
| 查询驱动 | 根据高频查询建索引，而非字段 | 为每个列建索引 |
| 选择性高 | 区分度高的列优先（80%以上唯一） | 性别、状态码建索引 |
| 最左前缀 | 复合索引从左到右使用 | 跳过复合索引第一列 |
| 覆盖索引 | 索引包含所有查询列，避免回表 | SELECT * 频繁回表 |
| 长度可控 | 字符串索引可指定前缀长度 | 长文本字段全列索引 |

**复合索引设计实战**

```sql
-- 场景：用户表，查询 WHERE status = 1 AND city = '北京' ORDER BY create_time DESC

-- ❌ 低效：为每个字段单独建索引
CREATE INDEX idx_status ON users(status);
CREATE INDEX idx_city ON users(city);

-- ✅ 高效：复合索引，遵循最左前缀 + 覆盖
CREATE INDEX idx_status_city_time ON users(status, city, create_time DESC);

-- ✅ 覆盖索引：查询字段全部在索引中，无需回表
EXPLAIN SELECT status, city, create_time FROM users
WHERE status = 1 AND city = '北京';
-- Extra: Using index（完全使用索引）
```

**前缀索引 vs 全列索引**

```sql
-- 字符串长度大时，用前缀索引减少空间
ALTER TABLE users ADD INDEX idx_email (email(10));

-- 前缀长度选择：区分度 > 0.9 最佳
SELECT COUNT(DISTINCT LEFT(email, 5)) / COUNT(*) AS sel_5,
       COUNT(DISTINCT LEFT(email, 10)) / COUNT(*) AS sel_10,
       COUNT(DISTINCT LEFT(email, 15)) / COUNT(*) AS sel_15
FROM users;
-- 选择接近全列区分度的最小前缀长度
```

**索引失效排查**

```sql
-- 使用 EXPLAIN 分析
EXPLAIN SELECT * FROM orders WHERE YEAR(create_time) = 2026;
-- 问题：使用了函数，索引失效

-- ✅ 改写为范围查询
EXPLAIN SELECT * FROM orders 
WHERE create_time >= '2026-01-01' AND create_time < '2027-01-01';
-- 使用 create_time 上的索引
```

**索引数量控制**

- 单表索引数量建议 ≤ 5~7 个
- 过多索引增加写入开销（每次 INSERT/UPDATE 需更新所有索引）
- 定期使用 `OPTIMIZE TABLE` 重建索引，删除冗余索引

```sql
-- 查看冗余索引
SELECT a.index_name, a.table_name, b.index_name AS redundant_index
FROM information_schema.statistics a
JOIN information_schema.statistics b 
  ON a.table_name = b.table_name 
 AND a.column_name = b.column_name
WHERE a.seq_in_index = 1 AND b.seq_in_index = 1
  AND a.index_name != b.index_name;
```

> 📚 参考：[MySQL 索引设计原则](https://zhuanlan.zhihu.com/p/400658265)

---

## 题目四：数据库连接池有哪些核心参数？如何调优以提升性能？

### ✅ 答案

**连接池核心参数解析**

| 参数 | 含义 | 过小问题 | 过大问题 |
|------|------|---------|---------|
| `maxConnections` | 最大连接数 | 并发不足，请求排队 | 数据库压力大，资源耗尽 |
| `minIdle` | 最小空闲连接 | 突发流量时频繁创建连接 | 浪费空闲连接资源 |
| `connectionTimeout` | 获取连接超时 | 短时大量请求超时 | 问题被隐藏，难发现 |
| `idleTimeout` | 空闲连接存活时间 | - | 连接长期占用 |
| `maxLifetime` | 连接最大生命周期 | - | 连接过期导致事务失败 |
| `poolName` | 连接池名称 | - | 日志排查用 |

**常见连接池对比**

| 连接池 | 特性 | 适用场景 |
|--------|------|---------|
| HikariCP | 性能最高，代码轻量 | Java 默认，性能敏感 |
| Druid | 监控强大，阿里开源 | 需要 SQL 监控 |
| c3p0 | 老牌，功能全 | 遗留系统 |
| PgBouncer | PostgreSQL 专用 | 多进程 Python/Node |
|ProxySQL | MySQL 中间件 | 读写分离架构 |

**HikariCP 调优实战**

```yaml
# application.yml
spring:
  datasource:
    hikari:
      maximum-pool-size: 20        # CPU核心数 * 2 + 磁盘节点数（MySQL建议20-50）
      minimum-idle: 5              # 正常负载下保活连接数
      connection-timeout: 30000    # 30秒获取连接超时
      idle-timeout: 600000         # 10分钟空闲回收
      max-lifetime: 1800000        # 30分钟强制断开重连
      connection-test-query: SELECT 1  # 旧版JDBC驱动需配置
```

**连接数计算公式**

```
HikariCP 官方建议：
maximum-pool-size = ((核心数 * 2) + 磁盘数)

MySQL 实际配置：
max_connections = min(连接池总和 + 10~20缓冲, 100~200)

PostgreSQL：
max_connections = 100~300 配合 PgBouncer 使用
```

**连接池监控指标**

```sql
-- MySQL 查看当前连接数
SHOW STATUS LIKE 'Threads_connected';  -- 当前连接
SHOW STATUS LIKE 'Max_used_connections'; -- 历史峰值

-- 查看连接来源
SHOW PROCESSLIST;
```

**连接池泄漏排查**

```java
// Druid 监控连接泄漏
// 配置 removeAbandoned=true 自动关闭超时连接
druid.removeAbandoned=true
druid.removeAbandoned-timeout=30  // 30秒超时

// 监控面板
// http://ip:8080/druid/index.html
```

> 📚 参考：[HikariCP 配置指南](https://github.com/brettwooldridge/HikariCP/wiki/About-Pool-Sizing)

---

## 题目五：如何优化大表的 DDL 操作（添加字段、添加索引）？有哪些非阻塞方案？

### ✅ 答案

**大表 DDL 的风险**

```sql
-- ❌ 直接加索引：表锁，全表复制，线上不可用
ALTER TABLE orders ADD INDEX idx_user_id(user_id);
-- MySQL 5.6 之前：Copy to New Table，1000万数据可能阻塞数小时
-- MySQL 5.6+：Online DDL，但仍可能长时间锁表
```

**MySQL 原生 Online DDL**

```sql
-- 添加索引，使用 ALGORITHM=INPLACE 避免全表复制
ALTER TABLE orders ADD INDEX idx_user_id(user_id), 
ALGORITHM=INPLACE, LOCK=NONE;

-- 添加字段，确保不为 NULL 时允许并发 DML
ALTER TABLE orders ADD COLUMN remark VARCHAR(500) DEFAULT '' COMMENT '备注',
ALGORITHM=INPLACE, LOCK=NONE;
```

**pt-online-schema-change（Percona Toolkit）**

```bash
# 安装
yum install percona-toolkit

# 执行在线加索引
pt-online-schema-change \
  --alter "ADD INDEX idx_user_id(user_id)" \
  --execute \
  --charset=utf8 \
  --user=root \
  --password=xxx \
  D=myapp,t=orders

# 工作原理：
# 1. 创建新表（新索引）
# 2. 在原表建触发器同步增量数据
# 3. 分批拷贝数据（每次1000行）
# 4. 交换新旧表
```

**gh-ost（GitHub Online Schema Tool）**

```bash
# MySQL 5.6+ 使用 gh-ost 无触发器方案
gh-ost \
  --origin="root:password@tcp(localhost:3306)/myapp" \
  --table="orders" \
  --alter="ADD INDEX idx_user_id(user_id)" \
  --execute

# 优点：
# - 不使用触发器，减少主库负载
# - 支持暂停、恢复
# - 幽灵表数据同步，可回滚
```

**大表分批删除**

```sql
-- ❌ 一次删除1000万行：产生大事务，阻塞并发
DELETE FROM logs WHERE create_time < '2025-01-01'; -- 1000万行

-- ✅ 分批删除，每次1000行，sleep降低压力
DELIMITER //
CREATE PROCEDURE batch_delete()
BEGIN
  DECLARE deleted INT DEFAULT 0;
  REPEAT
    DELETE FROM logs WHERE create_time < '2025-01-01' LIMIT 1000;
    SET deleted = ROW_COUNT();
    IF deleted > 0 THEN
      DO SLEEP(0.1);  -- 降低对主库影响
    END IF;
  UNTIL deleted = 0 END REPEAT;
END//
DELIMITER ;

CALL batch_delete();
```

**PostgreSQL 大表 DDL**

```sql
-- PostgreSQL 使用 CONCURRENTLY 避免锁
CREATE INDEX CONCURRENTLY idx_user_id ON orders(user_id);

-- 添加字段（PG 11+ 默认 Online）
ALTER TABLE orders ADD COLUMN remark VARCHAR(500);

-- VACUUM FULL 替代方案：pg_repack 重建表
pg_repack -t orders --order='id'
```

**DDL 执行前检查清单**

| 检查项 | 方法 |
|--------|------|
| 表大小 | `SELECT COUNT(*) FROM orders;` |
| 索引占用空间 | `SHOW TABLE STATUS LIKE 'orders';` |
| 预估执行时间 | 在从库测试 `EXPLAIN ALTER` |
| 主从延迟 | 监控 `Seconds_Behind_Master` |
| 业务低峰期 | 安排在凌晨2-6点 |

> 📚 参考：[MySQL Online DDL 原理](https://dev.mysql.com/doc/refman/8.0/en/innodb-online-ddl.html)

---

*整理自：牛客网、LeetCode、知乎、掘金等技术社区*
