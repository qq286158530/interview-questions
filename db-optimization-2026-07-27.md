# 数据库性能优化面试题

> 📅 日期：2026-07-27
> 📚 方向：MySQL / PostgreSQL 数据库性能优化
> 🔗 来源：综合整理自牛客网、掘金、SegmentFault 等技术社区

---

## 题目一：MySQL 索引失效的场景有哪些？如何避免？

### 答案

**常见的索引失效场景：**

1. **使用 `LIKE` 以通配符开头**
   ```sql
   -- 失效：LIKE '%关键字'
   SELECT * FROM user WHERE name LIKE '%张'
   
   -- 有效：LIKE '关键字%'
   SELECT * FROM user WHERE name LIKE '张%'
   ```

2. **使用函数或表达式**
   ```sql
   -- 失效：在索引列上使用函数
   SELECT * FROM user WHERE YEAR(create_time) = 2024
   
   -- 有效：改为范围查询
   SELECT * FROM user WHERE create_time >= '2024-01-01' AND create_time < '2025-01-01'
   ```

3. **类型转换**
   ```sql
   -- 失效：隐式类型转换
   SELECT * FROM user WHERE phone = 13800138000
   
   -- 有效：使用字符串
   SELECT * FROM user WHERE phone = '13800138000'
   ```

4. **使用 `OR` 连接条件**
   ```sql
   -- 失效：OR 导致索引中断
   SELECT * FROM user WHERE name = '张三' OR age = 20
   
   -- 有效：改为 UNION
   SELECT * FROM user WHERE name = '张三'
   UNION
   SELECT * FROM user WHERE age = 20
   ```

5. **联合索引不遵循最左前缀原则**
   ```sql
   -- 假设 idx(name, age, phone) 联合索引
   -- 失效：跳过 name
   SELECT * FROM user WHERE age = 20 AND phone = '138'
   
   -- 有效：遵循最左前缀
   SELECT * FROM user WHERE name = '张三' AND age = 20
   ```

6. **不等于 (`!=` / `<>`) 查询**
   ```sql
   -- 失效：索引无法用于不等于比较
   SELECT * FROM user WHERE status != 1
   ```

7. **`ORDER BY` 的坑**
   - 当 ORDER BY 的字段不在索引中，或排序方向不一致时
   - 优化：确保 ORDER BY 字段在索引中且方向一致

**如何避免索引失效：**
- 使用 `EXPLAIN` 分析查询计划
- 尽量使用覆盖索引（`SELECT` 的列都在索引中）
- 避免在索引列上使用函数
- 范围查询放在联合索引的最后

**来源：** [牛客网 - MySQL索引失效场景总结](https://www.nowcoder.com/activity/oj) | [掘金 - MySQL索引21连击](https://juejin.cn/post/6844903921596112910)

---

## 题目二：如何分析一条 SQL 语句的性能？`EXPLAIN` 各字段含义？

### 答案

**使用 `EXPLAIN` 分析 SQL：**
```sql
EXPLAIN SELECT * FROM user WHERE name = '张三' ORDER BY age;
```

**关键字段解析：**

| 字段 | 含义 | 好的值 |
|------|------|--------|
| `type` | 访问类型 | `const` > `eq_ref` > `ref` > `range` > `index` > `ALL` |
| `key` | 实际使用的索引 | 非 NULL |
| `rows` | 预计扫描行数 | 越少越好 |
| `Extra` | 额外信息 | `Using index`（覆盖索引）> `Using filesort`（需优化）|

**`Extra` 常见值及优化：**

- ✅ **`Using index`**：使用了覆盖索引，性能最佳
- ⚠️ **`Using filesort`**：需额外排序，应优化
- ⚠️ **`Using temporary`**：使用了临时表，应优化
- ⚠️ **`Using where`**：需要回表过滤数据

**优化策略：**
```sql
-- 添加合适索引消除 filesort
ALTER TABLE user ADD INDEX idx_name_age (name, age);

-- 验证优化效果
EXPLAIN SELECT name, age FROM user WHERE name = '张三' ORDER BY age;
```

**来源：** [MySQL官方文档 - EXPLAIN Output Format](https://dev.mysql.com/doc/refman/8.0/en/explain-output.html) | [美团技术 - MySQL性能优化神器EXPLAIN](https://tech.meituan.com/2021/01/14/mysql-explain.html)

---

## 题目三：MySQL InnoDB 和 MyISAM 存储引擎的区别？如何选择？

### 答案

| 特性 | InnoDB | MyISAM |
|------|--------|--------|
| **事务支持** | ✅ 支持 ACID 事务 | ❌ 不支持 |
| **行锁/表锁** | 行锁（高并发） | 表锁 |
| **外键约束** | ✅ 支持 | ❌ 不支持 |
| **崩溃恢复** | ✅ 自动恢复 | ❌ 需手动修复 |
| **全文索引** | 5.6+ 支持 | ✅ 原生支持 |
| **存储空间** | 较大（支持MVCC） | 较小 |
| **COUNT(*)** | 慢（全表扫描） | 快（存储了行数） |

**InnoDB 的核心优势：**

1. **MVCC（多版本并发控制）**
   - 读不加锁，写不加锁
   - 大幅提升并发读写能力

2. **行级锁 + 间隙锁**
   ```sql
   -- InnoDB 会锁定范围内的所有行
   SELECT * FROM user WHERE age BETWEEN 20 AND 30 FOR UPDATE;
   ```

3. **支持热备份**
   - 可用 `mysqldump --single-transaction` 在线备份

**如何选择：**

- ✅ **选 InnoDB**：生产环境、事务需求、高并发、 数据可靠性优先
- ⚠️ **选 MyISAM**：仅读/很少写的场景、需全文搜索、历史数据归档

**版本趋势：** MySQL 8.0+ 已将默认存储引擎从 MyISAM 改为 InnoDB。

**来源：** [MySQL官方文档 - InnoDB vs MyISAM](https://dev.mysql.com/doc/refman/8.0/en/innodb-introduction.html) | [高性能MySQL第三版](https://book.douban.com/subject/23047113/)

---

## 题目四：PostgreSQL 和 MySQL 在性能优化上的主要区别？

### 答案

**1. 索引机制差异**

| 特性 | MySQL | PostgreSQL |
|------|-------|------------|
| 索引类型 | B+Tree、全文、空间 | B+Tree、Hash、GIN、GiST、BRIN |
| 表达式索引 | ❌ 不支持 | ✅ 支持 |
| 部分索引 | ❌ 不支持 | ✅ 支持 |

```sql
-- PostgreSQL 表达式索引（MySQL做不到）
CREATE INDEX idx_user_upper ON user(UPPER(name));

-- PostgreSQL 部分索引
CREATE INDEX idx_active_user ON user(age) WHERE status = 'active';
```

**2. 执行计划分析**

```sql
-- PostgreSQL
EXPLAIN (ANALYZE, BUFFERS, FORMAT TEXT) SELECT * FROM user;

-- MySQL
EXPLAIN ANALYZE SELECT * FROM user;  -- 8.0+ 支持
```

**3. 并发模型**

- **PostgreSQL**：MVCC + S2PL（严格两阶段锁），读不阻塞写
- **MySQL InnoDB**：MVCC + 乐观锁，读不加锁

**4. 统计信息与查询优化器**

```sql
-- PostgreSQL 手动更新统计信息（大数据量后必须执行）
ANALYZE user;

-- MySQL 自动采样
ANALYZE TABLE user;
```

**5. 配置参数调优重点**

| 参数 | MySQL | PostgreSQL |
|------|-------|------------|
| 连接池 | `max_connections` | `max_connections` + `shared_buffers` |
| 缓存 | `innodb_buffer_pool_size` | `shared_buffers` + `effective_cache_size` |
| 写入调优 | `innodb_flush_log_at_trx_commit` | `synchronous_commit` |

**6. 实际选择建议**

- **选 PostgreSQL**：复杂查询、数据类型丰富（JSON/数组/地理信息）、需要表达式索引
- **选 MySQL**：Web 简单业务、高并发简单读写、生态环境成熟

**来源：** [PostgreSQL官方文档 - Performance Tips](https://www.postgresql.org/docs/current/performance-tips.html) | [Baeldung - MySQL vs PostgreSQL](https://www.baeldung.com/mysql-vs-postgresql)

---

## 题目五：数据库缓存策略有哪些？如何设计缓存架构？

### 答案

**1. 缓存策略分类**

| 策略 | 描述 | 适用场景 | 一致性 |
|------|------|----------|--------|
| **Cache Aside** | 应用先查缓存，缓存未命中查DB，写入缓存 | 读多写少 | 最终一致 |
| **Read Through** | 缓存负责加载数据，应用只查缓存 | 通用 | 最终一致 |
| **Write Through** | 写DB时同步写缓存 | 数据一致性要求高 | 强一致 |
| **Write Behind** | 异步批量写缓存 | 高吞吐写入 | 弱一致 |

**2. Cache Aside 最优实践**

```sql
-- 读操作
def get_user(user_id):
    user = redis.get(f"user:{user_id}")
    if not user:
        user = db.query("SELECT * FROM user WHERE id = %s", user_id)
        redis.setex(f"user:{user_id}", 3600, user)  # 1小时过期
    return user

-- 写操作（先写DB，再删缓存，而非更新缓存）
def update_user(user_id, data):
    db.execute("UPDATE user SET ... WHERE id = %s", user_id)
    redis.delete(f"user:{user_id}")  # 删除缓存而非更新
```

**3. 缓存常见问题及解决**

**缓存穿透：** 请求数据不存在
```python
# 解决方案：布隆过滤器 或 缓存空值
if bloom_filter.might_exist(key):
    # 查DB
else:
    return None  # 直接返回
```

**缓存击穿：** 热点key过期瞬间大量请求打到DB
```python
# 解决方案：分布式锁 + 永不过期
lock = redis.lock(f"lock:{key}", timeout=5)
if lock.acquire():
    data = db.query(...)
    redis.setex(key, None, data)  # 永不过期
    lock.release()
```

**缓存雪崩：** 大量key同时过期
```python
# 解决方案：随机过期时间 + 永不过期+后台更新
redis.setex(key, rand(3600, 7200), data)  # 随机1-2小时
```

**4. 多级缓存架构**

```
┌─────────────┐
│   用户请求   │
└──────┬──────┘
       ▼
┌─────────────┐   命中    ┌────────────┐
│  CDN/L1缓存 │ ────────→ │ 直接返回   │
└──────┬──────┘           └────────────┘
       ▼ 未命中
┌─────────────┐   命中    ┌────────────┐
│  Redis/L2   │ ────────→ │ 直接返回   │
└──────┬──────┘           └────────────┘
       ▼ 未命中
┌─────────────┐   命中    ┌────────────┐
│   本地缓存   │ ────────→ │ 直接返回   │
└──────┬──────┘           └────────────┘
       ▼ 未命中
┌─────────────┐
│   数据库     │
└─────────────┘
```

**5. Redis 性能优化配置**

```conf
# redis.conf 关键配置
maxmemory 10gb           # 限制内存
maxmemory-policy allkeys-lru  # 内存不足时淘汰策略
appendonly yes           # 开启AOF持久化
appendfsync everysec     # 每秒同步（平衡性能与安全）
```

**来源：** [Redis官方文档 - Persistence](https://redis.io/docs/management/persistence/) | [阿里云技术 - 缓存架构设计](https://developer.aliyun.com/article/743964) | [缓存那些事 - 美团技术团队](https://tech.meituan.com/2017/03/26/cache.html)

---

## 📌 面试小贴士

1. **索引优化口诀**：带头（索引最左列）不能死（避免函数/类型转换），中间（范围列）不能跳（断链），索引列上不计算
2. **SQL优化顺序**：先优化 `SELECT`/`WHERE`，再考虑索引，最后考虑缓存
3. **Explain 看点**：type（访问类型）、key（实际索引）、Extra（Using filesort/temporary 是优化重点）
4. **高频追问**：分布式环境下如何保证缓存一致性？

---

*📅 整理于 2026-07-27 | 适用于中高级数据库工程师面试准备*
