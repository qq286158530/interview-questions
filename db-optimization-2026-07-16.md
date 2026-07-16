# 数据库性能优化面试题

> 📅 整理日期：2026-07-16
> 🎯 涵盖：MySQL / PostgreSQL 索引优化、查询优化、存储引擎、缓存策略等核心知识点

---

## 题目一：如何定位并优化慢查询？

### 参考答案

**定位慢查询的步骤：**

1. **开启慢查询日志**
   ```sql
   -- MySQL
   SET GLOBAL slow_query_log = 'ON';
   SET GLOBAL long_query_time = 1;  -- 超过1秒记录
   SET GLOBAL slow_query_log_file = '/var/log/mysql/slow.log';

   -- PostgreSQL
   ALTER SYSTEM SET log_min_duration_statement = 1000;  -- 毫秒
   ```

2. **使用 EXPLAIN 分析执行计划**
   ```sql
   EXPLAIN ANALYZE SELECT * FROM orders WHERE status = 1;
   ```

3. **关键关注指标：**
   - `type`：最好达到 `ref` / `range`，避免 `ALL`（全表扫描）
   - `key`：实际使用的索引
   - `rows`：扫描行数，越少越好
   - `Extra`：是否出现 `Using filesort`、`Using temporary`

**优化手段：**

- 为 WHERE / ORDER BY / JOIN 字段建立合适索引
- 避免 SELECT *，只查必要字段
- 优化子查询，改用 JOIN
- 分页大偏移量改用游标分页
- 对历史数据做归档清理

> 📎 来源：[MySQL 慢查询日志配置与使用](https://dev.mysql.com/doc/refman/8.0/en/slow-query-log.html)

---

## 题目二：什么情况下索引会失效？如何避免？

### 参考答案

**常见索引失效场景：**

| 失效场景 | 示例 | 后果 |
|---------|------|------|
| 函数/运算 | `WHERE YEAR(create_time) = 2026` | 全表扫描 |
| 类型转换 | `WHERE phone = 13800138000`（phone 为 VARCHAR） | 全表扫描 |
| 前导模糊匹配 | `WHERE name LIKE '%张'` | 全表扫描 |
| OR 条件 | `WHERE id = 1 OR age = 20`（id有索引，age无） | 部分失效 |
| 组合索引不遵循最左前缀 | `WHERE b = 2`（索引为 `(a,b,c)`） | 索引失效 |
| 表数据过小 | 数据量 < 几百行 | 全表扫描更快 |

**PostgreSQL 额外注意：**
- 对 JSON 字段直接比较 `data->'name' = 'abc'` 不会用索引，需改用 `data->>'name' = 'abc'` 或创建表达式索引
- 统计信息不准时，Planner 可能选错执行计划，需定期 `ANALYZE`

**避免策略：**
- 确保字段类型一致
- 避免在索引列上使用函数
- 模糊查询尽量使用后缀或全文索引
- 组合索引遵循最左前缀原则

> 📎 来源：[MySQL Indexes and EXPLAIN](https://dev.mysql.com/doc/refman/8.0/en/using-explain.html)

---

## 题目三：MySQL InnoDB 与 MyISAM 的核心区别是什么？如何选型？

### 参考答案

| 特性 | InnoDB | MyISAM |
|------|--------|--------|
| 事务支持 | ✅ 支持 ACID | ❌ 不支持 |
| 行锁 | ✅ 支持高并发 | ❌ 表锁 |
| 外键 | ✅ 支持 | ❌ 不支持 |
| 崩溃恢复 | ✅ 自动恢复 | ❌ 需手动修复 |
| 全文索引 | ✅ 5.6+支持 | ✅ 原生支持 |
| 存储空间 | 约 2x（支持MVCC） | 较小 |
| 适用场景 | 核心业务、并发写入 | 读多写少、日志 |

**选型建议：**

- **InnoDB**：几乎所有业务系统首选（事务安全、并发高、支持外键）
- **MyISAM**：仅适用于极度追求存储空间、只读或极少并发写入的场景（如日志表历史归档）

> 📎 来源：[MySQL Storage Engines](https://dev.mysql.com/doc/refman/8.0/en/storage-engines.html)

---

## 题目四：如何设计分库分表方案？分片键如何选择？

### 参考答案

**分库分表适用场景：**
- 单表数据量超过千万级（MySQL）或过亿（PostgreSQL）
- 磁盘 IO 成为瓶颈
- 连接数达到上限

**水平分片策略：**

1. **按 ID 哈希分片**
   - 优点：数据分布均匀
   - 缺点：按时间范围查询困难，扩容需要数据迁移

2. **按时间分片**
   - 优点：适合时序数据，查询友好
   - 缺点：热点数据集中在最新分片

3. **按业务字段分片**（如 user_id、region）
   - 优点：同类型数据聚合，减少跨表JOIN
   - 缺点：数据倾斜风险

**分片键选择原则：**
- 查询频率最高字段优先（如用户侧查询选 user_id）
- 避免跨分片查询
- 避免数据热点集中

**常见中间件：**
- ShardingSphere（Java）
- Vitess（Go）
- pg_partman（PostgreSQL 原生分区）

> 📎 来源：[ShardingSphere 官方文档](https://shardingsphere.apache.org/)

---

## 题目五：数据库缓存策略如何设计？如何避免缓存穿透、击穿、雪崩？

### 参考答案

**缓存架构：**
```
客户端 → Redis/Memcached → 数据库
```

**缓存更新策略：**
- **Cache Aside（旁路缓存）**：读时先缓存后DB，写时先DB后删缓存（最常用）
- **Read Through**：缓存自动加载，代码透明
- **Write Through**：写缓存时同步写DB

**三大问题及解决方案：**

| 问题 | 原因 | 解决方案 |
|------|------|---------|
| **缓存穿透** | 查询不存在的数据每次都打到DB | 布隆过滤器 / 存空值（短期） |
| **缓存击穿** | 热点key过期瞬间大量请求打到DB | 互斥锁 / 永不过期 + 异步重建 |
| **缓存雪崩** | 大量key同时过期 | 过期时间加随机值 / Redis集群 |

**示例：互斥锁防击穿（Redis SETNX）**
```python
def get_cache(key):
    val = redis.get(key)
    if val is None:
        # 尝试获取锁
        lock = redis.setnx(key + "_lock", "1", ex=10)
        if lock:
            # 只有获取到锁才查库
            val = db.query(key)
            redis.setex(key, 3600, val)
            redis.delete(key + "_lock")
        else:
            # 没获取到锁，短暂等待后重试
            time.sleep(0.1)
            return get_cache(key)
    return val
```

> 📎 来源：[Redis 缓存设计与失效策略](https://redis.io/docs/manualpatterns/cache/)

---

## 📚 更多学习资源

- [MySQL 8.0 Reference Manual](https://dev.mysql.com/doc/refman/8.0/en/)
- [PostgreSQL Documentation](https://www.postgresql.org/docs/)
- [Redis Official Documentation](https://redis.io/documentation)
