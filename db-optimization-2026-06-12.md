# 数据库性能优化面试题

> 📅 更新时间：2026-06-12  
> 🐼 出题人：小憨宝  
> 📚 涵盖：MySQL / PostgreSQL 索引优化、查询优化、存储引擎、缓存策略

---

## 题目一：什么是覆盖索引？如何利用覆盖索引优化查询？

### 参考答案

**覆盖索引（Covering Index）是指一个索引包含了查询所需的所有字段（SELECT、WHERE、ORDER BY 等子句中涉及的列），无需回表查询主键数据行即可完成查询。**

### 利用覆盖索引优化的原理

普通索引查询流程：
1. 通过二级索引找到主键值（可能多棵B+树）
2. **回表**，根据主键到聚簇索引中读取完整数据行
3. 返回结果

覆盖索引查询流程：
1. 通过二级索引直接找到所有需要的数据（叶子节点已包含所有列）
2. **无需回表**，直接返回结果

### 示例

```sql
-- 假设有联合索引 (name, age, city)
-- 查询：SELECT name, age FROM users WHERE name = '张三';
-- 覆盖索引：idx_name_age_city(name, age, city)

EXPLAIN SELECT name, age FROM users WHERE name = '张三';
-- Extra 列显示 Using index，表示使用了覆盖索引
```

### 优化技巧

1. **分析 EXPLAIN 结果**：Extra 列出现 `Using index` 说明使用了覆盖索引
2. **避免 SELECT ***：只查询必要的列，让更多查询能够利用覆盖索引
3. **合理设计联合索引**：将高频查询的列按选择性和常用顺序组合
4. **索引下推（Index Condition Pushdown, ICP）**：MySQL 5.6+ 会在索引遍历时直接用索引列过滤条件，减少回表次数

### 注意事项

- 覆盖索引适合查询列少、筛选条件固定的场景
- 索引列不宜过多，否则写入性能下降（维护成本高）
- PostgreSQL 中对应概念为 **Index Only Scan**

> 📎 来源：[MySQL 官方文档 - Multi-Range Read Optimization](https://dev.mysql.com/doc/refman/8.0/en/optimization.html)

---

## 题目二：MySQL InnoDB 的 MVCC 机制是如何工作的？解决了什么问题？

### 参考答案

**MVCC（Multi-Version Concurrency Control，多版本并发控制）是一种乐观并发控制机制，InnoDB 通过它实现读写并发，避免了读操作加锁导致的性能开销。**

### 核心原理

InnoDB 为每行数据维护两个隐藏列：
- **DB_TRX_ID**：最近一次修改该行的事务 ID
- **DB_ROLL_PTR**：指向undo日志链的指针（用于回滚）

同时每条记录还有一个 **DELETE FLAG**（删除标记）。

### 读已提交（READ COMMITTED）与可重复读（REPEATABLE READ）的区别

两者核心差异在于 **快照的创建时机**：

| 隔离级别 | 快照时机 | 行为描述 |
|---------|---------|---------|
| READ COMMITTED | 每次读取SQL时生成 | 能看到其他事务已提交的修改 |
| REPEATABLE READ | 事务开始时生成 | 整个事务内读到的数据一致，start trans 时的状态 |

### 可见性判断规则

InnoDB 根据以下规则判断某行版本对当前事务是否可见：
1. **trx_id < 当前活跃最小事务ID** → 可见（该事务已提交）
2. **trx_id 在未提交列表中** → 不可见
3. **trx_id >= 当前事务ID** → 不可见（当前事务修改的）

### 解决了什么问题

- **读写不阻塞**：普通的 SELECT 不加锁，避免了读操作被写阻塞
- **减少锁竞争**：同一行数据的读和写可以并发执行
- **避免脏读和不可重复读**：通过版本控制保证隔离性

### 注意事项

- MVCC 只在 REPEATABLE READ 和 READ COMMITTED 隔离级别下生效
- 写操作（INSERT/UPDATE/DELETE）仍然需要排他锁
- 长事务会导致 undo 日志链过长，占用大量磁盘空间（undo tablespace）
- UPDATE/DELETE 操作在 InnoDB 中是先标记再写入，实际是物理删除在 purge 阶段

> 📎 来源：[MySQL 官方文档 - InnoDB Multi-Versioning](https://dev.mysql.com/doc/refman/8.0/en/innodb-multi-versioning.html)

---

## 题目三：PostgreSQL 的 MVCC 与 MySQL 有什么不同？各有什么优劣？

### 参考答案

### PostgreSQL MVCC 实现机制

PostgreSQL 使用 **Transaction ID (xmin/xmax)** + **Command ID** 的方式实现MVCC：

- **xmin**：插入该行版本的事务ID
- **xmax**：删除/更新该行版本的事务ID（为0表示未被删除）

PostgreSQL 通过 **Heap Only Tuple (HOT)** 优化更新：当更新同一页面内的行时，直接在原页面构造新版本，避免更新索引。

### 核心差异对比

| 维度 | MySQL InnoDB | PostgreSQL |
|-----|-------------|-----------|
| 实现方式 | 回滚段（Undo Log）+ 聚簇索引 | 隐藏列（xmin/xmax）+ Heap 堆表 |
| 索引结构 | 聚簇索引 = 主键数据，叶子节点存完整行 | 堆表 + 独立索引，索引叶子节点存 TID |
| 更新行为 | 原地更新，旧版本移至undo表空间 | 新建元组（直到VACUUM清理） |
| 垃圾回收 | 后台purge线程异步清理 | VACUUM 进程主动清理 |
| 事务ID | 5字节紧凑递增，高水位后 wrap around | 32位递增，VACUUM处理oldest xmin |

### MySQL InnoDB 优势
- **写入性能更好**：原地更新，避免碎片
- **聚簇索引减少回表**：主键查询只需一颗B+树

### MySQL InnoDB 劣势
- 依赖 undo 日志恢复历史版本，占用缓冲池
- 长事务可能导致 undo 膨胀

### PostgreSQL 优势
- **读性能更稳定**：不依赖undo，heap元组直接可见性判断
- **并发控制更精细**：每个事务快照独立，无脏读风险
- **支持更多隔离级别**：可序列化和可重复读都原生支持

### PostgreSQL 劣势
- **更新代价高**：旧版本不清理会膨胀
- **索引维护开销大**：每条更新需维护所有索引（HOT除外）
- **需要定期VACUUM**：维护负担重

### 实战建议

```sql
-- PostgreSQL 查看事务状态
SELECT txid_current();

-- 查看长事务（可能导致 VACUUM 无法回收）
SELECT * FROM pg_stat_activity WHERE state = 'active' AND query_start < now() - interval '10 minutes';
```

> 📎 来源：[PostgreSQL 官方文档 - MVCC](https://www.postgresql.org/docs/current/mvcc.html)

---

## 题目四：数据库分页查询深度分页（limit N, M）性能差，如何优化？

### 参考答案

### 深度分页性能问题原因

```sql
-- 性能极差的深度分页
SELECT * FROM orders ORDER BY id LIMIT 1000000, 10;
```

问题分析：
1. MySQL 需要先排序 **1000010 条记录**
2. 只返回最后 10 条，但排序过程无法跳过
3. 索引只能加速排序，无法减少扫描行数
4. 越深分页，扫描行数越多，IO放大严重

### 优化方案

#### 方案一：游标分页（基于主键）

```sql
-- 第一页
SELECT * FROM orders ORDER BY id LIMIT 10;

-- 下一页：记住上一页最后一条的id
SELECT * FROM orders WHERE id > 1000000 ORDER BY id LIMIT 10;
```

原理：利用主键索引的顺序性，通过 WHERE id > last_id 直接定位，无需扫描前面的数据。

#### 方案二：覆盖索引 + 子查询

```sql
-- 利用覆盖索引先查出id，再关联
SELECT t.* FROM orders t
INNER JOIN (
    SELECT id FROM orders ORDER BY id LIMIT 1000000, 10
) AS tmp ON t.id = tmp.id;
```

内部子查询使用覆盖索引只扫描 id 列不回表，然后关联回原表获取所有字段。

#### 方案三：倒排分页（针对大数据量）

```sql
-- 降序分页
SELECT * FROM orders ORDER BY id DESC LIMIT 10 OFFSET 1000000;

-- 优化：先查最大id，然后倒序
SELECT * FROM orders WHERE id < 1000000 ORDER BY id DESC LIMIT 10;
```

#### 方案四：缓存预热（适合固定分页）

对于管理后台等固定分页场景，可以：
1. 定期将分页数据物化到缓存表
2. 使用 Elasticsearch / ClickHouse 等搜索引擎处理复杂查询
3. 设置最大分页限制（如最多翻100页）

### 核心原则

| 分页深度 | 推荐方案 |
|---------|---------|
| 浅分页（< 10000） | 普通 LIMIT 即可，注意加索引 |
| 中等深度（1万~50万） | 游标分页（WHERE id >） |
| 深度分页（> 50万） | 覆盖索引子查询 或 搜索引擎 |
| 固定查询 | 缓存预热 或 物化视图 |

> 📎 来源：[MySQL 官方文档 - LIMIT Optimization](https://dev.mysql.com/doc/refman/8.0/en/limit-optimization.html)

---

## 题目五：如何设计一个数据库缓存策略？Redis 与 MySQL 如何保持一致性？

### 参考答案

### 缓存策略设计原则

#### 1. Cache Aside（旁路缓存）- 最常用

```
读：Cache → 命中 → 返回
    Cache → 未命中 → 读DB → 写Cache → 返回

写：写DB → 删除/更新Cache（而非更新Cache）
```

为什么写的时候删除而非更新？因为更新缓存后，在并发场景下可能造成数据不一致。

#### 2. Read Through / Write Through

- Read Through：缓存自动从DB加载，调用方只和缓存交互
- Write Through：写操作同时写缓存和DB，缓存保证一致性

### Redis 与 MySQL 一致性保障

#### 方案一：延迟双删（解决更新时的脏读）

```sql
-- 线程A：更新DB，删除Redis
UPDATE orders SET status = 1 WHERE id = 1;
DELETE FROM orders_cache WHERE id = 1;

-- 延迟500ms再删除（解决线程B在A更新DB前就写了旧数据到缓存）
-- 延迟删除放在异步任务中
Thread.sleep(500);
DELETE FROM orders_cache WHERE id = 1;
```

#### 方案二：订阅 MySQL Binlog 同步（Canal/Debezium）

```
MySQL → Binlog → Canal → Redis
         ↓
    按顺序重放
```

优点：完全解耦，数据变更自动同步；缺点：系统复杂度提升。

#### 方案三：RedLock 分布式锁

```python
# 获取锁再操作
lock = redis.setnx("order:1:lock", "1", ex=10)
if lock:
    # 读DB写缓存
    data = db.query("SELECT * FROM orders WHERE id = 1")
    redis.set("order:1", data)
    redis.delete("order:1:lock")
else:
    # 等待或直接读DB
    time.sleep(0.1)
```

### 一致性级别选择

| 业务场景 | 一致性要求 | 实现方式 |
|---------|-----------|---------|
| 金融/订单 | 强一致 | 分布式锁 + 延迟双删 |
| 电商商品 | 最终一致 | Cache Aside + Binlog同步 |
| 社交Feed | 弱一致 | 设置TTL过期 + 异步更新 |
| 配置中心 | 最终一致 | 短TTL + 主动推送 |

### 缓存常见问题处理

```sql
-- 缓存穿透：布隆过滤器或空值缓存
SELECT * FROM products WHERE id = ? AND is_deleted = 0;
-- 或用 Redis SETNX 空值
redis.setnx("cache:product:" + id, "NULL", ex=60);

-- 缓存击穿：互斥锁或热点数据永不过期
redis.set("product:1", value, ex=0);  -- 热点数据永不过期

-- 缓存雪崩：随机TTL + 限流降级
redis.set("key", value, ex=random(300, 600));
```

> 📎 来源：[Redis 官方文档 - Data types](https://redis.io/docs/data-types/)

---

> 🐼 关注公众号「程序员面试题库」，每日获取最新面试题推送！
> 📮 如有题目错误或建议，欢迎提交 PR 或 Issue