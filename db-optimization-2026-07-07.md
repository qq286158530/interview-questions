# 数据库性能优化面试题

> 📅 整理日期：2026-07-07
> 🏷️ 涵盖：索引优化、查询优化、存储引擎、缓存策略

---

## 题目 1：MySQL 中索引失效的常见场景有哪些？

### 参考答案

索引失效是面试中的高频考点，以下是几种典型场景：

**1. 使用函数或运算**
```sql
-- 索引失效：使用了 YEAR() 函数
SELECT * FROM orders WHERE YEAR(create_time) = 2026;

-- 正确写法：保留索引可用
SELECT * FROM orders WHERE create_time >= '2026-01-01' AND create_time < '2027-01-01';
```

**2. 类型转换**
```sql
-- phone 是 varchar 类型，传入数字会导致隐式转换，索引失效
SELECT * FROM users WHERE phone = 13800138000;  -- 隐式 CAST

-- 正确写法：使用字符串
SELECT * FROM users WHERE phone = '13800138000';
```

**3. LIKE 百分号在左**
```sql
-- 索引失效：'%开头的模糊查询无法使用 B-Tree 索引
SELECT * FROM products WHERE name LIKE '%手机';

-- 索引有效：百分号在右边
SELECT * FROM products WHERE name LIKE '华为%';
```

**4. OR 连接条件**
```sql
-- 如果 age 有索引，name 没有索引，OR 会导致索引失效
SELECT * FROM users WHERE age = 25 OR name = '张三';

-- 改用 UNION 或确保两列都有索引
SELECT * FROM users WHERE age = 25
UNION ALL
SELECT * FROM users WHERE name = '张三' AND age != 25;
```

**5. 联合索引不遵循最左前缀原则**
```sql
-- 创建联合索引 (a, b, c)
ALTER TABLE orders ADD INDEX idx_a_b_c(a, b, c);

-- 失效：跳过 a
SELECT * FROM orders WHERE b = 1 AND c = 2;

-- 失效：中间断开
SELECT * FROM orders WHERE a = 1 AND c = 2;

-- 有效：遵循最左前缀
SELECT * FROM orders WHERE a = 1;
SELECT * FROM orders WHERE a = 1 AND b = 2;
```

**6. 使用 NOT、!=、<>**
```sql
-- NOT、!=、<> 会导致索引失效（部分版本 8.0+ 有所改善）
SELECT * FROM users WHERE status != 1;
```

**7. ORDER BY 的坑**
```sql
-- 如果 SELECT 只查主键，ORDER BY 同一索引列，可能Using filesort
-- 混合升序降序时可能无法使用索引排序
SELECT id FROM orders ORDER BY create_time ASC, id DESC;
```

---

## 题目 2：如何定位并优化慢查询？

### 参考答案

**Step 1：开启慢查询日志**
```sql
-- 查看慢查询配置
SHOW VARIABLES LIKE 'slow_query%';
SHOW VARIABLES LIKE 'long_query_time';

-- 临时开启（会话级）
SET GLOBAL slow_query_log = 'ON';
SET GLOBAL long_query_time = 1;  -- 超过 1 秒记录

-- 永久开启需在 my.cnf 配置
[mysqld]
slow_query_log = 1
slow_query_log_file = /var/log/mysql/slow.log
long_query_time = 1
```

**Step 2：使用 EXPLAIN 分析执行计划**
```sql
EXPLAIN SELECT u.name, o.amount
FROM users u
INNER JOIN orders o ON u.id = o.user_id
WHERE u.status = 1;

-- 重点关注：
-- type: const > eq_ref > ref > range > index > ALL（尽量避免 ALL）
-- key: 实际使用的索引
-- rows: 扫描行数，越少越好
-- Extra: Using filesort / Using temporary 需重点优化
```

**Step 3：使用 SHOW PROFILE**
```sql
SET profiling = 1;
-- 执行你的慢查询
SELECT ...
SHOW PROFILES;
SHOW PROFILE FOR QUERY 1;  -- 查看各阶段耗时
```

**Step 4：常见优化手段**

| 优化手段 | 说明 |
|---------|------|
| 添加合适索引 | 根据 WHERE、JOIN、ORDER BY 字段建立索引 |
| 优化 SQL 结构 | 避免 SELECT *，减少JOIN层级 |
| 分库分表 | 单表数据过大时水平拆分 |
| 读写分离 | 读从库、写主库 |
| 覆盖索引 | 让查询所有字段都在索引中，避免回表 |

**Step 5：使用 pt-query-digest（生产环境推荐）**
```bash
# 分析慢查询日志，找出 Top N 慢查询
pt-query-digest /var/log/mysql/slow.log
```

---

## 题目 3：MySQL InnoDB 与 MyISAM 存储引擎的区别？如何选择？

### 参考答案

| 特性 | InnoDB | MyISAM |
|------|--------|--------|
| **事务支持** | ✅ 支持 ACID 事务 | ❌ 不支持 |
| **锁粒度** | 行级锁 + 间隙锁 | 表级锁 |
| **并发性能** | 高（行级锁） | 低（表级锁） |
| **外键约束** | ✅ 支持 | ❌ 不支持 |
| **崩溃恢复** | ✅ 自动恢复（redo log） | ❌ 需手动修复 |
| **全文索引** | 5.6+ 支持 | ✅ 原生支持（5.5前最优） |
| **存储结构** | .ibd（表空间） | .MYD + .MYI + .frm |
| **COUNT(*)** | 全表扫描（无专门计数器） | 保存了 count 值（快） |
| **索引结构** | B+ Tree（聚簇索引） | B+ Tree（非聚簇索引） |

**InnoDB 适用场景：**
- 需要事务支持（转账、订单）
- 高并发写入场景
- 数据可靠性要求高
- 绝大多数互联网应用

**MyISAM 适用场景：**
- 读多写少的静态表
- 不需要事务的场景
- 只需要全文检索的场景
- 数据量不大且只做分析

**面试追问：什么是聚簇索引？**
> InnoDB 的主键索引就是聚簇索引，叶子节点直接存储完整数据行。MyISAM 是非聚簇索引，叶子节点存储的是数据地址，需要多一次 IO 才能拿到数据。

---

## 题目 4：Redis 缓存与数据库一致性如何保证？有哪些方案？

### 参考答案

这是面试中的综合能力题，考察对分布式系统的理解。

**方案一：Cache Aside（旁路缓存）— 最常用**

```
读操作：
1. 先读缓存，命中则返回
2. 未命中，读数据库
3. 写入缓存，返回数据

写操作：
1. 先写数据库
2. 删除缓存（不是更新！）
```

```python
# Python 伪代码
def get_user(user_id):
    # 读缓存
    user = redis.get(f"user:{user_id}")
    if user:
        return json.loads(user)
    # 读数据库
    user = db.query("SELECT * FROM users WHERE id = %s", user_id)
    # 写缓存（注意过期时间）
    redis.setex(f"user:{user_id}", 3600, json.dumps(user))
    return user

def update_user(user_id, data):
    # 写数据库
    db.execute("UPDATE users SET ... WHERE id = %s", user_id)
    # 删除缓存（不是更新！避免数据不一致）
    redis.delete(f"user:{user_id}")
```

**为什么删除而不是更新？**
> 如果采用更新缓存策略，在并发场景下可能出现：线程A更新数据库→线程B更新数据库→线程B更新缓存→线程A更新缓存，导致缓存中是旧数据。删除缓存则不会有问题。

**方案二：Read Through / Write Through**

由缓存层自己负责读写数据库，应用代码只和缓存交互。实现复杂，一般不常用。

**方案三：延迟双删（解决并发问题）**

```python
def update_user(user_id, data):
    # 1. 先删缓存
    redis.delete(f"user:{user_id}")
    # 2. 写数据库
    db.execute("UPDATE users SET ...")
    # 3. 延迟再删一次（等待并发读完成）
    time.sleep(0.5)
    redis.delete(f"user:{user_id}")
```

**方案四：基于 Canal 监听 MySQL binlog**

MySQL 数据变更后，通过 Canal 推送变更到 MQ，消费端更新 Redis。实现准实时同步，但架构复杂度高。

**缓存常见问题：**

| 问题 | 解决方案 |
|------|---------|
| 缓存穿透 | 布隆过滤器 / 空值缓存 |
| 缓存击穿 | 互斥锁 / 永不过期 + 异步重建 |
| 缓存雪崩 | 过期时间加随机值 / 多级缓存 |

---

## 题目 5：如何设计一个分库分表方案？分片键如何选择？

### 参考答案

**什么时候需要分库分表？**
- 单表数据量超过千万级
- 单库 QPS 达到瓶颈（MySQL 一般 2000-3000 QPS）
- 磁盘 IO 成为瓶颈

**垂直拆分 vs 水平拆分**

```
┌─────────────────────────────────────────────┐
│              垂直拆分（按业务）               │
├──────────┬──────────┬──────────┬────────────┤
│  用户库   │  订单库   │  商品库   │   支付库   │
└──────────┴──────────┴──────────┴────────────┘

┌─────────────────────────────────────────────┐
│              水平拆分（按数据）               │
│  ┌──────────┐  ┌──────────┐  ┌──────────┐  │
│  │ orders_0 │  │ orders_1 │  │ orders_2 │  │
│  │ user%3=0 │  │ user%3=1 │  │ user%3=2 │  │
│  └──────────┘  └──────────┘  └──────────┘  │
└─────────────────────────────────────────────┘
```

**分片键选择原则：**

1. **业务热点字段** — 查询最频繁的条件
2. **数据分布均匀** — 避免数据倾斜和热点
3. **避免跨分片查询** — 尽量让查询落在单一分片

**常见分片算法：**

```sql
-- 1. 取模（Hash）
shard_key = user_id % 4
-- 优点：数据均匀
-- 缺点：扩容困难（取模基数变化要迁移数据）

-- 2. 按范围（Range）
-- user_id 1-250w -> 分片0
-- user_id 250w-500w -> 分片1
-- 优点：扩容简单，新增分片只需修改范围
-- 缺点：容易产生热点（新人集中在最新分片）

-- 3. 一致性 Hash
-- 解决扩容数据迁移问题，业界常用
```

**跨分片查询解决方案：**

| 方案 | 适用场景 | 缺点 |
|------|---------|------|
| 多次查询+内存聚合 | 数据量不大 | 延迟增加 |
| 异构索引表 | 按非分片键查询 | 同步延迟 |
| 分布式搜索（ES） | 复杂查询+聚合 | 架构复杂 |
| 冗余数据 | 热点查询 | 数据一致性难保证 |

**扩容方案（平滑扩容）：**

```
推荐：使用一致性 Hash + 虚拟节点
扩容时，将旧节点数据迁移到新节点，应用层按新规则路由
迁移过程中双写（老库+新库），切换完成后切读
```

---

> 📚 更多面试题欢迎访问：[GitHub - interview-questions](https://github.com/qq286158530/interview-questions)
> 
> ⚠️ 提示：以上答案为知识整理，仅供参考学习，实际面试中请结合自身经验灵活作答。
