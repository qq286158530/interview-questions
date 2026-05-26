# 数据库性能优化面试题 | 2026-05-26

> 涵盖 MySQL / PostgreSQL 索引优化、查询优化、存储引擎、缓存策略等核心知识点

---

## 题目一：MySQL InnoDB 索引失效的常见场景有哪些？

**参考答案：**

1. **使用函数或表达式**：在索引列上使用 `YEAR(created_at)`、`LEFT(name, 4)` 等函数，会导致索引失效。
2. **隐式类型转换**：若索引列为字符串类型，但传入数字参数，InnoDB 会隐式将列转为数字，导致全表扫描。
3. ** LIKE 前导通配**：`LIKE '%keyword'` 以通配符开头，无法利用 B+Tree 索引。
4. **范围查询右侧索引失效**：联合索引 (a, b, c)，查询条件为 `WHERE a = 1 AND b > 2 AND c = 3`，c 列无法使用索引。
5. **使用 `NOT NULL` / `<>` / `NOT IN`**：这类条件通常无法使用索引（少数优化器行为除外）。
6. **使用 `OR` 连接非索引列**：`WHERE col1 = 1 OR col2 = 2`，若 col2 无索引则全表扫描。
7. **数据量太小**：优化器认为全表扫描更快，直接放弃索引。

> 📎 来源：[深入理解 MySQL 索引失效机制](https://developer.aliyun.com/article/1398543)

---

## 题目二：如何优化深度分页（`LIMIT 100000, 20`）问题？

**参考答案：**

深度分页是数据库性能杀手，核心问题是**回表次数过多**。

### 方案一：游标分页（延迟关联）

```sql
-- 先通过索引定位偏移位置，再关联回原表取字段
SELECT * FROM orders o
INNER JOIN (
    SELECT id FROM orders
    WHERE create_time > '2025-01-01'
    ORDER BY create_time
    LIMIT 100000, 20
) t ON o.id = t.id;
```

### 方案二：游标式分页（记录上次的最后一条 ID）

```sql
-- 第一页
SELECT * FROM orders ORDER BY id LIMIT 20;
-- 第二页：传入上一页最后一条 id
SELECT * FROM orders
WHERE id > :last_id
ORDER BY id LIMIT 20;
```

### 方案三：范围条件替代 OFFSET

```sql
-- 用范围条件代替OFFSET，避免跳过大量数据
SELECT * FROM orders
WHERE id BETWEEN 100000 AND 100019;
```

### 根本原则

- **禁止在生产环境使用超大的 OFFSET**
- 业务层面限制最大页数（如最多展示100页）
- 对高频查询字段建立合适索引

> 📎 来源：[美团技术沙龙——MySQL 深度分页优化](https://tech.meituan.com/2017/06/09/mysql-deep-paging-optimization.html)

---

## 题目三：PostgreSQL 与 MySQL 在索引实现上的核心差异？

**参考答案：**

| 特性 | MySQL (InnoDB) | PostgreSQL |
|------|----------------|------------|
| 默认索引 | B+Tree | B-Tree |
| 多列索引 | 高度依赖列顺序 | 支持索引并列（多索引组合） |
| 表达式索引 | ❌ 不支持 | ✅ 支持 `CREATE INDEX idx ON t (lower(name))` |
| 部分索引 | ❌ 有限支持 | ✅ 支持 `WHERE` 子句过滤 |
| 覆盖索引 | 通过 ICP 优化 | 支持 Index Only Scan |
| 并行索引构建 | ❌ | ✅ 支持 `CONCURRENTLY` |
| NULL 排序 | 统一放在数据页末尾 | NULL 方向可配置（NULLS FIRST/LAST） |

PostgreSQL 的**表达式索引**和**部分索引**是其区别于 MySQL 的重要特性，可以大幅减少索引体积并提升查询效率。

> 📎 来源：[PostgreSQL 索引类型与 MySQL 对比](https://www.postgresql.org/docs/current/indexes.html)

---

## 题目四：如何排查和解决 MySQL 慢查询？

**参考答案：**

### Step 1：开启慢查询日志

```ini
slow_query_log = 1
slow_query_log_file = /var/log/mysql/slow.log
long_query_time = 1
```

### Step 2：使用 EXPLAIN 分析执行计划

```sql
EXPLAIN FORMAT=JSON SELECT ...
-- 关注：
-- - type: 是否达到 ref/range
-- - key: 实际使用的索引
-- - rows: 扫描行数（越少越好）
-- - extra: 是否出现 filesort / temporary
```

### Step 3：使用 PROFILING

```sql
SET profiling = 1;
-- 执行查询
SHOW PROFILES;
SHOW PROFILE FOR QUERY 1;
```

### Step 4：常见优化手段

| 问题 | 优化方案 |
|------|---------|
| 全表扫描 | 添加合适索引 |
| 回表过多 | 覆盖索引（覆盖查询字段） |
| 文件排序 | 优化 ORDER BY 字段或建立联合索引 |
| 临时表 | 避免在 WHERE 中使用函数，重写 SQL |
| JOIN 嵌套循环 | 确保被驱动表有索引，小表驱动大表 |

### Step 5：使用 SQL 诊断工具

- `pt-query-digest`（Percona Toolkit）分析慢查询日志
- `performance_schema` 监控各阶段耗时

> 📎 来源：[MySQL 慢查询优化官方指南](https://dev.mysql.com/doc/refman/8.0/en/slow-query-log.html)

---

## 题目五：Redis 缓存与数据库双写一致性问题如何处理？

**参考答案：**

双写一致性问题来源于：缓存和数据库操作非原子性，任一步骤失败都会导致不一致。

### 方案一：Cache Aside（最常用）

```
读：Cache命中 → 直接返回；未命中 → 查DB → 写Cache → 返回
写：更新DB → 删除Cache（非更新Cache）
```

**为什么删除而不是更新？** 因为更新 Cache 后，在并发场景下可能被另一个请求的旧数据覆盖。

### 方案二：延迟双删

```python
def write(key, value):
    db.update(key, value)
    cache.delete(key)
    # 异步等待500ms，再次删除（应对并发读导致的旧数据回填）
    sleep(0.5)
    cache.delete(key)
```

### 方案三：订阅 binlog 异步更新

```
DB变更 → 解析 binlog → 异步写入 Cache
（使用 Canal/MySQL Streamout 或 Debezium）
```

### 方案四：分布式锁（强一致）

```python
with redis.lock(f"write:{key}", blocking_timeout=10):
    db.update(key, value)
    cache.set(key, value)
```

### 一致性等级对比

| 方案 | 一致性 | 性能 | 实现复杂度 |
|------|--------|------|-----------|
| Cache Aside | 最终一致 | 高 | 低 |
| 延迟双删 | 最终一致 | 中 | 中 |
| binlog订阅 | 最终一致 | 高 | 高 |
| 分布式锁 | 强一致 | 低 | 高 |

> 📎 来源：[Redis 与数据库一致性深度分析](https://juejin.cn/post/6844904041675112461)

---

*📁 本题库由 GitHub 自动化推送，仓库：[github.com/qq286158530/interview-questions](https://github.com/qq286158530/interview-questions)*