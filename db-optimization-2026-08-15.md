# 数据库性能优化面试题

> 📅 2026-08-15 | 来源：面试题整理

---

## 题目一：MySQL 索引失效的场景有哪些？如何避免？

### 参考答案

**常见的索引失效场景：**

1. **使用函数或运算**
   ```sql
   -- 索引失效 ❌
   SELECT * FROM users WHERE YEAR(create_time) = 2026;
   
   -- 正确写法 ✅
   SELECT * FROM users WHERE create_time >= '2026-01-01' AND create_time < '2027-01-01';
   ```

2. **隐式类型转换**
   ```sql
   -- phone 是 varchar 类型，传入数字会导致索引失效 ❌
   SELECT * FROM users WHERE phone = 13800138000;
   
   -- 正确写法 ✅
   SELECT * FROM users WHERE phone = '13800138000';
   ```

3. ** LIKE  以通配符开头**
   ```sql
   -- 索引失效 ❌
   SELECT * FROM users WHERE name LIKE '%张%';
   
   -- 索引生效 ✅
   SELECT * FROM users WHERE name LIKE '张%';
   ```

4. **联合索引违反最左前缀原则**
   ```sql
   -- 假设 idx(a, b, c)
   WHERE b = 1  -- 索引失效 ❌
   WHERE a = 1 AND c = 1  -- 部分生效（a 生效，c 不生效）⚠️
   WHERE a = 1  -- 索引生效 ✅
   ```

5. **使用 OR 连接多个条件（且未覆盖所有列）**
   ```sql
   -- 索引失效 ❌（除非所有列都在同一个索引中）
   SELECT * FROM users WHERE name = '张三' OR age = 20;
   ```

6. **不等于比较（NOT、<>、!=）**
   ```sql
   -- 索引失效 ❌
   SELECT * FROM users WHERE status != 1;
   ```

7. **IS NULL / IS NOT NULL**
   - 单独使用 `IS NULL` 可以使用索引
   - 但如果数据分布不均匀，优化器可能选择全表扫描

**如何避免索引失效：**
- 避免在索引列上使用函数
- 使用参数时确保类型匹配
- 合理设计索引，考虑查询模式
- 使用 `EXPLAIN` 分析执行计划
- 遵循最左前缀原则设计联合索引

---

## 题目二：如何分析一条 SQL 语句的性能？EXPLAIN 主要关注哪些字段？

### 参考答案

**使用 EXPLAIN 分析 SQL：**
```sql
EXPLAIN SELECT * FROM users WHERE name = '张三';
```

**主要关注字段：**

| 字段 | 含义 | 理想值 |
|------|------|--------|
| **type** | 访问类型 | const、eq_ref、ref（避免 ALL 全表扫描） |
| **key** | 实际使用的索引 | 非 NULL |
| **rows** | 扫描行数 | 越少越好 |
| **Extra** | 额外信息 | Using index、Using filesort、Using temporary |

**type 从好到差：**
```
const > eq_ref > ref > range > index > ALL
```

- `const`：主键或唯一索引的等值查询
- `ref`：非唯一索引的等值查询
- `range`：范围查询
- `ALL`：全表扫描（最差）

**Extra 常见值分析：**
- `Using index`：使用了覆盖索引，性能好 ✅
- `Using filesort`：需要额外排序（使用 filesort 算法），性能差 ❌
- `Using temporary`：使用了临时表，性能差 ❌
- `Using index condition`：索引下推优化，性能较好 ✅

**实战建议：**
```sql
-- 查看更详细信息（包括成本）
EXPLAIN ANALYZE SELECT * FROM users WHERE name = '张三';
```

---

## 题目三：MySQL InnoDB 和 MyISAM 存储引擎的区别？如何选择？

### 参考答案

| 特性 | InnoDB | MyISAM |
|------|--------|--------|
| **事务支持** | ✅ 支持 ACID 事务 | ❌ 不支持 |
| **行锁** | ✅ 支持行级锁 | ❌ 只支持表级锁 |
| **外键** | ✅ 支持 | ❌ 不支持 |
| **全文索引** | 5.6+ 支持 | ✅ 原生支持 |
| **存储方式** | 每个表两个文件（.ibd + .frm） | 三个文件（.MYD + .MYI + .frm） |
| **索引** | clustered index（聚集索引） | non-clustered index |
| **MVCC** | ✅ 支持 | ❌ 不支持 |
| **崩溃恢复** | ✅ 自动恢复 | ❌ 需手动修复 |

**聚集索引 vs 非聚集索引：**
- **InnoDB**：数据文件按主键顺序存储，主键索引就是聚集索引
- **MyISAM**：数据文件和索引文件分离，索引指向数据物理位置

**如何选择：**

| 场景 | 推荐引擎 |
|------|----------|
| 需要事务、并发写入 | InnoDB ✅ |
| 读多写少，以全文搜索为主 | MyISAM |
| 需要外键约束 | InnoDB ✅ |
| 需要全文索引（5.6 之前版本） | MyISAM |
| 数据量巨大，只读的报表场景 | MyISAM（压缩表） |

**建议：** 现代 MySQL 场景下，除非有特殊原因，**默认使用 InnoDB**。

---

## 题目四：什么是数据库慢查询？如何优化慢查询？

### 参考答案

**慢查询定义：** 执行时间超过 `long_query_time` 阈值（默认 10 秒）的 SQL 查询。

**排查步骤：**

1. **开启慢查询日志**
   ```sql
   -- 查看慢查询配置
   SHOW VARIABLES LIKE 'slow_query%';
   SHOW VARIABLES LIKE 'long_query_time';
   
   -- 开启慢查询日志
   SET GLOBAL slow_query_log = 'ON';
   SET GLOBAL long_query_time = 1;  -- 超过 1 秒记录
   ```

2. **分析慢查询日志**
   ```bash
   mysqldumpslow -s t -t 10 /var/lib/mysql/slow.log
   # -s t 按时间排序
   # -t 10 显示前 10 条
   ```

3. **使用 EXPLAIN 分析执行计划**

**慢查询优化策略：**

1. **索引优化**
   - 添加合适索引
   - 避免索引失效

2. **SQL 改写**
   ```sql
   -- 改写前：大表分页深度查询
   SELECT * FROM orders ORDER BY id LIMIT 1000000, 10;
   
   -- 改写后：使用游标分页
   SELECT * FROM orders WHERE id > #{last_id} ORDER BY id LIMIT 10;
   ```

3. **避免 SELECT *** 
   ```sql
   -- 只查询需要的列 ✅
   SELECT user_id, name FROM users WHERE id = 1;
   ```

4. **分解大查询**
   ```sql
   -- 改写 IN 子查询
   SELECT * FROM orders WHERE user_id IN (SELECT id FROM users WHERE status = 1);
   
   -- 改为 JOIN ✅
   SELECT o.* FROM orders o INNER JOIN users u ON o.user_id = u.id WHERE u.status = 1;
   ```

5. **批量操作**
   ```sql
   -- 多次单条插入
   INSERT INTO orders VALUES (1), (2), (3);  -- ✅ 批量
   ```

6. **查询缓存**（MySQL 8.0 已移除）
   - 应用层缓存（Redis）替代

---

## 题目五：Redis 缓存策略有哪些？如何避免缓存穿透、击穿、雪崩？

### 参考答案

**常见缓存策略：**

| 策略 | 说明 | 适用场景 |
|------|------|----------|
| **Cache-Aside** | 应用先查缓存，miss 时查 DB 并写入缓存 | 读多写少 |
| **Read-Through** | 缓存自动加载数据，应用只查缓存 | 读多 |
| **Write-Through** | 写操作同时写缓存和 DB | 数据一致性要求高 |
| **Write-Behind** | 异步批量写 DB | 高并发写入 |

**Cache-Aside 模式示例：**
```python
def get_user(user_id):
    # 1. 先查缓存
    user = redis.get(f"user:{user_id}")
    if user:
        return json.loads(user)
    
    # 2. 缓存未命中，查数据库
    user = db.query("SELECT * FROM users WHERE id = ?", user_id)
    
    # 3. 写入缓存
    redis.setex(f"user:{user_id}", 3600, json.dumps(user))
    return user
```

---

**三大缓存问题及解决方案：**

### 1. 缓存穿透（查询不存在的数据）

**问题：** 大量请求查询不存在的数据，直接打到 DB。

**解决方案：**
```python
# 方案一：缓存空值（短过期时间）
if user is None:
    redis.setex(f"user:{user_id}", 60, "")  # 空值缓存 60 秒

# 方案二：布隆过滤器
bloom.add(user_id)
if not bloom.might_contain(user_id):
    return None  # 直接拒绝
```

### 2. 缓存击穿（热点 key 过期）

**问题：** 热点 key 过期瞬间，大量请求同时击穿到 DB。

**解决方案：**
```python
# 方案一：互斥锁（分布式锁）
lock = redis.setnx("lock:user:1", "1", nx=True, ex=10)
if lock:
    user = db.query(...)
    redis.setex("user:1", 3600, user)
    redis.delete("lock:user:1")
else:
    time.sleep(0.1)
    return get_user(user_id)  # 等待后重试

# 方案二：永不过期 + 异步更新
# 给缓存设置很长的 TTL，同时后台异步更新缓存
```

### 3. 缓存雪崩（大量 key 同时过期）

**问题：** 大量缓存 key 集中过期，导致 DB 压力骤增。

**解决方案：**
```python
# 方案一：过期时间随机化
redis.setex("user:1", 3600 + random.randint(0, 300), user)

# 方案二：多级缓存
# L1（本地缓存 Caffeine） -> L2（Redis） -> DB

# 方案三：服务降级 + 熔断
# 当 DB 压力大时，直接返回默认值，不查库
```

---

**缓存最佳实践：**
- 热点数据才缓存，冷数据无需缓存
- 设置合理的 TTL，避免集中过期
- 缓存要有降级方案，不能完全依赖缓存
- 监控缓存命中率，及时调整策略

---

> 📚 更多面试题请访问：[GitHub - interview-questions](https://github.com/qq286158530/interview-questions)
