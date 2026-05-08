# 数据库性能优化面试题

> 更新时间：2026-05-08
> 涵盖：MySQL / PostgreSQL 索引优化、查询优化、存储引擎、缓存策略等核心知识点

---

## 题目 1：MySQL 中索引失效的常见场景有哪些？如何避免？

### 参考答案

**索引失效的常见场景：**

1. **使用函数或运算**：在索引列上使用 `LEFT()`、`YEAR()`、`SUBSTRING()` 等函数，或进行算术运算（`id + 1 = 10`），导致索引失效。

2. **类型转换**：where 条件中字段类型与传入参数类型不匹配（如字符串字段用数字比较），MySQL 发生隐式类型转换。

3. **LIKE 以通配符开头**：`LIKE '%abc'` 导致全表扫描；`LIKE 'abc%'` 可以使用索引。

4. **使用 NOT、<>、!=**：部分情况下优化器选择全表扫描而非索引。

5. **联合索引违反最左前缀原则**：创建了 `(a, b, c)` 联合索引，但查询只使用 `WHERE c = ?`，无法使用索引。

6. **OR 连接不同字段**：当 OR 前后是不同字段，且字段上没有单独索引时，索引失效。

7. **使用 IS NULL / IS NOT NULL**：有时优化器放弃索引。

8. **表统计信息不准确**：数据量小或数据变化频繁时，优化器可能选择错误执行计划。

**避免策略：**
- 尽量使用索引覆盖（Covering Index）
- 避免在索引列上使用函数
- 使用强制索引提示 `FORCE INDEX`
- 定期 `ANALYZE TABLE` 更新统计信息
- 合理设计联合索引，遵循最左前缀原则

**来源参考：**
- https://dev.mysql.com/doc/refman/8.0/en/index-btrees.html
- https://www.cnblogs.com/jian-ong/p/mysql-indexes-and-null-values.html

---

## 题目 2：如何定位并优化慢查询？（从发现到解决的完整流程）

### 参考答案

**Step 1：开启慢查询日志**

```sql
-- 临时开启（MySQL）
SET GLOBAL slow_query_log = 'ON';
SET GLOBAL long_query_time = 1; -- 超过1秒记录
SET GLOBAL slow_query_log_file = '/var/log/mysql/slow.log';

-- PostgreSQL
ALTER SYSTEM SET log_min_duration_statement = 1000; -- 1000ms
SELECT pg_reload_conf();
```

**Step 2：分析慢查询日志**

```bash
# 使用 mysqldumpslow 汇总（MySQL）
mysqldumpslow -t 10 /var/log/mysql/slow.log

# 使用 pt-query-digest（推荐，功能更强大）
pt-query-digest /var/log/mysql/slow.log
```

**Step 3：使用 EXPLAIN 分析执行计划**

```sql
EXPLAIN SELECT ...;
EXPLAIN ANALYZE SELECT ...;  -- MySQL 8.0+ 可显示实际执行时间
```

重点关注：
- `type`: 尽可能达到 `ref`、`range`，避免 `ALL`（全表扫描）
- `key`: 确认实际使用的索引
- `rows`: 扫描行数，越少越好
- `Extra`: 避免 `Using filesort`、`Using temporary`

**Step 4：针对性优化**

| 问题 | 解决方案 |
|------|----------|
| 全表扫描 | 添加合适索引 |
| Using filesort | 优化 ORDER BY / GROUP BY，添加对应索引 |
| Using temporary | 优化 DISTINCT / UNION，增加 RAM 或改写 SQL |
| 扫描行数过多 | 缩小查询范围，使用分页 |

**Step 5：验证效果**

优化后再次 `EXPLAIN`，对比 `rows` 和执行时间。

**来源参考：**
- https://dev.mysql.com/doc/refman/8.0/en/slow-query-log.html
- https://www.postgresql.org/docs/current/performance-tips.html

---

## 题目 3：MySQL InnoDB 与 PostgreSQL 存储引擎的核心区别是什么？如何选型？

### 参考答案

**架构层面区别：**

| 特性 | InnoDB (MySQL) | PostgreSQL |
|------|---------------|------------|
| **并发模型** | MVCC + 行级锁 | MVCC + 多版本并发控制（更成熟） |
| **事务隔离级别** | 4级（READ UNCOMMITTED 到 SERIALIZABLE） | 4级，标准 SQL 隔离级别 |
| **索引结构** | B+Tree（聚簇索引为主） | B-Tree / Hash / GiST / GIN 等多种索引类型 |
| **外键支持** | 支持，但高并发下有额外锁开销 | 支持，且实现更完整 |
| **全文搜索** | 支持但较弱（InnoDB FTS） | 内置全文搜索 + 分词器 |
| **JSON支持** | JSON 函数，性能一般 | JSONB（Binary JSON），性能更好 |
| **表继承** | 不支持 | 支持，面向对象特性 |
| **锁粒度** | 行级锁 + Gap锁 | 行级锁 + 页级锁 |
| **崩溃恢复** | Crash Recovery（Redo Log） | WAL（预写日志）+ PITR |

**性能对比场景：**

- **OLTP 高并发短事务**：InnoDB 更优（轻量锁、内存缓存优化好）
- **OLAP 复杂查询**：PostgreSQL 更优（优化器更强大、并行查询）
- **需要 GIS / 数组 / JSONB**：PostgreSQL 更适合
- **需要全文检索**：PostgreSQL 体验更好

**选型建议：**
- 互联网高并发 Web 应用 → MySQL (InnoDB)
- 数据仓库 / 复杂分析 / 地理信息 → PostgreSQL
- 两者混用（读写分离 + ETL）也是常见架构

**来源参考：**
- https://dev.mysql.com/doc/refman/8.0/en/innodb-storage-engine.html
- https://www.postgresql.org/docs/current/intro-whatis.html

---

## 题目 4：什么是数据库缓存策略？Redis + MySQL 典型缓存架构如何设计？

### 参考答案

**缓存经典问题（Cache Aside 模式）：**

```
读请求：
1. 先查 Redis Cache
2. Cache 命中 → 直接返回
3. Cache 未命中 → 查 DB → 写入 Cache → 返回

写请求：
1. 写入 DB
2. 删除（而非更新）Cache 中的对应 Key
   （为什么不更新？避免并发写导致数据不一致）
```

**缓存经典问题及解决方案：**

| 问题 | 描述 | 解决方案 |
|------|------|----------|
| **缓存穿透** | 查询不存在的数据，每次都击穿到 DB | 布隆过滤器 / 缓存空值（NULL）|
| **缓存击穿** | 热 key 过期瞬间，大量请求击穿到 DB | 互斥锁 / 热点数据永不过期 |
| **缓存雪崩** | 大量 key 同时过期或 Redis 宕机 | TTL 随机化 + Redis 高可用 |
| **数据一致性** | 缓存与 DB 数据不一致 | 延迟双删 / Canal 监听 binlog |

**多级缓存架构示例：**

```
浏览器 → CDN → Nginx 本地缓存 → Redis Cluster → MySQL
```

**PostgreSQL 侧缓存：**

```sql
-- PostgreSQL 使用 pgpool-II 或内置 query cache
-- 或通过 Redis 缓存查询结果：
SELECT * FROM table WHERE ...;
-- TTL 60s，超时重新查询
```

**缓存Key设计原则：**
- `业务:表:id:字段` 格式，如 `user:profile:1001`
- 设置合理 TTL（根据数据更新频率）
- 热点数据预加载到 Redis

**来源参考：**
- https://redis.io/topics/lru-cache
- https://learn.redis.cn/04/02.html

---

## 题目 5：如何设计分库分表方案？ShardingSphere / MyCat 分片策略有哪些？

### 参考答案

**何时需要分库分表：**
- 单表数据量超过 1000 万行
- 单库 QPS 超过 1 万
- 磁盘存储接近瓶颈

**垂直拆分（按业务/字段）：**

```
用户库：user_id, username, email, password
订单库：order_id, user_id, amount, created_at
商品库：product_id, name, price, stock
```

**水平拆分（按数据分片）：**

| 分片策略 | 原理 | 适用场景 |
|----------|------|----------|
| **哈希分片** | `shard_key % node_count` | 数据均匀分布，但扩缩容成本高 |
| **范围分片** | 按 ID 区间或时间范围 | 适合按时间查询，但可能热点不均 |
| **一致性哈希** | 环形哈希空间 + 虚拟节点 | 扩缩容数据迁移量较小 |
| **基因法** | 基于雪花算法生成带库位信息的 ID | 需要 ID 本身包含分片信息 |

**ShardingSphere-JDBC 配置示例（哈希分片）：**

```yaml
spring:
  shardingsphere:
    rules:
      sharding:
        tables:
          t_order:
            actual-data-nodes: ds_${0..1}.t_order_${0..15}
            table-strategy:
              standard:
                sharding-column: order_id
                sharding-algorithm-name: order_inline
            key-generate-strategy:
              column: order_id
              key-generator-name: snowflake
```

**分片后跨表/跨库查询问题：**

- **分页**：在应用层做归并排序（分片越多越慢）
- **跨分片 JOIN**：在应用层多次查询后合并 / 使用广播表
- **分布式事务**：使用 Seata AT 或 TCC 模式
- **唯一ID**：雪花算法 / 分布式 ID 生成器（Leaf）

**扩容策略（平滑迁移）：**

1. 双写（旧库 + 新库同时写入）
2. 历史数据用工具同步（Canal / Debezium）
3. 灰度切读，逐步切流量
4. 确认无误后下掉旧库

**来源参考：**
- https://shardingsphere.apache.org/
- https://www.cnblogs.com/superfj/p/sharding.html

---

## 📚 更多优质资源

- [MySQL 8.0 Reference Manual](https://dev.mysql.com/doc/refman/8.0/en/)
- [PostgreSQL 15 Documentation](https://www.postgresql.org/docs/15/)
- [Redis 设计与实现](https://redis.io/topics/lru-cache)
- [ShardingSphere 官方文档](https://shardingsphere.apache.org/)

---

*整理自网络公开资源，仅供学习交流使用*
