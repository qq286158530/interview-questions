# 数据库性能优化面试题

> 📅 日期：2026-07-05
> 🏷️ 标签：MySQL | PostgreSQL | 性能优化 | 索引 | 查询优化

---

## 题目一：MySQL 中索引失效的常见场景有哪些？

### 参考答案

索引失效是面试中的高频考点，以下是常见场景：

1. **使用左模糊查询 `LIKE '%xxx'`**
   - `%` 在左边会导致全表扫描，因为索引 B+ 树按从左到右顺序组织。
   - 解决：尽量使用右模糊 `LIKE 'xxx%'`，或使用全文索引。

2. **使用函数或运算**
   ```sql
   -- 错误示例
   SELECT * FROM orders WHERE YEAR(create_time) = 2026;
   -- 正确示例
   SELECT * FROM orders WHERE create_time >= '2026-01-01' AND create_time < '2027-01-01';
   ```

3. **隐式类型转换**
   - 如果列类型是 INT，但传入字符串 `'123'`，MySQL 会将字符串转为数字，导致索引失效。

4. **OR 查询有部分列不含索引**
   ```sql
   -- 如果 status 没有索引，整个查询会全表扫描
   SELECT * FROM orders WHERE order_id = 123 OR status = 1;
   -- 改用 UNION
   SELECT * FROM orders WHERE order_id = 123
   UNION
   SELECT * FROM orders WHERE status = 1 AND order_id != 123;
   ```

5. **不等于比较 `<>` / `NOT IN`**
   - 通常无法使用索引，会退化为全表扫描。

6. **联合索引违反最左前缀原则**
   - 对于 `(a, b, c)` 联合索引，以下情况能命中索引：`a=1`、`a=1 AND b=2`、`a=1 AND b=2 AND c=3`。
   - 但 `b=2` 或 `c=3` 无法使用索引。

7. **MySQL 优化器判断全表扫描更快**
   - 当数据量很小（几十行）或选择性很低时，MySQL 优化器可能放弃索引直接全表扫描。

> 📚 来源：[MySQL索引失效场景全面解析 - 掘金](https://juejin.cn/post/7142292003152478215)

---

## 题目二：如何定位并优化慢查询？

### 参考答案

**Step 1：开启慢查询日志**

```sql
-- 查看是否开启
SHOW VARIABLES LIKE 'slow_query_log';
-- 开启（临时）
SET GLOBAL slow_query_log = 'ON';
SET GLOBAL long_query_time = 1;  -- 超过1秒记录
```

**Step 2：分析慢查询日志**

```bash
# 使用 mysqldumpslow 工具
mysqldumpslow -s t -t 10 /var/lib/mysql/slow.log
# -s: 排序方式（t=时间, c=次数）
# -t: 显示前N条
```

**Step 3：使用 EXPLAIN 分析执行计划**

```sql
EXPLAIN SELECT * FROM orders WHERE status = 1;
```

重点关注：
- `type`：最优为 `const`/`eq_ref`，最差为 `ALL`（全表扫描）
- `key`：实际使用的索引
- `rows`：扫描行数，越少越好
- `Extra`：是否出现 `Using filesort`、`Using temporary`

**Step 4：常见优化手段**

| 优化方向 | 具体做法 |
|---------|---------|
| 索引优化 | 添加合适索引、避免索引失效 |
| SQL 改写 | 避免 SELECT *、减少 JOIN、拆解复杂子查询 |
| 分页优化 | 延迟关联 / 游标分页，避免 `OFFSET` 大偏移 |
| 表结构 | 适当冗余字段、分库分表 |
| 架构层 | 读写分离、引入缓存（Redis） |

> 📚 来源：[MySQL慢查询优化实战 - 腾讯云技术社区](https://cloud.tencent.com/developer/article/1928920)

---

## 题目三：MySQL InnoDB 与 MyISAM 存储引擎的核心区别是什么？

### 参考答案

| 特性 | InnoDB | MyISAM |
|------|--------|--------|
| 事务支持 | ✅ 支持 ACID 事务 | ❌ 不支持 |
| 行锁 | ✅ 行级锁，并发好 | ❌ 表级锁 |
| 外键 | ✅ 支持 | ❌ 不支持 |
| 全文索引 | 5.6+ 支持 | ✅ 原生支持 |
| 索引结构 | B+ Tree（聚簇索引） | B+ Tree（非聚簇） |
| 崩溃恢复 | ✅ 自动崩溃恢复（redo log） | ❌ 需手动修复 |
| 存储空间 | 约 2 倍（支持MVCC） | 较小 |
| 适用场景 | 事务场景、高并发、OLTP | 读多写少、不需要事务 |

**重点补充 —— 聚簇索引 vs 非聚簇索引：**

- **InnoDB（聚簇索引）**：数据和主键索引一起存储，叶子节点直接存放完整数据行。辅助索引叶子节点存储主键值，查数据需二次查询。
- **MyISAM（非聚簇）**：数据和索引分开存储，索引叶子节点存储的是数据地址。

```sql
-- 查看引擎
SHOW TABLE STATUS FROM db_name WHERE Name = 'orders';
-- 修改引擎
ALTER TABLE orders ENGINE = InnoDB;
```

> 📚 来源：[MySQL存储引擎InnoDB与MyISAM对比 - 掘金](https://juejin.cn/post/6916441063468105735)

---

## 题目四：什么是数据库连接池？为什么要使用连接池？

### 参考答案

**连接池（Connection Pool）** 是一种复用数据库连接的技术，在程序启动时预先建立一批连接并放入池中，应用需要数据库操作时从池中获取，用完归还而非真正关闭。

**不使用连接池的问题：**

```
应用请求 → 建立TCP连接 → 数据库认证 → 执行SQL → 关闭连接 → 响应
```

每次请求都要经历建立/销毁连接的过程，TCP 三次握手 + 数据库认证耗时 10~100ms，在高并发下性能急剧下降。

**使用连接池后的流程：**

```
应用请求 → 从池中获取连接 → 执行SQL → 归还连接 → 响应
```

连接复用，耗时降至微秒级。

**常见连接池对比：**

| 连接池 | 适用语言 | 特点 |
|--------|---------|------|
| HikariCP | Java | 性能最强，Spring Boot 默认 |
| Druid | Java | 阿里开源，功能丰富（监控、防御SQL注入） |
| PGBouncer | PostgreSQL | 服务端连接池，支持多种部署模式 |
| Pgpool-II | PostgreSQL | 支持连接池+负载均衡+自动 failover |

**核心配置参数：**

```properties
# 连接池大小建议 = (核心数 * 2) + 有效磁盘数
# 最小连接数不要设太大，避免空闲连接浪费资源
spring.datasource.hikari.maximum-pool-size=20
spring.datasource.hikari.minimum-idle=5
spring.datasource.hikari.connection-timeout=30000
```

> 📚 来源：[数据库连接池原理与实践 - 美团技术博客](https://tech.meituan.com/2020/04/09/database-connection-pool-principle.html)

---

## 题目五：Redis 缓存与数据库如何解决双写一致性？

### 参考答案

**三种经典策略：**

**策略一：Cache Aside（旁路缓存）—— 最常用**

```
读：先读缓存，缓存未命中则读数据库，再写入缓存
写：先更新数据库，再删除缓存（而不是更新缓存）
```

> 为什么删除而不是更新？因为删除是幂等的，更新可能因并发导致脏数据。

**策略二：Read/Write Through（读写穿透）**

- 应用只操作缓存，缓存负责同步到数据库。
- 对应用透明，但实现复杂，性能一般。

**策略三：Write Behind（异步写入）**

- 先写缓存，异步批量同步到数据库。
- 性能最高，但数据安全性最低，可能丢数据。

**数据不一致问题及解决：**

| 问题 | 场景 | 解决方案 |
|------|------|---------|
| 并发导致脏读 | 线程A写DB→线程B写DB→线程A删缓存→线程B删缓存 | 延迟双删：先删缓存→更新DB→延迟再删缓存（几百ms） |
| 缓存击穿 | 缓存过期瞬间大量请求打到DB | 互斥锁 / 逻辑过期 / 永不过期+异步更新 |
| 缓存雪崩 | 大量缓存同时过期 | 随机 TTL / 多级缓存 / 熔断降级 |

**延迟双删示例（Python 伪代码）：**

```python
def write(key, value):
    redis.delete(key)           # 第一步删缓存
    db.update(key, value)       # 第二步更新DB
    time.sleep(0.5)             # 延迟
    redis.delete(key)           # 第三步再删缓存（清除脏数据）
```

**最终一致性 vs 强一致性：**

- 大多数业务场景用 Cache Aside + 延迟双删（最终一致）就够了。
- 对数据强一致性要求高（如金融交易），应 **先删缓存，再更新数据库，再删缓存**，并结合消息队列重试补偿。

> 📚 来源：[Redis与MySQL双写一致性方案详解 - 腾讯云技术社区](https://cloud.tencent.com/developer/article/1973845)

---

*📌 本题库由 GitHub 自动化推送生成：https://github.com/qq286158530/interview-questions*
