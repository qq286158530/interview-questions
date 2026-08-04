# 数据库性能优化面试题

> 📅 整理日期：2026-08-04
> 🏷️ 标签：MySQL | PostgreSQL | 性能优化 | 索引 | 查询优化

---

## 题目一：MySQL 索引失效的场景有哪些？如何避免？

### 参考答案

**常见的索引失效场景：**

1. **使用 SELECT *** — 只查询索引字段时可命中覆盖索引，避免回表。
2. **WHERE 子句中对索引列使用函数或表达式** — 如 `WHERE YEAR(create_time) = 2024` 会导致索引失效，应改为范围查询 `WHERE create_time >= '2024-01-01' AND create_time < '2025-01-01'`。
3. ** LIKE 开头使用通配符** — `LIKE '%keyword'` 无法使用 B+Tree 有序性，应改用前缀匹配或 Elasticsearch。
4. **类型转换** — 当列类型为 varchar，传入参数为 int 时会发生隐式类型转换，如 `WHERE phone = 13800138000`（phone 为 varchar）。
5. **OR 连接了非索引列** — `WHERE index_col = A OR normal_col = B` 会导致全表扫描，应拆分为 UNION。
6. **不等于比较** — `!=`、`<>` 在某些版本/优化器下无法利用索引。
7. **ORDER BY 的字段不在索引最左前缀** — 或 SELECT 了非索引列导致额外排序。
8. **多表 JOIN 时 ORDER BY 字段不在驱动表索引中**。

**优化建议：**
- 使用 `EXPLAIN` 分析执行计划，关注 `type`、`key`、`Extra` 列。
- 遵循最左前缀原则创建复合索引。
- 将范围查询（如 `>`、`<`、`IN`）放在复合索引的末尾。
- 尽量使用覆盖索引（索引包含所有 SELECT 字段）。

---

## 题目二：如何定位并优化慢查询？

### 参考答案

**定位慢查询的步骤：**

1. **开启慢查询日志**
   ```sql
   -- MySQL
   SET GLOBAL slow_query_log = 'ON';
   SET GLOBAL long_query_time = 1; -- 超过1秒记录
   SET GLOBAL slow_query_log_file = '/var/log/mysql/slow.log';
   ```

2. **使用 EXPLAIN 分析执行计划**
   ```sql
   EXPLAIN SELECT * FROM orders WHERE status = 'paid' AND create_time > '2024-01-01';
   ```
   重点关注：
   - `type`：最好达到 `ref/range`，避免 `ALL`（全表扫描）
   - `key`：实际使用的索引
   - `rows`：扫描行数，越少越好
   - `Extra`：出现 `Using filesort`、`Using temporary` 表示需要优化

3. **使用 SHOW PROFILE**
   ```sql
   SET profiling = 1;
   -- 执行查询
   SHOW PROFILES;
   SHOW PROFILE FOR QUERY 1;
   ```

4. **使用 Performance Schema（MySQL 5.6+）**
   ```sql
   SELECT * FROM performance_schema.events_statements_summary_by_digest ORDER BY SUM_TIMER_WAIT DESC LIMIT 10;
   ```

**常见优化手段：**
- 添加合适的索引
- 优化 SQL 结构（避免嵌套子查询，合理使用 JOIN）
- 拆分大事务为小事务
- 使用 EXPLAIN 的 `format=json` 获取更详细的成本分析

---

## 题目三：MySQL InnoDB 与 MyISAM 存储引擎如何选择？各自的适用场景是什么？

### 参考答案

| 特性 | InnoDB | MyISAM |
|------|--------|--------|
| 事务支持 | ✅ 支持 ACID 事务 | ❌ 不支持 |
| 行锁 | ✅ 支持行级锁，高并发 | ❌ 只支持表级锁 |
| 外键 | ✅ 支持 | ❌ 不支持 |
| 全文索引 | 5.6+ 支持 | ✅ 原生支持 |
| 崩溃恢复 | ✅ 自动恢复（redo log） | ❌ 需手动修复 |
| 存储空间 | 约 2 倍 MyISAM | 较小 |

**选择建议：**

- **选用 InnoDB**：
  - 需要事务支持（银行、订单、支付等业务）
  - 高并发写入场景（行级锁争用更小）
  - 需要崩溃快速恢复
  - 主从架构中作为 Master

- **选用 MyISAM**：
  - 以读为主、几乎无写操作（如配置表、历史日志表）
  - 需要全文索引且无法升级到 MySQL 5.6+
  - 查询频率远高于写入，且数据一致性要求不高

> ⚠️ MySQL 8.0 已将 MyISAM 标记为废弃，强烈建议优先使用 InnoDB。

---

## 题目四：什么是数据库连接池？PHP-Redis / Go-SQL 等客户端的连接池如何调优？

### 参考答案

**连接池的原理：**
数据库连接建立需要三次握手等网络开销。连接池在启动时预创建 N 个连接，请求到来时直接分配空闲连接，用完后归还，而非每次请求都新建/销毁连接。

**主要参数调优：**

| 参数 | 说明 | 调优建议 |
|------|------|----------|
| `max_connections` | MySQL 最大连接数 | 根据 `max_connections = (核心数 * 2) + 磁盘数` 估算 |
| `wait_timeout` | 空闲连接超时 | 避免连接长期占用，建议 8 小时（28800s） |
| `interactive_timeout` | 交互式连接超时 | 与 wait_timeout 保持一致 |
| `thread_cache_size` | 线程缓存大小 | `SHOW STATUS LIKE 'Threads_created'` 监控，避免频繁创建线程 |

**连接池配置（以 Python DBUtils 为例）：**
```python
pool = PooledDB(
    creator=pymysql,
    maxconnections=20,   # 最大连接数
    mincached=5,         # 初始空闲连接数
    maxcached=10,        # 最大空闲连接数
    blocking=True        # 连接耗尽时阻塞等待
)
```

**常见问题：**
- **连接泄漏**：获取连接后未正确释放，使用 `try-finally` 或上下文管理器确保归还。
- **连接数打满**：检查是否有慢查询占用连接，增大 `max_connections` 同时优化慢查询。
- **长事务**：事务未提交/回滚会导致连接长期占用，及时关闭事务。

---

## 题目五：Redis 缓存与数据库双写一致性如何保证？有哪些方案及其权衡？

### 参考答案

**三种常用方案：**

### 方案一：Cache Aside（旁路缓存）— 最常用

```
读：先读缓存，缓存未命中则读数据库，再写入缓存
写：先更新数据库，再删除缓存（而非更新缓存）
```

- 优点：实现简单，读性能高
- 缺点：删除缓存失败时可能出现短暂不一致（可采用延迟双删策略）
- 适用场景：读多写少，一致性要求适中

### 方案二：Read/Write Through（读写穿透）

- 应用程序只与缓存交互，缓存负责同步更新数据库
- 优点：一致性由缓存中间件保证
- 缺点：需要缓存支持写入穿透（如 Redis 本身不支持写入穿透，需借助其他组件）

### 方案三：Write Behind（异步写入）

- 写操作先写入缓存，异步批量同步到数据库
- 优点：写入性能极高
- 缺点：数据安全性低，缓存故障可能丢数据
- 适用场景：日志收集、计数服务等可容忍最终一致的业务

**延迟双删（解决删除失败问题）：**
```python
# 1. 先更新数据库
db.update(user)
# 2. 删除缓存
redis.delete('user:123')
# 3. 延迟 N 毫秒后再删除一次（应对并发读写导致的老数据回填）
import time
time.sleep(0.5)
redis.delete('user:123')
```

**分布式锁方案（强一致）：**
```python
# 使用 Redis SETNX 加锁，确保更新数据库和更新缓存原子执行
lock = redis.set('lock:user:123', '1', nx=True, ex=5)
if lock:
    try:
        db.update(user)
        redis.set('user:123', json.dumps(user))
    finally:
        redis.delete('lock:user:123')
```

---

## 📚 参考来源

1. [MySQL 索引失效场景详解 - 掘金](https://juejin.cn)
2. [MySQL 慢查询优化完全手册 - 美团技术团队](https://tech.meituan.com)
3. [InnoDB 与 MyISAM 存储引擎对比 - MySQL 官方文档](https://dev.mysql.com/doc/refman/8.0/en/innodb-introduction.html)
4. [数据库连接池调优指南 - 高并发数据库连接](https://www.cnblogs.com)
5. [Redis 与数据库双写一致性方案 - 阿里云开发者社区](https://developer.aliyun.com)

---

*由 AI 自动整理 | 欢迎 Star ⭐ [ interview-questions](https://github.com/qq286158530/interview-questions)*
