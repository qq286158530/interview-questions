# 数据库性能优化面试题

> 📅 整理日期：2026-08-21  
> 🏷️ 技术栈：MySQL / PostgreSQL  
> 📚 来源：综合整理自网络高质量技术文章

---

## 题目一：MySQL 中 InnoDB 引擎的索引原理是什么？为什么建议使用自增主键？

### 参考答案

InnoDB 使用 **B+Tree** 作为默认索引结构，所有索引都是 B+Tree。

**主键索引（聚簇索引）**：InnoDB 表中，数据行本身存储在主键索引的叶子节点中。每个 InnoDB 表都有一个主键索引，叶子节点包含完整的行数据。

**辅助索引（二级索引）**：叶子节点存储主键值，而非行数据的物理位置。因此通过辅助索引查询时，需要**回表**（先查辅助索引得到主键，再查主键索引获取完整行数据）。

**为什么建议使用自增主键？**
1. **顺序写入**：自增主键保证新插入的数据总是在叶子节点最后顺序追加，避免页分裂和随机IO。
2. **减少索引维护成本**：B+Tree 插入时，若主键无序（如 UUID），可能触发叶子节点分裂和平衡调整。
3. **占用空间更小**：INT/BIGINT 主键比字符串主键（UUID、varchar）占用更少空间，使辅助索引更紧凑，查询更快。
4. **覆盖索引友好**：主键值越小，辅助索引的每条记录占用空间越少，整体索引体积更小。

**例外场景**：对于需要分布式主键或分库分表场景，UUID/雪花算法更合适，但需要注意其对索引效率的影响。

> 📎 来源：https://dev.mysql.com/doc/refman/8.0/en/innodb-index-types.html  
> 📎 来源：https://www.cnblogs.com/wujian0118/p/12041099.html

---

## 题目二：如何定位并优化慢查询？请描述你的完整排查思路。

### 参考答案

**Step 1：开启慢查询日志，定位慢 SQL**

```sql
-- 查看慢查询配置
SHOW VARIABLES LIKE 'slow_query%';
SHOW VARIABLES LIKE 'long_query_time';

-- 开启慢查询日志（临时）
SET GLOBAL slow_query_log = 'ON';
SET GLOBAL long_query_time = 1; -- 超过1秒记录

-- 分析慢查询日志
mysqldumpslow -s t -t 10 /var/lib/mysql/slow.log
```

**Step 2：使用 EXPLAIN 分析执行计划**

```sql
EXPLAIN SELECT ... FROM ...
EXPLAIN ANALYZE SELECT ... FROM ...  -- MySQL 8.0+，会实际执行
```

重点关注字段：
- **type**：访问类型，const > eq_ref > ref > range > index > ALL（ALL 需要优化）
- **key**：实际使用的索引
- **rows**：扫描行数，越少越好
- **Extra**：Using filesort、Using temporary 表示需要优化

**Step 3：针对性优化**

| 问题现象 | 优化方案 |
|---------|---------|
| Using filesort | 避免在 ORDER BY 字句中使用无索引列；增加适当索引 |
| Using temporary | 优化 SQL 逻辑；增加索引覆盖所需列 |
| type=ALL 全表扫描 | 增建索引；调整 WHERE 条件 |
| 回表过多 | 建立覆盖索引；减少 SELECT * |
| 索引失效 | 检查类型转换、函数使用、隐式转换 |

**Step 4：验证优化效果**

使用 `SHOW STATUS` 或压测对比优化前后的 QPS/响应时间。

> 📎 来源：https://www.cnblogs.com/zhiheng/p/12366067.html  
> 📎 来源：https://help.aliyun.com/zh/rds/apsaradb-rds-for-mysql/slow-query-log

---

## 题目三：MySQL 和 PostgreSQL 在索引类型和使用场景上有什么区别？

### 参考答案

**MySQL 索引类型：**
- B-Tree（默认，InnoDB/MyISAM）
- Hash（Memory 引擎，支持等值查询，不支持范围）
- R-Tree（MyISAM，地理空间数据）
- FULLTEXT（全文索引）
- 复合索引（多列）

**PostgreSQL 索引类型：**
- B-Tree（默认，最通用）
- Hash（等值查询，性能优于 MySQL Hash）
- GiST（几何/地理数据、全文搜索）
- GIN（倒排索引，适合数组、JSON、全文搜索）
- BRIN（块范围索引，适合大表顺序扫描）
- 表达式/部分索引（非常强大）

**核心区别与场景：**

| 特性 | MySQL | PostgreSQL |
|-----|-------|-----------|
| JSON 支持 | JSON 函数索引 | 原生 JSONB + GIN 索引，查询性能更强 |
| 全文搜索 | FULLTEXT 索引 | 内置全文搜索 + GIN，分词器更灵活 |
| 部分索引 | ❌ 不支持 | ✅ 支持（WHERE 条件过滤） |
| 表达式索引 | ❌ 不支持 | ✅ 支持（对函数结果建索引） |
| 地理空间 | R-Tree | PostGIS（行业标准 GIS） |
| NULL 值排序 | 支持 | 支持，但实现更标准 |

**选型建议**：
- 传统业务系统、互联网产品后端 → MySQL（生态成熟，分库分表方案完善）
- 数据分析、复杂查询、GIS 应用、JSON 密集型 → PostgreSQL
- 需要强一致性事务 + 复杂数据类型 → PostgreSQL

> 📎 来源：https://www.postgresql.org/docs/current/indexes.html  
> 📎 来源：https://dev.mysql.com/doc/refman/8.0/en/optimization.html

---

## 题目四：如何设计一个高效的数据库缓存策略？Redis + MySQL 缓存一致性问题如何解决？

### 参考答案

**缓存策略设计原则：**

```
读请求 → Cache Hit → 直接返回
         Cache Miss → 查DB → 写Cache → 返回
```

**缓存粒度选择：**
- 缓存行级数据（如用户信息）：精确，内存友好
- 缓存聚合结果（如排行榜）：减少 DB 计算压力

**缓存过期策略：**
- TTL（Time To Live）：设置合理过期时间
- LRU/LFU：内存不足时淘汰策略
- 主动刷新：数据变更时主动更新缓存

---

**缓存一致性问题及解决方案：**

**方案一：Cache Aside（旁路缓存）—— 最常用**

```
读：Cache → 有则返回，无则读DB并写入Cache
写：先更新DB → 再删除Cache（下一次读时再填充）
```

> 注意：写操作是删除而非更新缓存，因为并发场景下更新缓存可能导致脏数据。

**方案二：Read Through / Write Through**

应用不直接操作缓存，而是由缓存层自动读写 DB。实现复杂，一般不采用。

**方案三：延迟双删（解决并发脏读）**

```python
# 线程A：更新DB
db.update(user)
# 线程A：删除Cache
cache.delete(user)
# 延迟一段时间，再删一次（异步）
time.sleep(0.5)
cache.delete(user)
```

**方案四：基于 Canal 监听 MySQL binlog 同步**

```
MySQL → Canal → MQ → 消费 → 更新Redis
```

优势：彻底解耦，DB 变更自动同步缓存。

**最佳实践总结：**
1. 读多写少 → Cache Aside + TTL
2. 写多读少 → 不建议缓存，或缩短 TTL
3. 强一致性要求 → 延迟双删 / Canal 方案
4. 避免缓存穿透 → 空值也缓存（加短TTL）
5. 避免缓存雪崩 → 过期时间加随机值 + 分布式锁

> 📎 来源：https://developer.aliyun.com/article/1312337  
> 📎 来源：https://www.cnblogs.com/throwable/p/15171876.html

---

## 题目五：PostgreSQL 中如何分析并优化大表的分页查询（OFFSET 大导致的性能问题）？

### 参考答案

**问题背景：**

```sql
SELECT * FROM orders ORDER BY id LIMIT 20 OFFSET 1000000;
```

当 OFFSET 很大时，数据库仍然需要扫描并丢弃前100万行数据，效率极低。即使有索引，也只是避免全表扫描，但回表成本仍然很高。

**优化方案：**

**方案一：游标分页（Keyset Pagination）—— 最佳方案**

利用上一页最后一条记录的主键进行过滤：

```sql
-- 第一页
SELECT * FROM orders ORDER BY id LIMIT 20;

-- 下一页：传入上一页最后一条的 id
SELECT * FROM orders 
  WHERE id > :last_id 
  ORDER BY id 
  LIMIT 20;
```

- 时间复杂度 O(1)，不受数据量影响
- 缺点：不能跳页，只能顺序翻页

**方案二：使用 BRIN 索引（适合物理顺序与主键有序的大表）**

```sql
CREATE INDEX idx_orders_id_brin ON orders USING BRIN(id);
-- 对于主键有序的大表，BRIN 索引体积极小，范围查询极快
```

**方案三：覆盖索引 + 子查询**

```sql
SELECT * FROM orders 
  INNER JOIN (
    SELECT id FROM orders ORDER BY id LIMIT 20 OFFSET 1000000
  ) AS t USING (id);
```

让子查询只返回主键列（走覆盖索引），主查询再通过主键回表，减少回表次数。

**方案四：分桶分页（物理分页）**

按时间或业务维度定期归档（如每月一张表），查询时先定位到目标表，再分页。

**方案五：Elasticsearch / ClickHouse 承接搜索和分页**

对于需要复杂条件 + 深分页的查询，引入 ES/ClickHouse 分担 MySQL 压力。

> 📎 来源：https://www.postgresql.org/docs/current/queries-limit.html  
> 📎 来源：https://www.cnblogs.com/winner-715/p/16219797.html

---

*本套题由数据库面试题助手整理 | 每日更新*
