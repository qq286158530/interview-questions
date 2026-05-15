# 数据库性能优化面试题

> 日期：2026-05-15
> 来源：整理自小林coding图解MySQL系列

---

## 题目一：MySQL索引失效的场景有哪些？

### 参考答案

在工作中，建立索引并不意味着任何查询语句都能走索引扫描。以下是常见的6种索引失效场景：

### 1. 左模糊或左右模糊匹配（like %xx / like %xx%）

```sql
-- 索引失效
SELECT * FROM t_user WHERE name LIKE '%林';

-- 索引生效（前缀匹配可以走索引）
SELECT * FROM t_user WHERE name LIKE '林%';
```

**原因**：索引B+树按索引值有序排列存储，只能根据前缀比较。后缀模糊匹配无法确定起始位置。

### 2. 对索引字段使用函数

```sql
-- 索引失效（全表扫描）
SELECT * FROM t_user WHERE LENGTH(name) = 3;

-- 需要创建函数索引才能走索引
ALTER TABLE t_user ADD INDEX idx_name_length ((LENGTH(name)));
SELECT * FROM t_user WHERE LENGTH(name) = 3; -- 此时走索引
```

**原因**：索引保存的是原始值，而不是函数计算后的值。

### 3. 对索引进行表达式计算

```sql
-- 索引失效
SELECT * FROM t_user WHERE id + 1 = 10;

-- 改写后可走索引
SELECT * FROM t_user WHERE id = 10 - 1;
```

**原因**：索引保存的是字段原始值，表达式计算后无法利用索引。

### 4. 索引字段发生隐式类型转换

```sql
-- phone是varchar类型，输入参数是整型，索引失效
SELECT * FROM t_user WHERE phone = 12345678901;

-- 反过来，id是整型，输入字符串参数，可以走索引
SELECT * FROM t_user WHERE id = '10';
```

**原因**：MySQL遇到字符串和数字比较时，会将字符串转为数字。相当于对索引列使用了CAST函数。

### 5. 联合索引非最左匹配

联合索引(a, b, c)遵循最左匹配原则：

```sql
-- 可以走索引
WHERE a = 1
WHERE a = 1 AND b = 2
WHERE a = 1 AND b = 2 AND c = 3

-- 索引失效
WHERE b = 2
WHERE c = 3
WHERE b = 2 AND c = 3
```

**原因**：联合索引数据先按第一列排序，第一列相同才按第二列排序，不连续使用会导致全局无序。

### 6. WHERE子句中OR前是索引列，OR后不是索引列

```sql
-- 索引失效（OR后条件列不是索引）
SELECT * FROM t_user WHERE id = 1 OR age = 22;
```

**解决**：将OR后的列也建立索引。

---

**来源**：[图解MySQL - 索引失效有哪些？](https://xiaolincoding.com/mysql/index/index_lose.html)

---

## 题目二：为什么MySQL InnoDB选择B+树作为索引的数据结构？

### 参考答案

### 1. B+Tree vs B Tree

- B+树只在叶子节点存储数据，B树的非叶子节点也要存储数据
- B+树单个节点数据量更小，相同磁盘I/O下能查询更多节点
- B+树叶子节点采用双链表连接，适合范围查找，B树无法做到

### 2. B+Tree vs 二叉树

- B+Tree搜索复杂度为O(logdN)，d为最大子节点个数（实际d>100）
- 二叉树搜索复杂度为O(logN)，但d=2，高度更高
- 数据量达千万级时，B+树只需3-4层，磁盘I/O次数少

### 3. B+Tree vs Hash

- Hash等值查询O(1)，效率很高
- 但Hash不适合范围查询，只适合等值查询
- B+树索引适用范围更广泛

### B+树结构特点

- 非叶子节点只存放索引（冗余索引）
- 叶子节点存放实际数据/主键值
- 叶子节点之间通过双向链表连接
- 千万级数据只需3-4次磁盘I/O即可定位

### InnoDB索引实现

- **聚簇索引（主键索引）**：叶子节点存放完整行数据
- **二级索引**：叶子节点只存放主键值，查询需回表
- **覆盖索引**：查询的字段在二级索引叶子节点中，无需回表

---

**来源**：[图解MySQL - 索引常见面试题](https://xiaolincoding.com/mysql/index/index_interview.html)

---

## 题目三：MySQL查询性能优化的常用方法有哪些？

### 参考答案

### 1. 索引优化

```sql
-- 避免全表扫描，为WHERE、JOIN、ORDER BY、GROUP BY的字段建索引
EXPLAIN SELECT * FROM orders WHERE customer_id = 100;

-- 利用覆盖索引减少回表
SELECT order_id, create_time FROM orders WHERE customer_id = 100;
```

### 2. 避免SELECT *

```sql
-- 低效：返回所有字段
SELECT * FROM orders WHERE order_id = 12345;

-- 高效：只查询需要的字段
SELECT order_id, status, total_amount FROM orders WHERE order_id = 12345;
```

### 3. 大批量插入优化

```sql
-- 批量插入，减少磁盘I/O
INSERT INTO orders VALUES
(1, 'A', 100.0),
(2, 'B', 200.0),
(3, 'C', 300.0);

-- 关闭唯一性检查和事务自动提交
SET unique_checks = 0;
SET autocommit = 0;
-- 批量插入...
SET autocommit = 1;
SET unique_checks = 1;
```

### 4. 分页查询优化

```sql
-- 低效：OFFSET过大时性能差
SELECT * FROM orders LIMIT 100000, 10;

-- 优化1：利用主键索引
SELECT * FROM orders WHERE id > 100000 LIMIT 10;

-- 优化2：子查询+JOIN
SELECT a.* FROM orders a 
INNER JOIN (SELECT id FROM orders LIMIT 100000, 10) b ON a.id = b.id;
```

### 5. 批量更新优化

```sql
-- 低效：逐条更新
UPDATE orders SET status = 'shipped' WHERE id = 1;
UPDATE orders SET status = 'shipped' WHERE id = 2;

-- 高效：批量更新
UPDATE orders SET status = 'shipped' WHERE id IN (1, 2, 3, 4, 5);
```

### 6. SQL语句优化

```sql
-- 避免隐式类型转换
-- phone是VARCHAR类型，不要写成：WHERE phone = 123

-- 避免在WHERE条件中对字段做运算
-- 低效：WHERE id + 1 = 10
-- 高效：WHERE id = 10 - 1

-- 使用EXPLAIN分析执行计划
EXPLAIN SELECT * FROM orders WHERE customer_id = 100;
```

---

**来源**：[图解MySQL - MySQL查询流程](https://xiaolincoding.com/mysql/base/how_select.html)

---

## 题目四：InnoDB和MyISAM存储引擎有什么区别？如何选择？

### 参考答案

### 核心区别

| 特性 | InnoDB | MyISAM |
|------|--------|--------|
| 事务支持 | 支持ACID事务 | 不支持事务 |
| 锁粒度 | 行级锁 + 表级锁 | 表级锁 |
| 外键 | 支持外键 | 不支持外键 |
| 索引类型 | 聚簇索引（数据在索引中） | 非聚簇索引（索引指向数据地址） |
| 记录存储 | 主键索引叶子节点存储完整数据 | 主键索引叶子节点存储数据物理地址 |
| 崩溃恢复 | 支持崩溃自动恢复（redo log） | 崩溃后可能损坏 |
| 并发性能 | 高并发下性能更优 | 只支持表级锁，并发差 |
| 全文索引 | 5.6+支持全文索引 | 支持全文索引 |

### InnoDB适用场景

- **需要事务支持**：如金融、订单系统
- **高并发读写**：行级锁支持更多并发
- **数据一致性要求高**：崩溃恢复能力强
- **表中有主键**：自动创建聚簇索引

### MyISAM适用场景

- **只读/静态数据**：不需要事务的场景
- **全文搜索**：较早版本的全文索引更成熟
- **空间占用敏感**：表级锁占用内存少
- **日志系统**：写少读多的场景

### 注意事项

从 MySQL 5.5 开始，InnoDB 成为默认存储引擎。在MySQL 8.0中，InnoDB已支持FULLTEXT索引，MyISAM的优势进一步减少。

---

**来源**：[MySQL存储引擎对比](https://xiaolincoding.com/mysql/architecture/mysql_architecture.html)

---

## 题目五：如何设计高效的数据库缓存策略？

### 参考答案

### 1. 缓存层级设计

```
应用层 → 本地缓存（如Caffeine/Guava） → 分布式缓存（如Redis） → 数据库
```

### 2. 缓存更新策略

#### Cache-Aside（旁路缓存）最常用

```python
# 读操作
def get_user(user_id):
    user = redis.get(f"user:{user_id}")
    if user is None:
        user = db.query("SELECT * FROM users WHERE id = ?", user_id)
        redis.setex(f"user:{user_id}", 3600, user)
    return user

# 写操作：先更新数据库，再删除缓存
def update_user(user_id, data):
    db.execute("UPDATE users SET ... WHERE id = ?", user_id)
    redis.delete(f"user:{user_id}")  # 而不是更新缓存
```

#### Read-Through

缓存负责从数据库加载数据，应用只与缓存交互。

#### Write-Through

写入时同步更新缓存和数据库。

#### Write-Behind

异步写入数据库，可能丢失数据但性能最高。

### 3. 缓存过期策略

```sql
-- 根据数据更新频率设置TTL
-- 频繁更新：TTL = 5~10分钟
-- 低频更新：TTL = 24小时
-- 配置类数据：TTL = 1小时或不过期
```

### 4. 缓存击穿/雪崩/穿透解决方案

```sql
-- 缓存击穿：使用互斥锁或双检
-- Redis SETNX 实现
SETNX lock:product:10086 "1"
EXPIRE lock:product:10086 10
-- 业务逻辑
DEL lock:product:10086

-- 缓存雪崩：过期时间加上随机值
TTL = base_ttl + random(0, 300)  # 5分钟基础TTL + 0~5分钟随机

-- 缓存穿透：布隆过滤器或空值缓存
-- 布隆过滤器：快速判断key是否一定不存在
-- 空值缓存：对不存在的数据也缓存短TTL
```

### 5. Redis集群方案

```
主从复制（Read-Write Splitting）
  ├── Master: 写入
  ├── Slave1: 读（报表）
  ├── Slave2: 读（业务）
  └── Sentinel: 自动故障转移

或采用 Redis Cluster 做数据分片
```

### 6. 缓存监控指标

- **命中率**：hit_rate = hits / (hits + misses)
- **内存使用率**：used_memory / maxmemory
- **响应时间**：P99延迟
- **淘汰次数**：evicted_keys

---

**来源**：[图解MySQL - Buffer Pool](https://xiaolincoding.com/mysql/buffer_pool/buffer_pool.html)

---

> 📚 更多面试题请访问：[小林coding图解MySQL](https://xiaolincoding.com/mysql/)