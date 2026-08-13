# 数据库性能优化面试题 · 2026-08-13

> 本系列由 [qq286158530/interview-questions](https://github.com/qq286158530/interview-questions) 收集整理

---

## 题目一：MySQL InnoDB 索引失效的常见场景有哪些？如何避免？

### 参考答案

InnoDB 索引失效的常见场景：

1. **使用函数或运算**
   ```sql
   -- 索引失效
   SELECT * FROM orders WHERE YEAR(create_time) = 2026;
   -- 正确写法：利用索引范围查询
   SELECT * FROM orders WHERE create_time >= '2026-01-01' AND create_time < '2027-01-01';
   ```

2. **隐式类型转换**
   ```sql
   -- phone 为 varchar，传入数字导致全表扫描
   SELECT * FROM users WHERE phone = 13800138000;
   -- 正确：使用字符串
   SELECT * FROM users WHERE phone = '13800138000';
   ```

3. **LIKE 以通配符开头**
   ```sql
   -- %在最左，全表扫描
   SELECT * FROM products WHERE name LIKE '%手机';
   -- 正确：前置确定字符，或使用全文索引
   SELECT * FROM products WHERE name LIKE '华为手机%';
   ```

4. **OR 连接不同类型列**
   ```sql
   -- or 导致索引失效（建议拆分为 UNION）
   SELECT * FROM users WHERE id = 1 OR email = 'a@b.com';
   ```

5. **NOT NULL / IS NOT NULL**
   - 若字段大部分为非空，索引效果差，建议建默认值

6. **联合索引违反最左前缀原则**
   ```sql
   -- 索引为 (status, type, create_time)
   -- 失效场景
   SELECT * FROM orders WHERE type = 1;
   -- 有效场景
   SELECT * FROM orders WHERE status = 1 AND type = 1;
   ```

**避免策略**：用 EXPLAIN 分析执行计划，确保 type 至少为 range 级别，避免 ALL（全表扫描）。

---

## 题目二：如何优化深度分页（Deep Pagination）问题？请给出至少3种方案。

### 参考答案

深度分页即 `LIMIT 10000, 10`，偏移量越大性能越差，因为 MySQL 先扫描前10010行再丢弃前10000行。

### 方案一：基于主键的范围查询

```sql
-- 慢：偏移量大时
SELECT * FROM orders ORDER BY id LIMIT 10000, 10;

-- 快：记录上次查询最大ID
SELECT * FROM orders WHERE id > last_max_id ORDER BY id LIMIT 10;
```

### 方案二：子查询 + 延迟关联

```sql
SELECT t.* FROM orders t
INNER JOIN (
    SELECT id FROM orders
    ORDER BY id
    LIMIT 10000, 10
) AS sub ON t.id = sub.id;
```

### 方案三：游标分页（Keyset Pagination）

```sql
-- 首次查询
SELECT * FROM orders ORDER BY create_time DESC LIMIT 10;
-- 下一页：传入上一页最后一条的时间戳
SELECT * FROM orders
WHERE create_time < '2026-08-01 10:00:00'
ORDER BY create_time DESC LIMIT 10;
```

### 方案四：ElasticSearch / ClickHouse 承接海量数据

- 分页只查询前 N 页，真正深度查询走搜索引擎。

---

## 题目三：MySQL 与 PostgreSQL 的 MVCC 机制有何区别？它们分别解决了什么问题？

### 参考答案

两者都使用 MVCC（多版本并发控制）解决读写冲突，但实现不同。

### MySQL InnoDB（Percona Server 8.0+）

- **Undo Log 链**：每行数据有 `DB_TRX_ID`（事务ID）和 `DB_ROLL_PTR`（回滚指针）
- 读取时根据 `READ_VIEW` 判断：若事务ID > 当前事务，则沿回滚指针找可见历史版本
- **Purge 线程**：定期清理不再需要的 undo 页
- 特点：快照读（consistent read）基于 undo 链构建

### PostgreSQL

- **Heap Tuple + VISIBLE_XMIN/XMAX**：每行存储 `xmin`（插入事务）、`xmax`（删除/更新事务）
- 每个事务有 `xmin`/`xmax` 对比来判断可见性
- **Vacuum 进程**：清理死亡元组，防止表膨胀
- **PostgreSQL 12+**：heap-only-tuple (HOT) 减少索引更新

### 核心区别

| 特性 | MySQL InnoDB | PostgreSQL |
|------|-------------|------------|
| 版本存储 | Undo Log 独立表空间 | 行数据直接存储历史版本 |
| 可见性判断 | READ_VIEW（事务快照） | 事务号对比（xmin/xmax） |
| 清理机制 | Purge 线程 | VACUUM 进程 |
| 写冲突 | 间隙锁（Next-Key Lock） | SSI（Serializable Snapshot Isolation） |

---

## 题目四：如何设计一套 Redis + MySQL 的缓存策略？缓存穿透、缓存击穿、缓存雪崩如何应对？

### 参考答案

### 缓存架构：Cache-Aside（旁路缓存）

```sql
-- 读：先查Redis，未命中查MySQL并回填
def get_user(uid):
    user = redis.get(f"user:{uid}")
    if user: return user
    user = mysql.query("SELECT * FROM users WHERE id = %s", uid)
    redis.setex(f"user:{uid}", 3600, user)
    return user

-- 写：先写MySQL，再删缓存（而非更新缓存）
def update_user(uid, data):
    mysql.execute("UPDATE users SET ... WHERE id = %s", uid)
    redis.delete(f"user:{uid}")  # 删除而非更新，避免脏数据
```

### 缓存穿透：查询不存在的数据

- **布隆过滤器**：将所有存在数据的Key存入Bitmap，拦截不存在数据的查询
- **空值缓存**：对查询结果为空的Key也缓存一个短TTL空值（如 `"NULL"`），TTL设为60s

### 缓存击穿：热点Key过期瞬间大量请求打到DB

- **互斥锁**：SETNX 保证只有一个请求回源重建缓存
- **永不过期**：不设置TTL，用后台异步线程更新，或逻辑过期（返回旧数据同时异步重建）

### 缓存雪崩：大量Key同时过期

- **过期时间随机化**：`TTL = base + random(0, 300)`
- **Redis 高可用**：主从 + Sentinel 或 Cluster 避免 Redis 全崩导致击穿
- **接口限流**：Hystrix/Sentinel 保护 MySQL
- **预热**：系统启动时主动加载热点数据到缓存

---

## 题目五：Explain 执行计划中 type 字段有哪些值？如何根据 type 判断索引效率？

### 参考答案

type 字段从最优到最差排序：

| type 值 | 含义 | 说明 |
|---------|------|------|
| **system** | 系统表，只有1行 | 最好 |
| **const** | 最多1行匹配，常量等值查询 | 极佳 |
| **eq_ref** | 唯一索引等值扫描，连表时每行只匹配1条 | 好 |
| **ref** | 非唯一索引等值扫描，可能匹配多行 | 较好 |
| **ref_or_null** | ref + 额外 NULL 查询 | 尚可 |
| **range** | 索引范围扫描（ BETWEEN、IN、>、<） | 可接受 |
| **index** | 全索引扫描（只扫描索引树） | 较差 |
| **ALL** | 全表扫描 | 最差，需优化 |

### 关键指标判断

```sql
EXPLAIN SELECT * FROM orders WHERE status = 1 AND type = 2;

-- 重点关注：
-- type: 应为 ref 或 range，拒绝 ALL/index
-- key: 实际使用的索引名
-- key_len: 索引覆盖长度，越长说明越精确
-- rows: 预计扫描行数，越少越好
-- Extra:
--   Using filesort → 有ORDER BY且无索引，需优化
--   Using temporary → 用了临时表，需优化
--   Using index → 覆盖索引，无需回表，最优
```

### 优化实战

```sql
-- 为 (status, type) 建立联合索引
ALTER TABLE orders ADD INDEX idx_status_type(status, type);

-- 验证：确保 type = range 而不是 ALL
EXPLAIN SELECT * FROM orders WHERE status = 1 AND type IN (1, 2, 3);
```

---

> 📚 更多面试题：[https://github.com/qq286158530/interview-questions](https://github.com/qq286158530/interview-questions)
