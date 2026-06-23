# 数据库性能优化面试题

> 📅 日期：2026-06-23
> 📚 来源：动力节点、小林coding、腾讯云开发者、CSDN 等

---

## 题目一：MySQL 如何分析性能？慢查询优化的基本步骤是什么？

### 参考答案

#### 一、MySQL 性能分析手段

**1. 慢查询日志**

MySQL 的慢查询日志用来记录响应时间超过阈值的 SQL 语句。默认阈值 `long_query_time = 10` 秒，需手动开启：

```sql
[mysqld]
slow_query_log = ON
slow_query_log_file = /var/lib/mysql/hostname-slow.log
long_query_time = 3
```

使用 `mysqldumpslow` 工具分析日志。

**2. EXPLAIN 执行计划**

使用 `EXPLAIN` 关键字模拟优化器执行 SQL，分析查询瓶颈：

```sql
EXPLAIN SELECT * FROM orders WHERE user_id = 100;
```

重点关注字段：`type`、`key`、`rows`、`filtered`、`Extra`。

**3. Show Profile 分析**

分析当前会话中 SQL 执行的资源消耗情况，默认保存最近 15 次运行结果：

```sql
SET profiling = 1;
SHOW VARIABLES LIKE "%pro%";
SHOW profiles;
SHOW profile FOR QUERY 1;
```

**4. Show 命令**

使用 `SHOW STATUS` / `SHOW GLOBAL STATUS` 查看系统状态和变量。

---

#### 二、慢查询优化基本步骤

1. **先运行看看是否真的很慢**，注意设置 `SQL_NO_CACHE`
2. **where 条件单表查，锁定最小返回记录表** — 把 where 条件应用到返回记录数最小的表开始查起
3. **explain 查看执行计划**，确认是否与预期一致
4. **order by limit 优先** — 排序的表优先查询
5. **了解业务方使用场景**
6. **加索引时参照建索引的几大原则**（最左前缀、覆盖索引、避免回表等）
7. **观察结果，不符合预期继续从 0 分析**

---

> 📎 来源：[动力节点 - MySQL性能优化面试题](https://www.bjpowernode.com/mysqlmst/xnyh.html)

---

## 题目二：为什么 MySQL InnoDB 选择 B+Tree 而非 B 树、二叉树或 Hash 作为索引结构？

### 参考答案

#### B+Tree vs B 树

- B+Tree **只在叶子节点存储数据**，B 树非叶子节点也存储数据，因此 B+Tree 单个节点数据量更小，相同磁盘 I/O 下能查询更多节点
- B+Tree 叶子节点采用**双链表连接**，适合范围查询；B 树无法做到这一点

#### B+Tree vs 二叉树

- B+Tree 搜索复杂度为 `O(logdN)`，d 为节点最大子节点个数（通常 d > 100）
- 即使数据达千万级，B+Tree 高度仍维持在 3~4 层，只需 3~4 次磁盘 I/O
- 二叉树搜索复杂度为 `O(logN)`，磁盘 I/O 次数更多

#### B+Tree vs Hash

- Hash 等值查询效率高，复杂度 O(1)
- **Hash 不适合范围查询**，而 B+Tree 索引适用范围更广

#### B+Tree 的核心优势总结

| 特性 | B+Tree | B 树 | 二叉树 | Hash |
|------|--------|------|--------|------|
| 范围查询 | ✅ | ❌ | ❌ | ❌ |
| 等值查询 | O(logdN) | O(logdN) | O(logN) | O(1) |
| 磁盘 I/O 次数 | 3~4 次（千万数据） | 较多 | 很多 | 1 次 |
| 节点数据密度 | 高 | 低 | 低 | 高 |

> B+Tree 千万级数据只需 3~4 层高度，查询最多 3~4 次磁盘 I/O，是平衡范围查询与等值查询的最优选择。

---

> 📎 来源：[小林coding - 索引常见面试题](https://xiaolincoding.com/mysql/index/index_interview.html)

---

## 题目三：MySQL 索引失效有哪些常见场景？如何避免？

### 参考答案

#### 常见的索引失效场景

**1. 在索引列上使用函数或运算**

```sql
-- 索引失效 ❌
SELECT * FROM orders WHERE YEAR(create_time) = 2026;

-- 正确做法 ✅
SELECT * FROM orders WHERE create_time >= '2026-01-01' AND create_time < '2027-01-01';
```

**2. 使用 LIKE 左前缀匹配**

```sql
-- 索引失效 ❌ (以 % 开头)
SELECT * FROM orders WHERE name LIKE '%王';

-- 索引有效 ✅ (以固定值开头)
SELECT * FROM orders WHERE name LIKE '王%';
```

**3. 类型隐式转换**

```sql
-- user_id 为 varchar 类型，传入数字会触发隐式转换，索引失效 ❌
SELECT * FROM orders WHERE user_id = 100;  -- 100 为整型

-- 正确做法 ✅
SELECT * FROM orders WHERE user_id = '100';
```

**4. 使用 OR 连接多个条件**

```sql
-- 部分条件无索引，导致全表扫描 ❌
SELECT * FROM orders WHERE user_id = 100 OR status = 1;

-- 改用 UNION ✅
(SELECT * FROM orders WHERE user_id = 100)
UNION ALL
(SELECT * FROM orders WHERE status = 1 AND user_id IS NOT NULL);
```

**5. 范围查询（>、<、BETWEEN）中断最左前缀**

```sql
-- 联合索引 (a, b, c)，a 和 b 能用索引，c 无法使用 ❌
SELECT * FROM orders WHERE a = 1 AND b > 10 AND c = 1;
```

**6. IS NULL / IS NOT NULL**

```sql
-- 可能导致索引失效 ❌
SELECT * FROM orders WHERE status IS NOT NULL;
```

**7. 使用 SELECT * 而非覆盖索引**

```sql
-- 需要回表查询全部数据，效率低 ❌
SELECT * FROM orders WHERE user_id = 100;

-- 只查需要的列，利用覆盖索引 ✅
SELECT user_id, status FROM orders WHERE user_id = 100;
```

#### 避免索引失效的建议

- ✅ 查询时尽量使用索引列，避免函数和运算
- ✅ 范围查询尽量放在索引最后
- ✅ 用小结果集驱动大结果集
- ✅ 使用 EXPLAIN 检查执行计划
- ✅ 优先使用覆盖索引，减少回表

---

> 📎 来源：[腾讯云开发者 - MySQL 性能优化的理解](https://cloud.tencent.com/developer/article/2478179) | [CSDN - MySQL面试题-性能优化](https://blog.csdn.net/qq_33129875/article/details/129396343)

---

## 题目四：百万级数据如何删除？如何优化大批量数据操作？

### 参考答案

#### 问题背景

删除百万级数据时，索引维护会产生额外成本——增/删/改操作都会触发索引文件更新，消耗额外 IO，降低执行效率。**删除速度与索引数量成正比**。

#### 正确做法：分批删除 + 先删索引

以删除 100 万条数据为例：

```
Step 1: 先删除索引（约 3 分钟）
Step 2: 删除无用数据（约 1~2 分钟，数据量少时很快）
Step 3: 重新创建索引（约 10 分钟，数据已较少）
```

#### 分批删除实现

```sql
-- 每批删除 1000 条，循环执行直到删完
DELETE FROM orders WHERE status = 0 LIMIT 1000;

-- 使用主键或索引列作为删除条件，提升效率
DELETE FROM orders WHERE id > last_id AND status = 0 ORDER BY id LIMIT 1000;
```

#### 大批量数据插入优化

```sql
-- 批量插入，减少 IO 次数
INSERT INTO orders (user_id, amount) VALUES
(1, 100), (2, 200), (3, 300), ...;  -- 一次插入多条

-- 关闭自动提交，提升批量插入性能
SET autocommit = 0;
-- 执行批量 INSERT
COMMIT;
```

#### 大批量数据更新优化

```sql
-- 分批更新，避免长时间锁表
UPDATE orders SET status = 1 WHERE id BETWEEN 1 AND 1000;
-- 循环处理后续批次
```

#### 核心原则

| 操作 | 优化策略 |
|------|----------|
| 删除 | 先删索引 → 再删数据 → 重建索引 |
| 插入 | 批量插入 + 关闭自动提交 |
| 更新 | 分批更新 + LIMIT 限制每批数量 |
| 迁移 | 使用 `INSERT ... SELECT` 而非逐行 |

---

> 📎 来源：[动力节点 - MySQL性能优化面试题](https://www.bjpowernode.com/mysqlmst/xnyh.html)

---

## 题目五：MySQL 性能优化的四大层面是什么？各有何具体手段？

### 参考答案

MySQL 性能优化可以分为以下 **4 大层面**：

#### 1. 硬件和操作系统层面

| 因素 | 说明 |
|------|------|
| CPU | 多核处理能力 |
| 内存 | `innodb_buffer_pool_size` 缓存池大小 |
| 磁盘 | SSD 替代 HDD，RAID 配置 |
| 网络 | 带宽和网络配置 |

**核心原则**：根据服务承载的体量提出合理指标，避免资源浪费。

#### 2. 架构设计层面

| 方案 | 适用场景 |
|------|----------|
| 主从集群 | 保证高可用，单点故障时自动切换 |
| 读写分离 | 读多写少场景，避免读写冲突 |
| 分库分表 | 降低单服务器 IO 压力，减少单表数据量 |
| 引入缓存 | Redis/MongoDB 等缓存热点数据，减轻 MySQL 压力 |

```sql
-- 示例：读写分离后，从库处理查询
SELECT * FROM orders WHERE user_id = 100;  -- 走从库
INSERT INTO orders ...;                       -- 走主库
```

#### 3. MySQL 程序配置优化（my.cnf）

```ini
[mysqld]
# 最大连接数（默认 151，按需调整）
max_connections = 1000

# InnoDB 缓存池大小（建议为服务器物理内存的 70%~80%）
innodb_buffer_pool_size = 12G

# 开启 binlog（用于主从复制和数据恢复）
log_bin = mysql-bin

# 慢查询日志阈值
slow_query_log = ON
long_query_time = 2
```

**注意**：
- 全局参数修改对已存在会话不生效
- 会话参数随会话销毁而失效
- 建议将全局配置写入配置文件，重启后仍生效

#### 4. SQL 优化（开发侧核心）

**第一步：慢 SQL 定位**
```sql
-- 开启慢查询日志
SET slow_query_log = 'ON';
SET global slow_query_log_file = '/var/lib/mysql/slow.log';
```

**第二步：执行计划分析**
```sql
EXPLAIN SELECT * FROM orders WHERE user_id = 100;
-- 重点关注：type、key、rows、filtered、Extra
```

**第三步：Profile 详细分析**
```sql
SET profiling = 1;
SHOW profiles;
SHOW profile FOR QUERY 1;  -- 查看 CPU、IO、内存开销
```

#### SQL 优化核心规则汇总

- ✅ SQL 查询基于索引扫描
- ✅ 避免索引列上使用函数或运算
- ✅ LIKE `%` 尽量放在右边
- ✅ 联合索引从左到右命中越多越好
- ✅ 优先使用索引完成排序，避免文件排序
- ✅ 只查询需要的列，少用 `SELECT *`
- ✅ 永远用小结果集驱动大结果集

---

> 📎 来源：[腾讯云开发者 - MySQL 性能优化的理解](https://cloud.tencent.com/developer/article/2478179) | [JavaGuide - MySQL高性能优化规范](https://javaguide.cn/database/mysql/mysql-high-performance-optimization-specification-recommendations.html)

---

*本文件由 AI 自动生成，每日更新。*
*GitHub 仓库：https://github.com/qq286158530/interview-questions*