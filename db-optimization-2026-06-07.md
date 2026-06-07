# 数据库性能优化面试题集锦

>📅 日期：2026-06-07  
> 🏷️ 标签：MySQL | PostgreSQL | 数据库优化 | 面试题

---

## 题目一：MySQL 中 B+树索引的结构特点是什么？为什么比 B-树更适合数据库？

### 参考答案

**B+树 vs B-树核心区别：**

| 特性 | B-树 | B+树 |
|------|------|------|
| **数据存储** | 所有节点都存储数据 | 只有叶子节点存储数据 |
| **叶子节点** | 无链表连接 | 叶子节点通过双向链表连接 |
| **查询稳定性** | 最好O(1)，最差O(log n) | 所有查询都是O(log n) |
| **范围查询** | 需要中序遍历 | 叶子链表直接遍历 |

**B+树更适合数据库的原因：**

1. **查询性能稳定**：所有数据都在叶子节点，查找路径长度一致
2. **范围查询高效**：叶子链表直接遍历，避免中序回溯
3. **磁盘读写友好**：
   - 非叶子节点只存索引，节点更小
   - 相同数据量下树高更低，IO次数更少
   - 节点大小通常等于页大小（16KB），符合磁盘预读

```sql
-- InnoDB 页大小 16KB 示例
-- 假设索引键大小 8字节，非叶子节点可存储 16KB/8 ≈ 2000 个键
-- 高度为 3 的 B+树可支持：2000 × 2000 × 2000 = 80亿条记录
```

**面试加分点：**
- InnoDB 聚簇索引，叶子节点存储完整行数据
- 非主键索引叶子节点存储主键值，需要回表
- 覆盖索引可避免回表，提升查询性能

> 📚 来源：https://www.mysql.com/products/innodb.html

---

## 题目二：什么是数据库慢查询？如何定位和分析慢查询？

### 参考答案

**慢查询定义：**
- MySQL：`long_query_time` 参数控制，默认 10 秒
- PostgreSQL：`log_min_duration_statement` 参数控制

**定位慢查询的方法：**

**1. MySQL 慢查询日志**
```sql
-- 开启慢查询日志
SET GLOBAL slow_query_log = 'ON';
SET GLOBAL slow_query_log_file = '/var/log/mysql/slow.log';
SET GLOBAL long_query_time = 2;  -- 超过2秒记录

-- 查看慢查询日志
mysqldumpslow -s t -t 10 /var/log/mysql/slow.log
```

**2. 使用 EXPLAIN 分析**
```sql
EXPLAIN ANALYZE SELECT * FROM orders WHERE status = 1;
-- EXPLAIN ANALYZE 会实际执行并显示真实耗时
```

**3. 性能模式（Performance Schema）**
```sql
-- 启用相关监控
UPDATE performance_schema.setup_instruments
SET enabled = 'YES' WHERE name LIKE '%statement/%';

-- 查询最慢的SQL
SELECT SQL_TEXT, SUM(TIMER_WAIT)/1000000000000 AS total_time_sec
FROM performance_schema.events_statements_summary_by_digest
ORDER BY total_time_sec DESC LIMIT 5;
```

**分析维度：**
- **type**：是否为全表扫描（ALL）
- **key**：是否使用了索引
- **rows**：扫描行数是否过大
- **Extra**：是否有 Using filesort、Using temporary 等警告

**优化步骤：**
1. 开启慢查询记录
2. 抓取 Top SQL
3. EXPLAIN 分析执行计划
4. 针对问题建立索引或重写SQL
5. 验证优化效果

> 📚 来源：https://dev.mysql.com/doc/refman/8.0/en/slow-query-log.html

---

## 题目三：PostgreSQL 的 MVCC 机制是什么？如何减少锁冲突提升并发性能？

### 参考答案

**MVCC（多版本并发控制）核心原理：**

PostgreSQL 通过「可见性判断」实现并发，每个事务看到的数据是某个时间点的快照：

```
事务A (txid=100) 写入: UPDATE SET balance=200 WHERE id=1
事务B (txid=101) 读取: SELECT balance FROM users WHERE id=1
                                    ↓
                          查看元组头部 xmin=100
                          判断 txid=101 是否可见 t100 的修改
                          (取决于事务100是否已提交)
```

**关键概念：**
- **xmin/xmax**：元组头部存储创建事务ID和删除事务ID
- **t_xmin/t_xmax**：PostgreSQL 系统列
- **CommandId**：命令序号
- **Snapshot**：可见性判断的数据结构

**减少锁冲突的策略：**

**1. 读已提交（默认）vs 可重复读**
```sql
-- 读已提交：每个语句看到已提交的数据
SET TRANSACTION ISOLATION LEVEL READ COMMITTED;

-- 可重复读：整个事务看到相同的快照
SET TRANSACTION ISOLATION LEVEL REPEATABLE READ;
```

**2. 乐观锁减少写冲突**
```sql
-- 不使用 SELECT FOR UPDATE，改为版本号判断
UPDATE orders SET status=2, version=version+1
WHERE id=1 AND version=5;
-- 失败则重试业务逻辑
```

**3. 避免长事务**
```sql
-- 错误示例：大事务
BEGIN;
SELECT * FROM large_table; -- 快照维持
-- 业务处理... (可能需要几分钟)
COMMIT;

-- 正确示例：分批处理
FOR i IN 1..100 LOOP
    UPDATE orders SET status=2 WHERE id = i AND status=1;
    COMMIT; -- 每批提交
END LOOP;
```

**4. 使用 Advisory Lock 业务锁**
```sql
SELECT pg_advisory_lock(12345);  -- 申请业务锁
-- 执行业务逻辑
SELECT pg_advisory_unlock(12345); -- 释放锁
```

> 📚 来源：https://www.postgresql.org/docs/current/mvcc.html

---

## 题目四：如何优化大表的 DDL 操作（添加字段/索引）？有哪些非阻塞方案？

### 参考答案

**DDL 操作的风险：**
- MySQL 5.6 之前：DDL 会锁表，导致服务不可用
- 大表 DDL 可能需要数小时，期间数据库卡死

**MySQL 优化方案：**

**方案一：pt-online-schema-change（推荐）**
```bash
pt-online-schema-change \
  --alter "ADD INDEX idx_status_time(status, create_time)" \
  --execute D=test,t=orders
```
- 工作原理：创建新表 → 批量迁移数据 → 替换原表
- 允许期间正常读写，几乎无停机时间

**方案二：gh-ost（GitHub 出品）**
```bash
gh-ost --database=test --table=orders \
  --alter="ADD INDEX idx_status_time(status, create_time)" \
  --execute
```
- 轻量级，基于 binlog 实时同步
- 可暂停、可回滚

**方案三：MySQL 8.0 即时 DDL**
```sql
-- MySQL 8.0 支持 ALGORITHM=INSTANT
ALTER TABLE orders ADD COLUMN remark VARCHAR(255), ALGORITHM=INSTANT;
-- 仅修改元数据，不动数据页，快速完成
```

**PostgreSQL 优化方案：**

**方案一：CONCURRENTLY 选项**
```sql
-- 创建索引不锁表（但需要较长时间）
CREATE INDEX CONCURRENTLY idx_orders_status ON orders(status);

-- 注意：CREATE INDEX CONCURRENTLY 不支持在事务内执行
-- 需要使用 BEGIN...COMMIT
```

**方案二：使用 pg_repack 重建表**
```bash
pg_repack -d test -t orders --no-kill-backend
```
- 在线重建表，不锁表
- 支持压缩膨胀的表

**方案三：分区表降复杂度**
```sql
-- 将大表转为分区表，单个分区 DDL 更快
CREATE TABLE orders_part (
    id BIGINT, ...
) PARTITION BY RANGE (create_time);

-- 对单个分区操作更快
ALTER TABLE orders_part DETACH PARTITION orders_2024;
```

> 📚 来源：https://www.percona.com/doc/percona-toolkit/LATEST/pt-online-schema-change.html

---

## 题目五：数据库缓存策略如何设计？Redis 与数据库双写一致性如何保证？

### 参考答案

**缓存策略设计原则：**

**1. Cache Aside（旁路缓存）— 最常用**
```python
def get_user(user_id):
    # 先查缓存
    user = redis.get(f"user:{user_id}")
    if user:
        return json.loads(user)
    
    # 缓存不存在，查数据库
    user = db.query("SELECT * FROM users WHERE id = ?", user_id)
    
    # 写入缓存，TTL 1小时
    redis.setex(f"user:{user_id}", 3600, json.dumps(user))
    return user

def update_user(user_id, data):
    # 先更新数据库
    db.execute("UPDATE users SET ... WHERE id = ?", user_id)
    
    # 再删除缓存（不是更新）
    redis.delete(f"user:{user_id}")
```

**2. 缓存读写策略对比**

| 策略 | 优点 | 缺点 | 适用场景 |
|------|------|------|----------|
| **Cache Aside** | 一致性好，逻辑清晰 | 首次请求慢 | 读多写少 |
| **Read Through** | 编码简单 | 缓存依赖存储 | 读多 |
| **Write Through** | 数据确定写入 | 更新慢 | 写多 |
| **Write Behind** | 写入快 | 可能丢数据 | 日志类 |

**双写一致性保证：**

**方案一：延迟双删**
```python
def update_user(user_id, data):
    db.execute("UPDATE users SET ... WHERE id = ?", user_id)
    
    # 删除缓存
    redis.delete(f"user:{user_id}")
    
    # 延迟再删（应对并发）
    time.sleep(0.5)
    redis.delete(f"user:{user_id}")
```

**方案二：订阅 Binlog 同步**
```
数据库 → Binlog → Canal/Kafka → 消费 → 更新Redis
```
- 优点：彻底解耦，延迟低
- 缺点：架构复杂

**方案三：分布式锁**
```python
def update_user(user_id, data):
    lock = redis.lock(f"user:lock:{user_id}", timeout=10)
    if not lock.acquire():
        raise Exception("系统繁忙")
    
    try:
        db.execute("UPDATE users SET ... WHERE id = ?", user_id)
        redis.delete(f"user:{user_id}")
    finally:
        lock.release()
```

**缓存问题应对：**

| 问题 | 解决方案 |
|------|----------|
| **缓存穿透** | 空值缓存 或 布隆过滤器 |
| **缓存击穿** | 互斥锁 或 热点数据永不过期 |
| **缓存雪崩** | TTL随机化 或 多级缓存 |

> 📚 来源：https://redis.io/topics/lru

---

## 📖 更多学习资源

- [MySQL 8.0 Reference Manual - Optimization](https://dev.mysql.com/doc/refman/8.0/en/optimization.html)
- [PostgreSQL Performance Tips](https://www.postgresql.org/docs/current/performance-tips.html)

---

*本面试题集由 AI 自动整理生成，如有问题欢迎指正*