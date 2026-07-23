# 数据库性能优化面试题

> 📅 2026-07-23 | 精选5道高频面试题，涵盖索引、查询、存储引擎、缓存等核心知识点

---

## 题目一：MySQL 如何定位并优化慢 SQL？（索引优化）

### 🔍 问题描述

当一条 SQL 查询很慢时，你是如何分析的？又如何优化？

### ✅ 详细答案

**第一步：开启慢查询日志，定位慢 SQL**

```sql
-- 查看慢查询配置
SHOW VARIABLES LIKE 'slow_query_log%';
SHOW VARIABLES LIKE 'long_query_time%';

-- 开启慢查询（临时）
SET GLOBAL slow_query_log = 'ON';
SET GLOBAL long_query_time = 1; -- 超过1秒记录

-- 分析慢查询日志
mysqldumpslow -s t -t 10 /var/lib/mysql/slow.log
```

**第二步：使用 EXPLAIN 分析执行计划**

```sql
EXPLAIN SELECT * FROM orders WHERE user_id = 100 AND status = 'paid';
```

重点关注：
- `type`：最好达到 `ref`/`range`，避免 `ALL`（全表扫描）
- `key`：实际使用的索引
- `rows`：扫描行数，越少越好
- `Extra`：`Using filesort` / `Using temporary` 是危险信号

**第三步：优化手段**

| 优化手段 | 适用场景 |
|---------|---------|
| 添加合适索引 | `WHERE`、`JOIN`、`ORDER BY` 字段建立索引 |
| 覆盖索引 | 查询字段都在索引中，避免回表 |
| 索引下推 (ICP) | 减少回表次数 |
| 联合索引最左前缀 | 合理设计索引顺序 |
| 前缀索引 | 字符串字段可使用前缀减少索引体积 |

**经典案例：**

```sql
-- 原始慢查询
SELECT * FROM orders WHERE user_id = 100 AND status = 'paid' ORDER BY created_at DESC;

-- 分析发现 type=ALL，全表扫描
-- 优化：建立联合索引
ALTER TABLE orders ADD INDEX idx_user_status_time (user_id, status, created_at DESC);

-- 优化后查询直接走索引，rows 从 10万 降到几十
```

### 📚 参考来源
- 小林Coding：https://xiaolincoding.com/mysql/
- MySQL 官方文档：https://dev.mysql.com/doc/refman/8.0/en/optimization.html

---

## 题目二：数据库索引失效的常见原因有哪些？

### 🔍 问题描述

明明建了索引，为什么查询还是很慢？索引什么时候会失效？

### ✅ 详细答案

**常见索引失效场景：**

**1. 违反最左前缀原则**
```sql
-- 联合索引 (a, b, c)
-- ✅ 生效：WHERE a=1 / WHERE a=1 AND b=2
-- ❌ 失效：WHERE b=2 / WHERE c=3
```

**2. 使用了函数或运算**
```sql
-- ❌ 索引失效
SELECT * FROM users WHERE YEAR(created_at) = 2024;

-- ✅ 正确写法
SELECT * FROM users WHERE created_at >= '2024-01-01' AND created_at < '2025-01-01';
```

**3. 类型转换**
```sql
-- phone 是 varchar 类型
-- ❌ 隐式类型转换，索引失效
SELECT * FROM users WHERE phone = 13800138000;

-- ✅ 正确写法
SELECT * FROM users WHERE phone = '13800138000';
```

**4. LIKE 开头使用通配符**
```sql
-- ❌ '%xxx' 开头导致全表扫描
SELECT * FROM products WHERE name LIKE '%手机';

-- ✅ 可以优化为覆盖索引，或使用全文索引
SELECT * FROM products WHERE name LIKE '手机%'; -- 前缀匹配可以走索引
```

**5. OR 连接条件**
```sql
-- ❌ OR 导致索引失效（除非两边都有单独索引）
SELECT * FROM users WHERE name = '张三' OR email = 'zhangsan@example.com';

-- ✅ 使用 UNION 改写
SELECT * FROM users WHERE name = '张三'
UNION ALL
SELECT * FROM users WHERE email = 'zhangsan@example.com' AND name != '张三';
```

**6. 使用 NOT / != / <>**
```sql
-- ❌ 索引通常不支持
SELECT * FROM orders WHERE status != 'paid';

-- ✅ 可考虑用 IN 改写
SELECT * FROM orders WHERE status IN ('pending', 'shipped', 'delivered');
```

**7. 排序时违反索引顺序**
```sql
-- 索引 (a, b)，但 ORDER BY b, a
-- ❌ 索引失效，需要额外排序
```

### 📚 参考来源
- 美团技术团队：https://tech.meituan.com/
- 掘金 MySQL 专栏：https://juejin.cn/post/6844903918901772301

---

## 题目三：MySQL InnoDB 与 MyISAM 存储引擎的区别？如何选型？

### 🔍 问题描述

InnoDB 和 MyISAM 有什么区别？实际项目中如何选择？

### ✅ 详细答案

**核心区别对比：**

| 特性 | InnoDB | MyISAM |
|------|--------|--------|
| 事务支持 | ✅ 支持 ACID 事务 | ❌ 不支持 |
| 行锁 | ✅ 支持行级锁 | ❌ 只支持表级锁 |
| 外键 | ✅ 支持 | ❌ 不支持 |
| 崩溃恢复 | ✅ 自动恢复 (redo log) | ❌ 损坏率更高 |
| 全文索引 | ✅ 5.6+ 支持 | ✅ 原生支持 |
| 存储结构 | 聚簇索引（数据在B+Tree叶子节点） | 非聚簇索引（索引和数据分开存储） |
| 并发能力 | 高（行锁+MVCC） | 低（表锁） |
| 适用场景 | 高并发、事务需求 | 读多写少、不需要事务 |

**InnoDB 为什么是默认存储引擎？**

1. **事务安全**：提交/回滚保证数据一致性
2. **行级锁**：高并发下性能更好
3. **崩溃恢复**：通过 redo log 自动恢复
4. **外键约束**：保证参照完整性

**选型建议：**

```
✅ 选 InnoDB 的场景：
- 电商订单、支付等核心业务（事务刚需）
- 并发写入多的系统
- 需要崩溃快速恢复的生产环境
- 大部分在线业务系统

✅ 选 MyISAM 的场景：
- 日志表、审计表（只写少读）
- 全文检索需求（5.6之前）
- 数据仓库的辅助表
- 静态/历史数据查询
```

### 📚 参考来源
- MySQL 官方文档：https://dev.mysql.com/doc/refman/8.0/en/innodb-introduction.html
- 高性能 MySQL：https://www.oreilly.com/library/view/high-performance-mysql/9781492080503/

---

## 题目四：如何设计分库分表方案？跨节点查询如何解决？

### 🔍 问题描述

单表数据量过亿时，如何做分库分表？跨节点查询怎么处理？

### ✅ 详细答案

**分库分表策略：**

**1. 垂直拆分（按业务拆分）**
```
用户库（users, addresses, accounts）
订单库（orders, order_items）
商品库（products, categories, inventory）
```
优点：减少单库负载，业务解耦
缺点：跨库 JOIN 需要额外处理

**2. 水平拆分（按数据拆分）**

| 拆分方式 | 原理 | 适用场景 |
|---------|------|---------|
| 按 ID 哈希 | `hash(id) % N` 分散到 N 表 | 用户、订单等无时间顺序的数据 |
| 按时间 | 按月/季度分表 | 日志、流水等有时序特征的数据 |
| 按地区 | 按省份/城市分库 | 区域性业务 |

**经典按哈希分表示例：**
```sql
-- 假设拆成 8 张表：orders_0 ~ orders_7
-- 路由算法：table = hash(order_id) % 8

-- 优点：数据分布均匀
-- 缺点：扩容困难（需要迁移数据）
```

**3. 扩容方案（一致性哈希）**
```
减少扩容时数据迁移量，使用一致性哈希环
节点扩缩时只需迁移 1/N 的数据
```

**跨节点查询解决方案：**

**方案一：异构表（汇总表）**
```sql
-- 定时任务将各分表数据聚合到汇总表
-- 供查询使用（有一定延迟）
CREATE TABLE orders_summary (
    date DATE,
    city VARCHAR(20),
    total_amount DECIMAL(12,2),
    order_count INT
);
```

**方案二：ES/Hive 离线查询**
```
写入 MySQL 同时，异步写入 Elasticsearch
复杂查询走 ES，全文检索走 ES
```

**方案三：分布式数据库中间件**
```
ShardingSphere / MyCat 透明支持分库分表
应用层无感知，自动路由
```

### 📚 参考来源
- ShardingSphere 官方文档：https://shardingsphere.apache.org/
- 教你在58到家如何做数据库拆分：https://mp.weixin.qq.com/s/K新一代数据库技术

---

## 题目五：Redis 缓存与数据库一致性如何保证？

### 🔍 问题描述

缓存和数据库双写时如何保证一致性？缓存雪崩、穿透、击穿如何处理？

### ✅ 详细答案

**缓存读写策略：**

**Cache Aside（最常用）**
```
读：缓存命中 → 直接返回
    缓存未命中 → 查DB → 写入缓存 → 返回

写：更新DB → 删除缓存（不是更新缓存）
```

为什么删除而不是更新？
- 避免并发时缓存与DB不一致
- 删除代价更低，下次读时自然填充

**Read/Write Through**
```
应用只操作缓存，缓存负责同步DB
读写性能好，但实现复杂
```

**Write Behind**
```
先写缓存，异步批量写DB
性能最高，但有数据丢失风险
```

**如何保证最终一致性？**

```python
# 延迟双删策略
def write(key, value):
    db.write(key, value)      # 1. 先更新数据库
    cache.delete(key)         # 2. 删除缓存
    time.sleep(0.5)           # 3. 延迟500ms
    cache.delete(key)         # 4. 再删除一次（兜底）

# 订阅 binlog 异步更新
# 使用 Canal 监听 MySQL binlog，变更后同步更新 Redis
```

**缓存三大问题及解决方案：**

**1. 缓存雪崩**
```
问题：大量缓存同时过期，请求打到DB
解决：
  - 过期时间加随机值：TTL + random(0, 300)
  - 永不过期 + 异步重建
  - 多级缓存架构
```

**2. 缓存穿透**
```
问题：查询不存在的数据，每次都打到DB
解决：
  - 布隆过滤器（快速判断是否存在）
  - 空值缓存（设置短TTL）
  - 参数校验拦截
```

**3. 缓存击穿**
```
问题：热点key过期瞬间，大量请求击穿到DB
解决：
  - 互斥锁（只有一个请求去查DB）
  - 热点数据永不过期
  - 逻辑过期 + 异步构建
```

```python
# 互斥锁实现
def get_with_lock(key):
    cache_val = cache.get(key)
    if cache_val:
        return cache_val
    
    # 获取锁，只允许一个请求查DB
    lock = redis.lock(f"lock:{key}", timeout=3)
    if lock.acquire():
        try:
            # 双重检查
            cache_val = cache.get(key)
            if cache_val:
                return cache_val
            db_val = db.query(key)
            cache.setex(key, 3600, db_val)
            return db_val
        finally:
            lock.release()
    else:
        time.sleep(0.1)
        return get_with_lock(key)  # 重试
```

### 📚 参考来源
- Redis 设计与实现：https://redisbook.com/
- 美团技术团队缓存设计：https://tech.meituan.com/

---

## 📖 更多资源

- GitHub 面试题仓库：https://github.com/qq286158530/interview-questions
- 欢迎 Star & PR 投稿更多面试题！

---

*本文件由 AI 自动生成，每日定时更新*
