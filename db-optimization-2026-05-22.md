# 数据库性能优化面试题

> 📅 更新时间：2026-05-22  
> 📂 分类：数据库 | MySQL / PostgreSQL  
> 🔗 GitHub 仓库：[qq286158530/interview-questions](https://github.com/qq286158530/interview-questions)

---

## 题目一：MySQL 中 InnoDB 索引失效的常见场景有哪些？

### 参考答案

InnoDB 索引失效的常见场景包括：

**1. where 条件中使用函数或运算**
```sql
-- 索引失效
SELECT * FROM users WHERE YEAR(created_at) = 2024;

-- 正确做法：使用索引列直接比较
SELECT * FROM users WHERE created_at >= '2024-01-01' AND created_at < '2025-01-01';
```

**2. where 条件中对索引列进行隐式类型转换**
```sql
-- phone 是 varchar 类型，"="右边用了数字，隐式转换导致索引失效
SELECT * FROM users WHERE phone = 13800138000;

-- 正确写法：使用字符串字面量
SELECT * FROM users WHERE phone = '13800138000';
```

**3. 使用 like 进行模糊匹配时以通配符开头**
```sql
-- 索引失效
SELECT * FROM users WHERE name LIKE '%三';

-- 可以利用索引（但仍需扫描全索引）
SELECT * FROM users WHERE name LIKE '三%';
```

**4. 索引列参与 OR 运算，且 OR 条件中存在非索引列**
```sql
-- 如果 age 有索引，email 没有索引，则索引失效
SELECT * FROM users WHERE age > 25 OR email = 'test@example.com';

-- 建议：拆分查询或确保所有列都有索引
```

**5. 使用 not、in、!=、<> 等导致索引失效**
```sql
-- 索引失效
SELECT * FROM users WHERE status != 1;
SELECT * FROM users WHERE status NOT IN (1, 2);
```

**6. 多列索引未使用最左前缀原则**
```sql
-- 建立索引 (a, b, c)
-- 失效场景
SELECT * FROM users WHERE b = 2;  -- 跳过 a
SELECT * FROM users WHERE c = 3;  -- 跳过 a, b
```

---

## 题目二：如何定位并优化慢查询？

### 参考答案

**第一步：开启慢查询日志**
```sql
-- 查看慢查询配置
SHOW VARIABLES LIKE 'slow_query%';
SHOW VARIABLES LIKE 'long_query_time';

-- 开启慢查询日志（需配置 log_output）
SET GLOBAL slow_query_log = 'ON';
SET GLOBAL long_query_time = 1; -- 设置超过1秒记录
```

**第二步：使用 EXPLAIN 分析查询**
```sql
EXPLAIN SELECT * FROM orders WHERE status = 1;
EXPLAIN ANALYZE SELECT * FROM orders WHERE status = 1;  -- MySQL 8.0+ 含实际执行信息
```

关键关注字段：
- **type**：const > eq_ref > ref > range > index > ALL（尽量避免 ALL）
- **key**：实际使用的索引
- **Extra**：Using filesort / Using temporary（需优化）

**第三步：使用性能分析工具**
- MySQL：`SHOW FULL PROCESSLIST;` 查看当前连接
- MySQL Workbench / pt-query-digest 分析慢查询日志
- PostgreSQL：`pg_stat_statements` 插件、`EXPLAIN ANALYZE`

**第四步：常见优化手段**
1. 添加合适索引
2. 避免 SELECT *，只查询必要字段
3. 使用覆盖索引避免回表
4. 优化子查询，改用 JOIN
5. 分页优化：`WHERE id > last_id LIMIT 100` 代替 `OFFSET`

---

## 题目三：MySQL InnoDB 与 PostgreSQL 的 MVCC 实现机制有何区别？

### 参考答案

**MySQL InnoDB 的 MVCC（基于回滚段）**

- 每行数据隐藏列：`DB_TRX_ID`（事务ID）、`DB_ROLL_PTR`（回滚指针）
- 更新数据时，将旧数据写入 **回滚段（Undo Log）**，新数据的回滚指针指向旧版本
- 读取数据时，根据事务 Read View 判断可见性：
  - 已提交事务修改的数据可见
  - 未提交或活跃事务修改的数据不可见
- 快照读（SELECT）不加锁，通过 MVCC 实现可重复读

**PostgreSQL 的 MVCC（基于多版本元组）**

- 每次更新不修改原行，而是创建新版本元组（称为 **tuple**）
- 老版本通过 `xmin`（创建事务ID）和 `xmax`（删除事务ID）管理生命周期
- 依赖 **VACUUM** 进程清理过期元组
- 不依赖 Undo Log，读取时通过 `xmin/xmax` + 事务状态判断可见性

**核心区别对比**

| 特性 | InnoDB | PostgreSQL |
|------|--------|------------|
| 版本存储 | Undo Log（回滚段） | 行版本（多版本 tuple） |
| 清理机制 | InnoDB 自动清理 | 需 VACUUM 手动/自动清理 |
| 事务ID | 递增整数 | 递增整数 |
| 垃圾回收 | 后台线程 | VACUUM 进程 |
| 索引结构 | 聚簇索引 | 堆表 + 索引分离 |

---

## 题目四：如何设计分库分表方案？跨节点查询如何处理？

### 参考答案

**1. 分库分表策略**

- **垂直拆分**：按业务模块拆分，如订单库、用户库
- **水平拆分**：按数据特征拆分，如按 user_id 取模、按时间分区

常用中间件：ShardingSphere、MyCAT、Vitess（Kubernetes）

**2. 分片键选择原则**
- 选择查询条件中最常用的字段（如 user_id）
- 避免数据倾斜（热点数据集中在某一节点）
- 跨节点事务场景使用分布式事务（如 Seata AT/TCC 模式）

**3. 跨节点查询处理方案**

**方案一：异构索引 + 双写**
```
用户库按 user_id 分片，同时将数据同步到 ES
查询时先查 ES 获取目标 shard，再路由查询
```

**方案二：广播查询**
```
将查询下发到所有节点，并行查询后合并结果
适用于低频跨节点查询
```

**方案三：基因法分片**
```
分片键 = hash(user_id) % n + shard_id
保证同一用户数据在同一节点
```

**4. 分布式 ID 解决方案**
- 雪花算法（Snowflake）
- 数据库自增步长
- Leaf（美团分布式ID生成器）

---

## 题目五：Redis 缓存与数据库一致性问题如何解决？

### 参考答案

**缓存与数据库不一致的根源**：更新操作顺序问题（先更新数据库 vs 先删缓存）

**方案一：Cache Aside（旁路缓存）- 最常用**
```
读：缓存命中 → 返回；未命中 → 查DB → 写缓存 → 返回
写：更新DB → 删除缓存
```
问题：并发场景下可能短暂不一致，实际影响小

**方案二：Read Through**
```
缓存负责查询DB并写入缓存，对应用透明
```

**方案三：Write Through**
```
写操作同时更新缓存和DB（性能差，较少用）
```

**方案四：延迟双删（解决并发问题）**
```
1. 删除缓存
2. 更新数据库
3. 延迟 N 毫秒再删除缓存（异步）
```

**方案五：canal 订阅 binlog 异步更新**
```
应用 ← canal ← MySQL binlog ← 缓存
MySQL 更新 → 解析 binlog → 更新 Redis
```

**最佳实践建议**
- 读多写少场景：Cache Aside + TTL 过期
- 写多读少场景：直接操作DB，不强依赖缓存
- 强一致性场景：分布式锁 + 延迟双删，或放弃缓存走DB
- 最终一致性场景：消息队列异步同步

---

> 📌 以上题目均为数据库性能优化领域经典面试题，答案仅供参考学习。
> 💡 建议结合实际项目经验深入理解，而非死记硬背答案。