# 数据库性能优化面试题（2026-05-06）

> 本文档整理了 5 道高质量的 MySQL/PostgreSQL 数据库性能优化面试题，涵盖索引优化、查询优化、存储引擎、缓存策略等核心知识点。题目来源见各题附注。

---

## 题目一：MySQL 中如何定位并优化慢 SQL？（索引失效与 EXPLAIN 分析）

### 参考答案

定位慢 SQL 通常分以下几步：

**1. 开启慢查询日志**

```sql
-- 查看慢查询是否开启
SHOW VARIABLES LIKE 'slow_query_log%';

-- 开启慢查询日志，设置阈值（如超过 1 秒记录）
SET GLOBAL slow_query_log = 'ON';
SET GLOBAL long_query_time = 1;
SET GLOBAL slow_query_log_file = '/var/lib/mysql/slow.log';
```

**2. 使用 EXPLAIN 分析执行计划**

```sql
EXPLAIN SELECT * FROM orders WHERE status = 'paid' ORDER BY create_time DESC;
```

重点关注以下字段：

| 字段 | 含义 |
|------|------|
| `type` | 查询类型，`const/eq_ref/ref/range/index/all` 逐步变差，**最好达到 `ref` 级别** |
| `key` | 实际使用的索引 |
| `rows` | 扫描行数，越少越好 |
| `Extra` | 包含 `Using filesort`、`Using temporary` 说明需要额外排序或临时表，应优化 |
| `possible_keys` | 可能用到的索引 |

**3. 常见索引失效场景**

- WHERE 条件中使用 `OR`（除非两侧字段都有索引）
- 字符串字段使用 LIKE 以 `%` 开头，如 `LIKE '%abc'`
- WHERE 条件中索引列参与运算或函数：`WHERE YEAR(create_time) = 2024`
- 数据类型不一致导致隐式转换：`status` 为 int 但写 `WHERE status = '1'`（字符串）
- 未遵循最左前缀原则：创建了 `INDEX(a,b,c)` 但查询只用 `WHERE b = 1`
- 范围查询（`>`、`<`、`BETWEEN`）右边的列无法使用索引

**4. 优化手段**

- 添加合适索引：为 WHERE、ORDER BY、JOIN 条件的列建立索引
- 避免 `SELECT *`，只查需要的列（可利用覆盖索引）
- 分解关联查询，减少锁竞争
- 分页优化：延迟关联，先用索引分页再关联

> **来源：** [CSDN - MySQL数据库面试题（2024最新版）](https://blog.csdn.net/2401_87555681/article/details/143655474)  
> **来源：** [腾讯云 - 面试中被问到SQL优化](https://cloud.tencent.com/developer/article/2142125)

---

## 题目二：MyISAM 与 InnoDB 存储引擎的区别？如何选型？

### 参考答案

| 对比项 | MyISAM | InnoDB |
|--------|--------|--------|
| 事务支持 | ❌ 不支持 | ✅ 支持 ACID 事务 |
| 锁粒度 | 表级锁 | 行级锁 + 表级锁 |
| 外键 | ❌ 不支持 | ✅ 支持 |
| 全文索引 | ✅ 支持 | ❌ 不支持（5.6+ 支持） |
| 索引实现 | 非聚簇索引 | 聚簇索引（主键索引叶子节点存行数据） |
| 崩溃恢复 | 慢，需修复 | 自动崩溃恢复，快 |
| SELECT 性能 | 优（适合读多） | 较低（写并发能力强） |
| `COUNT(*)` | 快（内部维护计数器） | 慢（全表扫描） |
| 文件结构 | `.MYD` 数据 + `.MYI` 索引 | `.ibd` 数据和索引统一存储 |

**InnoDB 的核心优势：**
- 支持行级锁，并发写性能更好
- 支持事务和外键，数据完整性高
- 通过 MVCC（多版本并发控制）减少读写冲突
- 支持自适应哈希索引（AHI）加速查询

**选型建议：**
- **InnoDB**：绝大多数场景（更新频繁、并发量高、需要事务支持）
- **MyISAM**：读多写少、不需要事务的场景，如博客、新闻类应用

> **来源：** [CSDN - MySQL数据库面试题（2024最新版）](https://blog.csdn.net/2401_87555681/article/details/143655474)

---

## 题目三：如何设计高效的索引？有哪些优化策略？

### 参考答案

**1. 遵循最左前缀原则**

复合索引 `(A, B, C)` 可加速以下查询：
- `WHERE A = x`
- `WHERE A = x AND B = y`
- `WHERE A = x AND B = y AND C = z`

但无法加速 `WHERE B = x` 或 `WHERE C = x`。

**2. 前缀索引**

对超长字符串字段（如 VARCHAR(255)）使用前缀索引：

```sql
ALTER TABLE users ADD KEY (email(10));  -- 取前 10 个字符
```

通过 `SELECT COUNT(DISTINCT LEFT(email,10))/COUNT(*)` 选择合适前缀长度，平衡选择性。

**3. 覆盖索引**

查询的所有列都包含在索引中，MySQL 直接从索引返回数据，无需回表：

```sql
-- 如果 idx(name, age) 存在
SELECT name, age FROM users WHERE name = '张三';  -- 覆盖索引，极快
```

**4. 避免重复索引和冗余索引**

- `(A, B)` 和 `(A)` 重复，保留 `(A, B)` 即可
- 主键索引无需额外唯一索引

**5. 索引列避免表达式或函数**

```sql
-- 错误：索引列参与运算，失效
SELECT * FROM orders WHERE YEAR(create_time) = 2024;

-- 正确：使用范围查询，索引有效
SELECT * FROM orders WHERE create_time >= '2024-01-01' AND create_time < '2025-01-01';
```

**6. 范围查询放在最后**

范围查询右侧的列无法使用索引：

```sql
-- 假设 INDEX(status, create_time)
-- 这个查询 create_time 无法使用索引
SELECT * FROM orders WHERE status = 'paid' AND create_time > '2024-01-01';

-- 调整顺序也无法解决，应将范围查询放最后或拆分
```

> **来源：** [CSDN - MySQL数据库面试题（2024最新版）](https://blog.csdn.net/2401_87555681/article/details/143655474)  
> **来源：** [InfoQ - 数据库面试题从浅入深高频必刷（2024版）](https://xie.infoq.cn/article/cf7b39eb313b054732ccd23b0)

---

## 题目四：PostgreSQL 的 MVCC 机制是什么？如何影响性能与并发？

### 参考答案

**1. 什么是 MVCC**

MVCC（Multi-Version Concurrency Control，多版本并发控制）通过为每个事务保存数据的**时间点快照**，实现读写不阻塞：

- **写操作**：不覆盖旧数据，生成新版本（行记录），旧版本通过 `t_xmin` / `t_xmax` 字段标记事务状态
- **读操作**：根据事务的 `snapshot`，读取符合条件的可见版本，无需加读锁

**2. 事务隔离级别**

| 隔离级别 | 脏读 | 不可重复读 | 幻读 |
|----------|------|------------|------|
| Read Uncommitted | 可能 | 可能 | 可能 |
| Read Committed | ❌ | 可能 | 可能 |
| Repeatable Read | ❌ | ❌ | 可能（PostgreSQL通过MVCC几乎避免） |
| Serializable | ❌ | ❌ | ❌ |

PostgreSQL 默认 `Read Committed`，每个语句生成新快照。

**3. MVCC 对性能的影响**

**优点：**
- 读不阻塞写，写不阻塞读，并发性能好
- 避免脏读和脏写，数据一致性高

**缺点：**
- 旧版本数据积累导致表膨胀（**表膨胀 / bloat**）
- 需要定期 `VACUUM` 清理 dead tuples
- `SELECT COUNT(*)` 无法利用索引（需全表扫描）

**4. 性能优化建议**

```sql
-- 手动 VACUUM 清理
VACUUM ANALYZE orders;

-- 查看表膨胀情况
SELECT relname, n_dead_tup, n_live_tup FROM pg_stat_user_tables;

-- 配置 autovacuum 自动清理
ALTER TABLE orders SET (autovacuum_vacuum_scale_factor = 0.01);
```

> **来源：** [博客园 - 数据库面试总结（PostgreSQL MVCC）](https://www.cnblogs.com/zerogbc/p/18370893)  
> **来源：** [腾讯云 - 面试中被问到SQL优化](https://cloud.tencent.com/developer/article/2142125)

---

## 题目五：数据库缓存策略有哪些？如何与 Redis / Memcached 配合优化性能？

### 参考答案

**1. 缓存策略分类**

| 策略 | 原理 | 适用场景 |
|------|------|----------|
| **Cache Aside** | 先查缓存，命中则返回；未命中查 DB，写入缓存 | 读多写少，最常用 |
| **Read Through** | 缓存层自动加载数据，应用只与缓存交互 | 缓存透明化 |
| **Write Through** | 同步写缓存和 DB，任意一个失败则回滚 | 数据一致性要求高 |
| **Write Behind** | 异步批量写 DB，高并发场景 | 写入性能要求极高，允许短暂不一致 |

**2. MySQL + Redis 缓存经典模式**

```python
# Cache Aside 实现
def get_user(user_id):
    # Step 1: 查 Redis
    user = redis.get(f"user:{user_id}")
    if user:
        return json.loads(user)
    
    # Step 2: 缓存未命中，查 MySQL
    user = db.query("SELECT * FROM users WHERE id = %s", user_id)
    
    # Step 3: 写入 Redis，设置过期时间
    if user:
        redis.setex(f"user:{user_id}", 3600, json.dumps(user))  # 1小时过期
    
    return user
```

**3. 缓存常见问题及解决**

| 问题 | 描述 | 解决方案 |
|------|------|----------|
| **缓存穿透** | 查询不存在的数据，绕过缓存直击 DB | 布隆过滤器 / 缓存空值 |
| **缓存击穿** | 热点 key 过期，瞬间大量请求打到 DB | 互斥锁 / 热点数据永不过期 |
| **缓存雪崩** | 大量 key 同时过期 | 过期时间加随机值 / 多级缓存 |
| **数据一致性** | 缓存与 DB 数据不同步 | 先更新 DB，再删缓存（而非更新缓存） |

**4. SQL 级别优化补充**

- 批量操作：减少网络往返，如 `INSERT INTO ... VALUES (...), (...), (...)`
- 分页优化：延迟关联，覆盖索引分页
- 避免复杂 JOIN：大表拆小表，分步查询
- 使用 `EXPLAIN ANALYZE` 查看实际执行计划

> **来源：** [梯子教程网 - MySQL性能优化面试题](https://www.tizi365.com/question/topic-411.html)  
> **来源：** [阿里云开发者社区 - 架构面试题汇总：40道题吃透mysql（2024版）](https://developer.aliyun.com/article/1549790)

---

## 小结

数据库性能优化是综合性问题，核心围绕以下方向：

1. **索引优化** — 正确的索引设计是性能基石
2. **查询优化** — EXPLAIN 分析 + 避免全表扫描
3. **存储引擎选择** — InnoDB vs MyISAM 按场景选型
4. **缓存策略** — Redis/Cache 减少数据库压力
5. **运维规范** — 定期 VACUUM、慢查询分析、参数调优

---
*整理于 2026-05-06 | 更多面试题访问：https://github.com/qq286158530/interview-questions*