# 数据库性能优化面试题

> 📅 更新时间：2026-05-11  
> 📚 来源：综合整理（MySQL InnoDB / PostgreSQL 核心知识点）

---

## 题目一：MySQL InnoDB 索引失效的常见场景有哪些？

### 参考答案

InnoDB 索引失效的常见场景包括：

1. **使用函数或表达式**
   ```sql
   -- 失效：WHERE YEAR(created_at) = 2025
   -- 生效：WHERE created_at >= '2025-01-01' AND created_at < '2026-01-01'
   ```

2. **隐式类型转换**
   ```sql
   -- phone 为 varchar，但传入数字
   WHERE phone = 13800138000  -- 导致索引失效
   WHERE phone = '13800138000'  -- 正常使用索引
   ```

3. ** LIKE 模式以通配符开头**
   ```sql
   WHERE name LIKE '%张%'   -- 失效（最左前缀原则破坏）
   WHERE name LIKE '张%'    -- 可使用索引
   ```

4. **使用 OR 连接非索引列**
   ```sql
   WHERE name = '张三' OR age = 25
   -- 若 age 无索引，name 索引也会失效
   ```

5. **索引列参与运算**
   ```sql
   WHERE id + 1 = 100  -- 失效
   WHERE id = 100 - 1  -- 生效
   ```

6. **统计信息不准确导致走错执行计划**
   `ANALYZE TABLE` 刷新统计信息可改善。

---

## 题目二：如何定位和解决 MySQL 慢查询？请描述具体步骤。

### 参考答案

**步骤一：开启慢查询日志**

```ini
slow_query_log = 1
slow_query_log_file = /var/log/mysql/slow.log
long_query_time = 2
```

**步骤二：分析慢查询日志**

使用 `mysqldumpslow` 工具统计高频慢 SQL：
```bash
mysqldumpslow -t 5 /var/log/mysql/slow.log
```

**步骤三：使用 EXPLAIN 分析执行计划**

```sql
EXPLAIN SELECT ... FROM ...
EXPLAIN ANALYZE SELECT ... FROM ...  -- MySQL 8.0+ 可看实际行数
```

重点关注：
- `type`：最好达到 `ref`/`range`，避免 `ALL`
- `key`：确认实际使用了索引
- `rows`：估算扫描行数，越少越好
- `Extra`：出现 `Using filesort`/`Using temporary` 需优化

**步骤四：根据结果优化**

- 添加合适索引
- 优化 SQL 结构（子查询改 JOIN、分页优化）
- 拆分大事务
- 读写分离

**步骤五：验证效果**

重复执行 EXPLAIN 或查看 QPER 监控指标变化。

---

## 题目三：PostgreSQL 与 MySQL 在索引实现上有哪些主要区别？

### 参考答案

| 特性 | MySQL (InnoDB) | PostgreSQL |
|------|---------------|------------|
| 主键索引 | 聚簇索引（数据按主键顺序存储） | 非聚簇，数据独立存储 |
| 索引类型 | B+Tree（默认）、R-Tree（空间）、FULLTEXT | B-Tree、Hash、GIN、GIST、BRIN、GiST、SP-GiST |
| 表达式索引 | ❌ 不支持 | ✅ 支持：`CREATE INDEX idx ON t (lower(name))` |
| 部分索引 | ❌（MySQL 8.0+ 支持） | ✅ 支持 |
| 覆盖索引 | ✅ Extra 列直接返回 | ✅ 支持 |
| 并行索引构建 | ✅ 支持（MAX_BUILD_INDEX_CONCURRENCY） | ✅ 支持（`CONCURRENTLY`） |
| 索引 Hint | ✅ 支持强制使用指定索引 | ❌ 靠统计信息和查询计划调整 |

PostgreSQL 的索引更灵活，适合复杂场景；MySQL 的 B+Tree 实现更简单，范围查询性能更稳定。

---

## 题目四：如何设计分页查询才能避免深度分页性能问题？

### 参考答案

**问题根源**：深度分页 `LIMIT 100000, 10` 需要跳过 100010 条记录后再取 10 条，极慢。

**方案一：基于主键的范围查询（推荐）**

```sql
-- 第1页
SELECT * FROM orders WHERE id > 0 ORDER BY id LIMIT 10;

-- 第2页（上一页最后一条的 id）
SELECT * FROM orders WHERE id > 100000 ORDER BY id LIMIT 10;
```

**方案二：延迟关联**

```sql
SELECT o.* FROM orders o
INNER JOIN (
    SELECT id FROM orders ORDER BY id LIMIT 100000, 10
) AS t ON o.id = t.id;
```

子查询只查主键不回表，再 JOIN 获取完整数据，比直接分页快很多。

**方案三：游标分页（PostgreSQL 的 Keyset）**

```sql
-- 使用 Last_visible_date 作为游标
SELECT * FROM posts
WHERE created_at < :cursor
ORDER BY created_at DESC
LIMIT 20;
```

**方案四：Total Count 缓存**

`FOUND_ROWS()` 代价高，改为前端展示「约 xxx 条」，后端定时统计即可。

---

## 题目五：数据库缓存策略有哪些？如何避免缓存击穿/穿透/雪崩？

### 参考答案

**常用缓存策略：**

- **Cache-Aside**：应用先查缓存，未命中查 DB，回填缓存（最常用）
- **Read-Through**：缓存组件自动加载，应用只访问缓存
- **Write-Through**：写操作同时写 DB 和缓存，保证一致性
- **Write-Behind**：异步写入 DB，高性能但有丢数据风险

**缓存问题应对：**

| 问题 | 现象 | 解决方案 |
|------|------|---------|
| **缓存击穿** | 热点 key 失效时大量请求打到 DB | 互斥锁 / 永久不过期 + 异步重建 / 热点数据永不过期 |
| **缓存穿透** | 查询不存在的数据，直接打 DB | 布隆过滤器 / 空值缓存 / 参数校验 |
| **缓存雪崩** | 大量 key 同时失效 | 随机过期时间 / 分级过期 / 分布式锁 / 高可用架构 |

**实践建议：**

```python
# 缓存击穿：使用分布式锁
def get_data(key):
    data = redis.get(key)
    if data is None:
        lock = redis.lock(f"lock:{key}", timeout=10)
        if lock.acquire():
            data = db.query(...)
            redis.setex(key, 3600, data)
            lock.release()
        else:
            time.sleep(0.1)
            return get_data(key)  # 重试
    return data
```

Redis 配合数据库是经典组合，注意设置合理的 TTL 和过期策略。

---

> 📌 **提示**：面试时除了回答知识点，最好能结合实际项目经验举例，效果更佳！