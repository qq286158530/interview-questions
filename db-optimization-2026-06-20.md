# 数据库性能优化面试题

> 📅 更新时间：2026-06-20
> 📚 涵盖：MySQL 索引优化、查询优化、存储引擎、缓存策略、PostgreSQL 优化

---

## 题目 1：MySQL 中 B+ 树索引的工作原理是什么？为什么比 B 树更适合数据库？

### 答案

**B+ 树索引的工作原理：**

B+ 树是一种多路平衡查找树，所有数据都存储在叶子节点，叶子节点之间通过双向链表连接。

1. **查找过程**：从根节点开始，根据索引列的值逐层向下比较，找到对应的叶子节点，然后在叶子节点的双向链表中进行范围查找。

2. **插入过程**：找到对应的叶子节点页，如果页满则分裂，向上层追加索引记录。

3. **删除过程**：如果页内数据过少，会与相邻页合并。

**B+ 树 vs B 树 —— 为什么 B+ 树更适合数据库：**

| 特性 | B+ 树 | B 树 |
|------|-------|------|
| 数据存放位置 | 仅叶子节点 | 所有节点 |
| 树高度 | 更低（通常 3-4 层） | 相对更高 |
| 范围查询 | 叶子链表 O(1) | 需要中序遍历 |
| 查询稳定性 | 所有查询路径长度相同 | 不稳定 |
| 磁盘读写 | 非叶子节点不存储数据，单页能容纳更多索引项 | 每个节点都存数据 |

**核心原因**：
- **磁盘友好**：非叶子节点不存储数据，每个磁盘页能容纳更多索引项，树高更矮，IO 次数更少
- **范围查询高效**：叶子节点链表连接，适合 `>`、`<`、`BETWEEN`、`LIKE` 等范围查询
- **查询稳定**：所有数据查询都需要走到叶子节点，无退化风险

---

## 题目 2：如何优化慢查询？请描述完整的分析和优化流程。

### 答案

**完整流程：**

### 第一步：定位慢查询

```sql
-- 开启慢查询日志
SET GLOBAL slow_query_log = 'ON';
SET GLOBAL long_query_time = 1;  -- 超过 1 秒记录
SET GLOBAL slow_query_log_file = '/var/log/mysql/slow.log';

-- 查看慢查询日志
SHOW VARIABLES LIKE 'slow_query%';

-- 使用 pt-query-digest 分析慢查询日志
pt-query-digest /var/log/mysql/slow.log
```

### 第二步：使用 EXPLAIN 分析执行计划

```sql
EXPLAIN SELECT * FROM orders WHERE status = 'paid' AND create_time > '2026-01-01';
```

重点关注字段：
- **type**：最好达到 `ref` 或 `range`，避免 `ALL`（全表扫描）
- **key**：实际使用的索引
- **rows**：扫描行数，越少越好
- **Extra**：
  - `Using filesort`：需要额外排序，性能差
  - `Using temporary`：使用了临时表，性能差
  - `Using index`：覆盖索引，无需回表

### 第三步：针对性优化

1. **索引优化**
   - 为 WHERE、ORDER BY、GROUP BY 涉及的列建索引
   - 遵循最左前缀原则
   - 避免在索引列上使用函数或类型转换

2. **SQL 语句优化**
   - 避免 `SELECT *`，只查需要的列
   - 用 `EXISTS` 替代 `IN`，用 `JOIN` 替代子查询
   - 分解大查询，小查询优于大查询

3. **表结构优化**
   - 适当使用冗余字段减少 JOIN
   - 避免 null 值（InnoDB 对 null 索引支持不完善）
   - 适度拆分大表（分库分表）

4. **业务层优化**
   - 添加 Redis 缓存热点数据
   - 读写分离，主从分流
   - 使用 ES 处理复杂检索

---

## 题目 3：InnoDB 和 MyISAM 存储引擎的区别是什么？如何选择？

### 答案

| 特性 | InnoDB | MyISAM |
|------|--------|--------|
| **事务支持** | 支持 ACID 事务 | 不支持 |
| **锁粒度** | 行级锁 + 间隙锁 | 表级锁 |
| **MVCC** | 支持（多版本并发控制） | 不支持 |
| **外键** | 支持 | 不支持 |
| **崩溃恢复** | 自动恢复（redo log） | 较差（myisamchk） |
| **索引** | Clustered Index（主键索引） | Non-Clustered Index |
| **全文索引** | 5.6+ 支持 | 原生支持 |
| **COUNT(*)** | 全表扫描（无缓存） | 有专门计数器，快 |
| **适用场景** | 写入多、并发高、需要事务 | 读多写少、不需要事务 |

**InnoDB 的核心优势：**
- **行级锁 + MVCC** → 高并发写入/读取
- **Clustered Index** → 主键查询极快（数据和索引一起存）
- **Crash Safe** → 异常崩溃后自动恢复

**如何选择：**
- **选 InnoDB**：生产环境默认选它，适合 OLTP 场景
- **选 MyISAM**：日志表、只读静态表、需全文检索的旧系统

**补充**：MySQL 8.0+ 已移除 MyISAM，建议使用 InnoDB + 全文索引。

---

## 题目 4：什么是数据库缓存策略？Redis 和 MySQL 如何配合使用？

### 答案

**缓存策略核心思想**：将热点数据从内存读取，减少数据库磁盘 IO。

**典型架构：**
```
客户端 → Redis（缓存） → MySQL（持久化）
              ↓ 命中
           直接返回
```

**缓存读写模式：**

### 模式一：Cache-Aside（旁路缓存，最常用）

```python
# 读
def get_user(user_id):
    user = redis.get(f"user:{user_id}")
    if user:
        return json.loads(user)
    user = mysql.query("SELECT * FROM users WHERE id = %s", user_id)
    redis.setex(f"user:{user_id}", 3600, json.dumps(user))  # TTL 1小时
    return user

# 写
def update_user(user_id, data):
    mysql.execute("UPDATE users SET ... WHERE id = %s", user_id)
    redis.delete(f"user:{user_id}")  # 删除缓存，而非更新
```

### 模式二：Read-Through / Write-Through

缓存自动加载/写入，对应用透明。

**缓存问题及解决方案：**

| 问题 | 描述 | 解决方案 |
|------|------|----------|
| **缓存穿透** | 查询不存在的数据，每次都打满 DB | 布隆过滤器 / 缓存空值 |
| **缓存击穿** | 热点 key 过期，瞬间大量请求打 DB | 互斥锁 / 热点永不过期 |
| **缓存雪崩** | 大量 key 同时过期 | 随机 TTL / 多级缓存 |

**Redis 缓存失效策略：**
- `volatile-lru`：过期键中选最近最少使用
- `allkeys-lru`：所有键中选最近最少使用
- `no-eviction`：禁止驱逐（返回错误）

**缓存一致性维护：**
- 先更新数据库，再删除缓存（延迟双删）
- 分布式锁保证原子性

---

## 题目 5：PostgreSQL 相比 MySQL 在性能优化上有哪些独特优势？如何进行优化？

### 答案

**PostgreSQL 的独特优势：**

| 特性 | PostgreSQL | MySQL |
|------|-----------|-------|
| **MVCC 实现** | 真正的多版本，无读锁 | 依赖 InnoDB，介于之间 |
| **索引类型** | B-tree、Hash、GIN、GiST、BRIN、覆盖索引 | B-tree、全文、GIS |
| **并行查询** | 支持并行_seq_scan、并行聚合 | 8.0+ 才支持 |
| **JSON 支持** | 原生 JSONB（Binary），高性能 | 5.7+ JSON 函数，较弱 |
| **分区表** | 声明式分区，灵活 | 5.7+ 支持，语法较复杂 |
| **物化视图** | 支持，缓存复杂查询结果 | 不支持 |
| **CTE / 递归查询** | 完整支持 | 8.0+ 部分支持 |

**PostgreSQL 核心优化手段：**

### 1. 索引优化
```sql
-- 表达式索引
CREATE INDEX idx_lower_email ON users(lower(email));

-- 部分索引（只索引满足条件的行）
CREATE INDEX idx_active_users ON users(created_at) WHERE is_active = true;

-- 覆盖索引（INCLUDE）
CREATE INDEX idx_cover ON orders(user_id) INCLUDE (amount, status);

-- GIN 索引（全文搜索、JSONB）
CREATE INDEX idx_gin_data ON products USING gin(data jsonb_path_ops);
```

### 2. EXPLAIN ANALYZE 分析执行计划
```sql
EXPLAIN (ANALYZE, BUFFERS, FORMAT TEXT)
SELECT * FROM orders WHERE status = 'shipped' AND created_at > '2026-01-01';
```
重点关注：`Seq Scan`（顺序扫描）过多 → 考虑建索引。

### 3. 统计信息维护
```sql
-- 手动更新统计信息（大批量数据变更后）
ANALYZE orders;

-- 查看表统计信息
SELECT schemaname, tablename, n_live_tup, n_dead_tup FROM pg_stat_user_tables;
```

### 4. 批量插入优化
```sql
-- 使用 COPY 替代 INSERT（性能提升 10 倍+）
COPY orders FROM '/tmp/orders.csv' WITH (FORMAT csv);

-- 批量 INSERT 合并
INSERT INTO orders (id, user_id, amount) VALUES 
  (1, 100, 50.0), (2, 101, 60.0), (3, 102, 70.0);
```

### 5. 连接池配置（PgBouncer）
```ini
; pgbouncer.ini
pool_mode = transaction
max_client_conn = 1000
default_pool_size = 20
```

### 6. 监控慢查询
```sql
-- 开启 track_activities 和 log_min_duration_statement
ALTER SYSTEM SET log_min_duration_statement = 1000;  -- 记录超过 1s 的查询
```

---

## 📚 参考来源

1. **小林 Coding**：https://xiaolincoding.com/mysql/ （MySQL 进阶首选）
2. **GitHub 面试题汇总**：https://github.com/0voice/interview_internal_reference
3. **MySQL 官方文档**：https://dev.mysql.com/doc/refman/8.0/en/innodb-storage-engine.html
4. **PostgreSQL 官方文档**：https://www.postgresql.org/docs/current/index.html
5. **Redis 设计与实现**：https://github.com/huangz1990/redis-3.0-annotated

---

*本文件由 AI 自动生成并推送至 GitHub 仓库*
