# 数据库性能优化面试题

> 📅 更新时间：2026-05-14
> 🏷️ 分类：MySQL / PostgreSQL / 数据库原理

---

## 题目一：MySQL 索引失效的场景有哪些？如何避免？

### 参考答案

索引失效是开发中最容易遇到的性能问题，以下是常见的 7 种场景：

**1. 使用函数或运算**
```sql
-- 失效：WHERE YEAR(created_at) = 2026
-- 正确：WHERE created_at >= '2026-01-01' AND created_at < '2027-01-01'
```

**2. 左模糊匹配 (LIKE '%xxx')**
```sql
-- 失效：WHERE name LIKE '%张'
-- 正确：WHERE name LIKE '张%'（可以使用索引）
```

**3. 隐式类型转换**
```sql
-- 失效：phone 是 VARCHAR，WHERE phone = 13800138000（数字）
-- 正确：WHERE phone = '13800138000'
```

**4. 联合索引不符合最左前缀原则**
```sql
-- 创建：INDEX idx(a, b, c)
-- 失效：WHERE b = ? 或 WHERE c = ?
-- 生效：WHERE a = ? / WHERE a = ? AND b = ? / WHERE a = ? AND b = ? AND c = ?
```

**5. 使用 OR 连接非索引列**
```sql
-- 失效：WHERE name = '张三' OR age = 20（age 没有索引）
-- 改善：拆分为两条 SQL + UNION，或者为 age 创建索引
```

**6. 范围查询右边的列无法使用索引**
```sql
-- 创建：INDEX idx(name, age)
-- 失效：WHERE name = '张三' AND age > 18
-- 说明：age 的索引在 name 范围条件右边，无法使用
```

**7. IS NOT NULL / IS NULL**
```sql
-- 可能失效：取决于数据分布和优化器判断
-- 注意：NOT NULL 限制有助于索引使用
```

### 避坑建议
- 用 `EXPLAIN` 检查执行计划，确认索引是否被使用
- 避免在 WHERE 中对索引列做计算或函数处理
- 设计联合索引时考虑查询组合，合理安排列顺序

---

**📎 来源：**
- [MySQL 索引详解 | 菜鸟教程](https://www.runoob.com/mysql/mysql-index.html)
- [MySQL 官方文档 - Optimization](https://dev.mysql.com/doc/refman/8.0/en/optimization.html)

---

## 题目二：如何定位和解决慢查询？请描述完整的流程。

### 参考答案

**Step 1：开启慢查询日志**
```sql
-- 查看慢查询配置
SHOW VARIABLES LIKE 'slow_query%';
SHOW VARIABLES LIKE 'long_query_time';

-- 开启（临时）
SET GLOBAL slow_query_log = 'ON';
SET GLOBAL long_query_time = 1; -- 超过 1 秒记录

-- 永久配置需修改 my.cnf
```

**Step 2：分析慢查询日志**
```bash
# 使用 mysqldumpslow 工具汇总
mysqldumpslow -t 5 /var/lib/mysql/slow.log
```

**Step 3：EXPLAIN 分析执行计划**
```sql
EXPLAIN SELECT ... FROM users WHERE ...
-- 重点关注：
-- type: const > eq_ref > ref > range（尽量避免 ALL 全表扫描）
-- key: 实际使用的索引
-- rows: 扫描行数（越大越慢）
-- Extra: Using filesort / Using temporary（需要优化）
```

**Step 4：使用 SHOW PROCESSLIST 实时排查**
```sql
SHOW PROCESSLIST;
-- 找到长时间运行的 SQL 的 ID
KILL <id>;  -- 终止问题查询
```

**Step 5：优化方案**

| 问题类型 | 优化方法 |
|---------|---------|
| 全表扫描 | 添加合适索引 |
| 深分页 | 改用游标分页：`WHERE id > last_id` |
| 低选择性索引 | 考虑联合索引或覆盖索引 |
| 子查询 | 改写为 JOIN |
| 大量排序 | 减少 ORDER BY 字段或增加 sort_buffer_size |

**Step 6：验证优化效果**
```sql
-- 对比优化前后的执行时间和 EXPLAIN 结果
```

### 工具推荐
- **MySQL Enterprise Monitor**：商业版监控
- **Percona Toolkit**：开源慢查询分析工具
- **Performance Schema**：MySQL 5.6+ 内置性能分析

---

**📎 来源：**
- [MySQL 慢查询日志配置 | 官方文档](https://dev.mysql.com/doc/refman/8.0/en/slow-query-log.html)
- [PostgreSQL Performance Tips | 官方文档](https://www.postgresql.org/docs/current/performance-tips.html)

---

## 题目三：MySQL InnoDB 和 MyISAM 存储引擎的区别？如何选择？

### 参考答案

| 特性 | InnoDB | MyISAM |
|-----|--------|--------|
| 事务支持 | ✅ 支持 ACID 事务 | ❌ 不支持 |
| 行级锁 | ✅ 支持，高并发好 | ❌ 仅表级锁 |
| 外键约束 | ✅ 支持 | ❌ 不支持 |
| 崩溃恢复 | ✅ 自动恢复（redo log） | ❌ 需手动修复 |
| 全文索引 | ✅ 5.6+ 支持 | ✅ 支持 |
| 存储空间 | 较大（支持行压缩） | 较小（紧凑存储） |
| 适用场景 | 写并发高、需要事务 | 读为主、少量写入 |

### 核心区别详解

**1. 事务与锁机制**
```sql
-- InnoDB 支持行锁 + MVCC
BEGIN;
UPDATE users SET age = 30 WHERE id = 1;  -- 只锁这一行
COMMIT;  -- 其他事务可正常读写其他行

-- MyISAM 表锁，写入会阻塞所有读写
```

**2. 崩溃恢复**
```sql
-- InnoDB 使用 redo log 实现持久性
-- 事务提交 → 写入 redo log → 刷盘
-- 即使宕机，重启后自动恢复未提交的事务
```

**3. 索引实现**
```sql
-- InnoDB：聚簇索引（数据文件和主键索引绑一起）
-- MyISAM：非聚簇索引（索引和数据文件分离）
-- InnoDB 的主键查询直接定位数据，辅助索引存储主键值
```

### 选型建议

**选 InnoDB**：99% 的场景，尤其：
- 订单、交易、金融等需要事务的场景
- 高并发写入场景
- 需要崩溃自动恢复
- 有外键约束需求

**选 MyISAM**：特殊场景：
- 日志型系统，纯append操作
- 数据仓库的只读报表
- 表空间小的场景（如嵌入式设备）

### 注意
MySQL 8.0 已将 MyISAM 标记为废弃，强烈建议所有场景使用 InnoDB。

---

**📎 来源：**
- [InnoDB 简介 | MySQL 官方文档](https://dev.mysql.com/doc/refman/8.0/en/innodb-introduction.html)
- [MySQL 存储引擎对比 | 菜鸟教程](https://www.runoob.com/mysql/mysql-storage-engines.html)

---

## 题目四：PostgreSQL 的 MVCC 机制是什么？它如何提升并发性能？

### 参考答案

### MVCC 核心概念

MVCC（Multi-Version Concurrency Control，多版本并发控制）让数据库**同一时刻可以存在多个数据版本**，读写之间互不阻塞，大幅提升并发性能。

### PostgreSQL 实现方式

**每个事务有自己的「快照」（Snapshot）**

```sql
-- 事务 A 开启（id=100）
BEGIN;
SELECT * FROM users WHERE id = 1;  -- 看到的是提交过的数据版本

-- 事务 B 同时修改 id=1 的行 age=30 并提交

-- 事务 A 再次查询，看到的还是旧值（事务 A 的快照没变）
SELECT * FROM users WHERE id = 1;  -- 仍然是旧值
```

**关键实现：Tuple 的隐藏列**

| 隐藏字段 | 说明 |
|---------|------|
| xmin | 插入该行的事务 ID |
| xmax | 删除/更新该行的事务 ID |
| cmin/cmax | 命令 ID（事务内） |
| t_xmax = 0 + t_xmin = 提交事务 | 表示该行有效 |

**事务可见性判断规则**
```
当 t_xmin = T 且 t_xmax = 0：
  → 该行对事务 T 可见（已提交且未删除）
  
当 t_xmax ≠ 0：
  → 如果 t_xmax 未提交，对 T 可见（仍用旧版本）
  → 如果 t_xmax 已提交，对 T 不可见（已更新/删除）
```

### MVCC 的优势

**1. 读写不阻塞**
```sql
-- 事务 A 读取数据时，事务 B 可以同时写入
-- 读操作不会被写锁阻塞（快照读）
```

**2. 避免脏读和不可重复读**
```
通过快照隔离级别（Snapshot Isolation），防止脏读和不可重复读
```

**3. 写入不阻塞读**
```
使用 UPDATE 创建新版本（行头指针更新），而非原地修改
```

### MVCC 的代价

| 代价 | 说明 |
|-----|------|
| 存储开销 | 多版本数据占用更多磁盘空间 |
| 清理开销 | 需要定期 VACUUM 清理过期版本 |
| 索引维护 | 更新需要维护所有相关索引 |

### 性能调优建议

```sql
-- 监控死亡元组
SELECT * FROM pg_stat_database;

-- 手动 VACUUM 清理（不影响性能）
VACUUM ANALYZE users;

-- 避免 VACUUM 阻塞：使用 VACUUM (VACUUM)
VACUUM (ANALYZE, VERBOSE) users;

-- 配置自动清理（建议开启）
alter system set autovacuum = on;
```

### 与 MySQL InnoDB MVCC 对比

| 方面 | PostgreSQL | MySQL InnoDB |
|-----|-----------|-------------|
| 实现方式 | 每个事务快照，读不阻塞写 | 乐观锁 + Undo 段 |
| 隔离级别 | SI 为默认，支持 SERIALIZABLE | RC/RR 为默认 |
| 垃圾回收 | VACUUM 进程自动清理 | InnoDB purge 线程 |

---

**📎 来源：**
- [PostgreSQL MVCC 机制 | 官方文档](https://www.postgresql.org/docs/current/mvcc-intro.html)
- [PostgreSQL Performance Tips | 官方文档](https://www.postgresql.org/docs/current/performance-tips.html)

---

## 题目五：如何设计数据库缓存策略？Redis 与 MySQL 如何配合使用？

### 参考答案

### 缓存设计原则

**Cache Aside（旁路缓存）是最常用的模式**

```
读取：先查缓存 → 命中则返回 → 未命中则查 DB → 写入缓存 → 返回
写入：先更新 DB → 删除缓存（而非更新缓存）
```

### 完整实现

**1. 读操作**
```python
def get_user(user_id):
    # 1. 查 Redis
    cache_key = f"user:{user_id}"
    user = redis.get(cache_key)
    if user:
        return json.loads(user)
    
    # 2. 缓存未命中，查数据库
    user = db.query("SELECT * FROM users WHERE id = %s", user_id)
    
    # 3. 写入缓存，设置过期时间
    redis.setex(cache_key, 3600, json.dumps(user))  # 1小时过期
    return user
```

**2. 写操作（先更新DB，再删缓存）**
```python
def update_user(user_id, data):
    # 1. 更新数据库
    db.execute("UPDATE users SET name=%s WHERE id=%s", data['name'], user_id)
    
    # 2. 删除缓存（不是更新！）
    redis.delete(f"user:{user_id}")
```

**为什么删除而不是更新？**
- 避免并发写入时缓存和 DB 数据不一致
- 删除操作幂等，更新不幂等

### 缓存问题处理

**1. 缓存穿透（查询不存在的数据）**
```python
# 解决方案：对不存在的数据也缓存一个短过期时间的空值
if user is None:
    redis.setex(cache_key, 60, "NULL")  # 60秒内不再查 DB
```

**2. 缓存击穿（热点 key 过期，大量请求击穿到 DB）**
```python
# 解决方案：使用分布式锁，只允许一个请求查 DB 并写缓存
lock = redis.setnx(f"lock:{cache_key}", "1")
if lock:
    user = db.query(...)
    redis.setex(cache_key, 3600, json.dumps(user))
    redis.delete(f"lock:{cache_key}")
```

**3. 缓存雪崩（大量 key 同时过期）**
```python
# 解决方案：过期时间加随机偏移量
expire_time = 3600 + random.randint(0, 300)  # 1小时~1小时5分钟
redis.setex(cache_key, expire_time, json.dumps(user))
```

### 缓存与数据库一致性

| 方案 | 适用场景 | 代价 |
|-----|---------|-----|
| Cache Aside（删除缓存） | 读多写少 | 可能短暂不一致 |
| 延迟双删 | 写操作较多 | 多一次延迟删除 |
| 更新 DB + 更新缓存 | 强一致性要求 | 复杂，有并发问题 |

**延迟双删（推荐）**
```python
def update_user(user_id, data):
    # 1. 删除缓存
    redis.delete(f"user:{user_id}")
    
    # 2. 更新数据库
    db.execute("UPDATE users SET ... WHERE id=%s", user_id)
    
    # 3. 延迟 500ms 再次删除（处理并发：先删的请求被旧数据重新缓存）
    time.sleep(0.5)
    redis.delete(f"user:{user_id}")
```

### Redis 配置建议

```bash
# 内存策略：当内存满时删除数据
maxmemory-policy allkeys-lru  # LRU 淘汰，保留热点数据

# 持久化：混合 RDB + AOF
appendonly yes
appendfsync everysec  # 每秒刷盘，兼顾性能和数据安全
```

### 监控指标

```
Redis：内存使用率、hit rate（命中率）、连接数
DB：QPS、响应时间、缓存穿透率
```

---

**📎 来源：**
- [Redis Documentation | 官方文档](https://redis.io/documentation)
- [MySQL 官方文档 - Query Optimization](https://dev.mysql.com/doc/refman/8.0/en/optimization.html)

---

> 📚 更多面试题请访问：[小林coding](https://xiaolincoding.com)、[牛客网面试题库](https://www.nowcoder.com)