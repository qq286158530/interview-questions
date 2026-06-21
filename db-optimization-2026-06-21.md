# 数据库性能优化面试题

> 📅 整理日期：2026-06-21  
> 🐼 来源：整理自小林coding、MySQL官方文档、PostgreSQL官方文档等

---

## 题目一：为什么 MySQL InnoDB 选择 B+Tree 作为索引的数据结构？

### 参考答案

**为什么不是 B-Tree？**

B+Tree 只在叶子节点存储数据，而 B-Tree 的非叶子节点也要存储数据，所以 B+Tree 的单个节点的数据量更小，在相同的磁盘 I/O 次数下，就能查询更多的节点。

另外，B+Tree 叶子节点采用的是双向链表连接，适合 MySQL 中常见的基于范围的顺序查找，而 B-Tree 无法做到这一点。

**为什么不是二叉树？**

对于有 N 个叶子节点的 B+Tree，其搜索复杂度为 O(logdN)，其中 d 表示节点允许的最大子节点个数为 d 个。在实际应用中，d 值是大于100的，这样就保证了即使数据达到千万级别时，B+Tree 的高度依然维持在 3~4 层左右，也就是说一次数据查询操作只需要做 3~4 次磁盘 I/O 操作。

而二叉树的每个父节点的儿子节点个数只能是 2 个，意味着其搜索复杂度为 O(logN)，二叉树检索到目标数据所经历的磁盘 I/O 次数要更多。

**为什么不是 Hash？**

Hash 在做等值查询时效率很高，搜索复杂度为 O(1)。但是 Hash 表不适合做范围查询，它更适合做等值的查询，这也是 B+Tree 索引要比 Hash 表索引有着更广泛的适用场景的原因。

**B+Tree 的优势总结：**

1. **单次 I/O 能加载更多数据**：B+Tree 非叶子节点不存储数据，每个节点能存储更多的索引值，树高更矮
2. **范围查询友好**：叶子节点双向链表便于范围查找
3. **查询稳定**：所有查询都需要走到叶子节点，时间复杂度固定
4. **适合磁盘存储**：节点大小设为页大小（如 16KB），每次 I/O 正好读取一个节点

> 📚 **来源**：[小林coding - 索引常见面试题](https://xiaolincoding.com/mysql/index/index_interview.html)

---

## 题目二：什么是联合索引的最左前缀原则？哪些情况会导致索引失效？

### 参考答案

### 联合索引结构

通过将多个字段组合成一个索引，该索引就被称为联合索引。比如，将商品表中的 `product_no` 和 `name` 字段组合成联合索引 `(product_no, name)`。

联合索引的 B+Tree 是先按 `product_no` 进行排序，然后再 `product_no` 相同的情况再按 `name` 字段排序。

### 最左前缀原则

在使用联合索引进行查询的时候，如果不遵循「最左匹配原则」，联合索引会失效，无法利用到索引快速查询的特性。

**能匹配上的情况：**
- `WHERE a = 1`
- `WHERE a = 1 AND b = 2 AND c = 3`
- `WHERE a = 1 AND b = 2`

**无法匹配的情况（索引失效）：**
- `WHERE b = 2`
- `WHERE c = 3`
- `WHERE b = 2 AND c = 3`

### 索引失效的常见场景

1. **左模糊匹配（LIKE %xx）**：使用 `LIKE %xx` 会导致索引失效，因为索引按前缀有序
2. **对索引列做计算、函数、类型转换**：如 `WHERE YEAR(create_time) = 2024`
3. **OR 前后不全是索引列**：`WHERE a = 1 OR b = 2`，如果 b 不是索引列，则索引失效
4. **联合索引不满足最左前缀**：如直接查 b、c 字段
5. **范围查询后的列**：在 `WHERE a > 1 AND b = 2` 中，b 列可能无法使用索引（`>=、<=、BETWEEN` 除外）

### 优化建议

- 建立联合索引时，把区分度大的字段排在前面
- 尽量使用覆盖索引（covering index），避免回表
- 使用 EXPLAIN 查看执行计划，确认是否使用索引

> 📚 **来源**：[小林coding - 索引常见面试题](https://xiaolincoding.com/mysql/index/index_interview.html)

---

## 题目三：MySQL InnoDB 和 PostgreSQL 的存储引擎/数据结构有何区别？如何选择？

### 参考答案

### MySQL InnoDB

| 特性 | 说明 |
|------|------|
| **索引结构** | B+Tree（聚簇索引），数据存储在叶子节点 |
| **聚簇索引** | 主键索引的叶子节点存储完整行数据 |
| **二级索引** | 叶子节点存储主键值，需要回表 |
| **事务支持** | 支持 ACID 事务，默认自动提交 |
| **锁机制** | 行级锁，支持 MVCC |
| **外键** | 支持 |
| **默认隔离级别** | REPEATABLE READ |

### PostgreSQL

| 特性 | 说明 |
|------|------|
| **索引结构** | B-Tree（默认）、Hash、GIN、GiST、BRIN 等多种索引 |
| **索引类型** | 支持表达式索引、部分索引、覆盖索引 |
| **事务支持** | 支持 ACID 事务，每个事务必须显式 BEGIN |
| **MVCC** | 真正的 MVCC，读不阻塞写，写不阻塞读 |
| **锁机制** | 多版本并发控制，细粒度锁 |
| **外键** | 支持 |
| **CTE 支持** | 强大的 WITH 子句（公用表表达式） |

### 核心区别

1. **MVCC 实现**：PostgreSQL 的 MVCC 更加完善，真正实现读写不互斥；InnoDB 的 REPEATABLE READ 隔离级别下可能产生幻读
2. **索引多样性**：PostgreSQL 支持更多索引类型（GiST 适合地理数据、GIN 适合全文搜索）
3. **表结构**：InnoDB 聚簇索引将数据与主键绑定；PostgreSQL 使用 MVCC + Heap 结构
4. **性能特点**：高并发写入场景 InnoDB 更优；复杂查询和地理信息场景 PostgreSQL 更优

### 选择建议

- **选择 InnoDB**：互联网高并发 Web 应用、OLTP 场景、简单 CRUD 操作
- **选择 PostgreSQL**：复杂数据分析、地理信息系统（GIS）、需要强大 SQL 标准支持、频繁复杂 JOIN 场景

> 📚 **来源**：[PostgreSQL 官方文档 - Performance Tips](https://www.postgresql.org/docs/current/performance-tips.html)

---

## 题目四：如何使用 EXPLAIN/EXPLAIN ANALYZE 分析 SQL 查询性能？

### 参考答案

### MySQL EXPLAIN

```sql
EXPLAIN SELECT * FROM orders WHERE user_id = 100;
```

**关键字段说明：**

| 字段 | 含义 |
|------|------|
| `type` | 扫描方式，从好到差：const > eq_ref > ref > range > index > ALL |
| `key` | 实际使用的索引 |
| `key_len` | 索引长度，越短越好 |
| `rows` | 预计扫描的行数，越少越好 |
| `Extra` | 额外信息，Using index（覆盖索引）最好，Using filesort 最差 |

**type 字段说明：**
- `const`：主键或唯一索引与常量比较，最多返回一条记录
- `eq_ref`：多表联查中，主键或唯一索引作为关联条件
- `ref`：普通索引等值匹配
- `range`：索引范围扫描
- `index`：全索引扫描
- `ALL`：全表扫描，应尽量避免

**Extra 常见值：**
- `Using index`：使用了覆盖索引，性能好
- `Using where`：在存储引擎层应用了 WHERE 条件
- `Using filesort`：需要额外排序，效率低
- `Using temporary`：使用了临时表，效率低

### PostgreSQL EXPLAIN/EXPLAIN ANALYZE

```sql
EXPLAIN (ANALYZE, BUFFERS, FORMAT TEXT) 
SELECT * FROM orders WHERE user_id = 100;
```

**PostgreSQL 特有：**
- `cost`：预估成本，分为启动成本和总成本
- `rows`：预估返回行数
- `actual time`：实际执行时间
- `loops`：实际执行次数
- `BUFFERS`：缓冲区命中情况，shared hit（命中缓存）vs read（从磁盘读）

### 优化步骤

1. 先用 EXPLAIN 查看执行计划，确认是否全表扫描
2. 关注 type 是否为 ALL（最差）
3. 检查 key 是否正确使用了索引
4. 用 EXPLAIN ANALYZE 查看实际执行时间与预估的差异
5. 检查是否有 Using filesort、Using temporary 等低效操作
6. 根据分析结果调整索引或 SQL 语句

> 📚 **来源**：[PostgreSQL 官方文档 - Using EXPLAIN](https://www.postgresql.org/docs/9.6/using-explain.html)

---

## 题目五：如何设计数据库缓存策略来提升性能？Redis 和 MySQL 如何配合使用？

### 参考答案

### 缓存策略模式

#### 1. Cache Aside（旁路缓存）- 最常用

**读操作：**
1. 先查 Redis 缓存
2. 缓存命中 → 直接返回
3. 缓存未命中 → 查 MySQL → 写入 Redis → 返回

**写操作：**
1. 先写 MySQL
2. 删除 Redis 缓存（而非更新，避免并发问题）

```python
# 伪代码示例
def get_user(user_id):
    user = redis.get(f"user:{user_id}")
    if user:
        return json.loads(user)
    user = db.query("SELECT * FROM users WHERE id = ?", user_id)
    redis.setex(f"user:{user_id}", 3600, json.dumps(user))
    return user

def update_user(user_id, data):
    db.execute("UPDATE users SET ... WHERE id = ?", user_id)
    redis.delete(f"user:{user_id}")  # 删除缓存
```

#### 2. Read Through（读穿透）

缓存自动加载数据，应用只与缓存交互。

#### 3. Write Through（写穿透）

写操作同时更新缓存和数据库。

#### 4. Write Behind（写回）

先写缓存，异步批量写回数据库。

### 缓存常见问题及解决方案

| 问题 | 解决方案 |
|------|---------|
| **缓存穿透** | 空值缓存、布隆过滤器 |
| **缓存击穿** | 互斥锁、永不过期 + 异步更新 |
| **缓存雪崩** | 随机过期时间、热点数据不过期 |
| **数据不一致** | 延迟双删、订阅 binlog 同步 |

### Redis + MySQL 最佳实践

1. **缓存粒度**：建议缓存整个对象而非单字段，序列化方式选 JSON 或 msgpack
2. **过期时间**：热点数据 1-60 分钟，冷数据可设更长
3. **Key 命名**：`业务:表:id` 如 `order:user:100`
4. **容量规划**：确保 Redis 内存 > 热点数据总量
5. **数据库瓶颈时再加缓存**：不要过早优化

### PostgreSQL 缓存

PostgreSQL 自带强大的缓存机制（shared_buffers），配合操作系统缓存（page cache）效果很好。对于复杂查询结果，可以使用 Materialized View 缓存结果。

> 📚 **来源**：[小林coding - MySQL 性能优化](https://xiaolincoding.com/mysql/optimize/optimize.html)
