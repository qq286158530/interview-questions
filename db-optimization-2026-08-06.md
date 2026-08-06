# 数据库性能优化面试题

> 📅 日期：2026-08-06
> 🐼 来源：整理自经典面试题库 + 权威技术文档

---

## 题目一：MySQL 查询突然变慢，如何排查？

**🔍 问题描述：**
一条原本执行很快的 SQL，突然变得很慢，可能是什么原因？如何系统性地排查？

**📖 参考答案：**

**1. 是否使用了索引（Explain 分析）**
```sql
EXPLAIN SELECT * FROM orders WHERE user_id = 10086;
```
- 检查 `type` 列：是否为 `ALL`（全表扫描）？
- 检查 `key` 列：是否用上了索引？
- 检查 `rows` 列：扫描行数是否过多？
- 检查 `Extra` 列：是否有 `Using filesort`、`Using temporary` 等警告

**2. 索引失效的常见原因**
- 使用了 `LIKE '%xxx'`（前缀通配符）导致索引失效
- 列做了函数/计算，如 `WHERE YEAR(create_time) = 2024`
- 数据类型不匹配，如字符串字段用数字查询
- 使用了 `OR` 且两侧条件列不在同一索引中
- 隐式类型转换导致索引失效

**3. 表统计信息过期**
```sql
ANALYZE TABLE orders;  -- 重新统计信息
OPTIMIZE TABLE orders; -- 整理碎片（InnoDB）
```

**4. 数据量变化（分页深度问题）**
```sql
-- 低效：偏移量过深时，MySQL 先扫描前面所有行
SELECT * FROM orders LIMIT 1000000, 10;

-- 优化：使用主键延迟关联
SELECT * FROM orders WHERE id > 1000000 ORDER BY id LIMIT 10;
```

**5. 锁等待问题**
```sql
SHOW ENGINE INNODB STATUS;  -- 查看锁等待
SHOW PROCESSLIST;           -- 查看连接状态
```
可能是长事务持有排他锁，导致其他查询等待。

**6. 其他常见原因**
- 服务器参数配置变化（buffer pool、tmp_table_size 等）
- 缓存失效（如 Buffer Pool 被其他大查询占满）
- 网络延迟（针对远程数据库）
- 表结构变更（删减字段、修改类型）

**✅ 排查步骤总结：**
```
EXPLAIN → 索引分析 → 统计信息 → 锁等待 → 服务器参数 → 业务逻辑
```

---

## 题目二：InnoDB 与 MyISAM 的核心区别？何时选 MyISAM？

**🔍 问题描述：**
InnoDB 和 MyISAM 有什么区别？在实际项目中如何选择？

**📖 参考答案：**

| 特性 | InnoDB | MyISAM |
|------|--------|--------|
| **事务支持** | ✅ 支持 ACID 事务 | ❌ 不支持 |
| **行级锁** | ✅ 支持行锁，并发好 | ❌ 只支持表锁 |
| **外键约束** | ✅ 支持 | ❌ 不支持 |
| **崩溃恢复** | ✅ 自动恢复（redo log） | ❌ 需手动修复 |
| **MVCC** | ✅ 支持，高并发 | ❌ 不支持 |
| **全文索引** | ✅ 5.6+ 支持 | ✅ 原生支持（早期优势） |
| **COUNT(*)** | 全表扫描，较慢 | 独立存储，很快 |
| **存储结构** | 簇索引（聚簇索引） | 堆表（非聚簇索引） |

**InnoDB 的核心优势：**
- **聚簇索引**：主键和数据行在一起，叶子节点存储完整数据，减少 I/O
- **MVCC（多版本并发控制）**：读写不互斥，支持更高的并发
- **自适应哈希索引**：InnoDB 会自动为热点数据建立哈希索引加速查询
- **插入缓冲（Insert Buffer）**：对非唯一索引的插入进行批量合并，减少随机 I/O

**何时可以用 MyISAM？**
- **只读/静态表**：数据不频繁更新，不需要事务
- **全文搜索场景**：MyISAM 的全文索引在 5.6 之前是唯一选择
- **COUNT(*) 频繁**：MyISAM 将行数存储在元数据中，`SELECT COUNT(*)` 极快
- **历史遗留系统**：老项目迁移成本高，暂保留

**⚠️ 注意：** MySQL 8.0 已移除 MyISAM，强烈建议使用 InnoDB。

**来源：** [MySQL 官方文档 - InnoDB 特性](https://dev.mysql.com/doc/refman/8.0/en/innodb-introduction.html)

---

## 题目三：PostgreSQL 如何定位和优化慢查询？

**🔍 问题描述：**
PostgreSQL 数据库中有慢查询，如何定位瓶颈并优化？

**📖 参考答案：**

**1. 开启慢查询日志**
```sql
-- 方法一：修改配置 postgresql.conf
log_min_duration_statement = 1000  -- 记录超过 1000ms 的查询

-- 方法二：运行时修改（临时生效）
ALTER SYSTEM SET log_min_duration_statement = 1000;
SELECT pg_reload_conf();
```

**2. 使用 pg_stat_statements（推荐）**
```sql
-- 安装扩展
CREATE EXTENSION pg_stat_statements;

-- 查询最慢的 SQL（按总耗时排序）
SELECT query, calls, total_exec_time, mean_exec_time, rows
FROM pg_stat_statements
ORDER BY total_exec_time DESC
LIMIT 10;

-- 查询调用次数最多
SELECT query, calls, mean_exec_time
FROM pg_stat_statements
ORDER BY calls DESC
LIMIT 10;
```

**3. 使用 EXPLAIN ANALYZE 分析执行计划**
```sql
EXPLAIN (ANALYZE, BUFFERS, FORMAT TEXT)
SELECT * FROM orders WHERE user_id = 10086;
```
- `Seq Scan`：全表扫描，考虑加索引
- `Hash Join` / `Nested Loop`：关注连接方式是否合理
- `BUFFERS`：命中缓存越多越好，`shared hit` 为缓存命中，`read` 为磁盘读取

**4. 常见优化手段**
- **创建合适的索引**：
  ```sql
  -- 单列索引
  CREATE INDEX idx_user_id ON orders(user_id);
  
  -- 联合索引（遵循最左前缀原则）
  CREATE INDEX idx_user_status ON orders(user_id, status);
  
  -- 表达式索引（避免函数导致索引失效）
  CREATE INDEX idx_created ON orders(DATE(create_time));
  ```

- **使用覆盖索引避免回表**：
  ```sql
  -- 查询字段都在索引中，直接返回，无需回表
  CREATE INDEX idx_covering ON orders(user_id) INCLUDE (status, amount);
  ```

- **分区表**（对大表按时间/地区拆分）：
  ```sql
  CREATE TABLE orders (
      id BIGSERIAL,
      create_time TIMESTAMP,
      amount DECIMAL(10,2)
  ) PARTITION BY RANGE (create_time);
  ```

**5. 使用 VACUUM 和 ANALYZE 维护统计信息**
```sql
VACUUM ANALYZE orders;  -- 清理死元组 + 更新统计信息
```

**6. 配置参数调优**
```sql
-- 共享缓冲区（建议 OS 内存的 25%）
shared_buffers = 4GB

-- 有效缓存大小（影响查询规划）
effective_cache_size = 12GB

-- 工作内存（排序/哈希操作）
work_mem = 64MB
```

**来源：** [PostgreSQL 官方文档 - 使用 EXPLAIN](https://www.postgresql.org/docs/current/using-explain.html)

---

## 题目四：数据库缓存策略——Redis + MySQL 如何配合使用？

**🔍 问题描述：**
在高并发系统中，Redis 和 MySQL 通常如何配合工作？有哪些经典模式？

**📖 参考答案：**

**经典模式：Cache-Aside（旁路缓存）**

```
读：先查 Redis，命中则返回；未命中查 MySQL，写入 Redis 后返回
写：先更新 MySQL，再删除（而非更新）Redis 缓存
```

**读操作：**
```python
def get_user(user_id):
    user = redis.get(f"user:{user_id}")
    if user:
        return json.loads(user)
    
    user = db.query("SELECT * FROM users WHERE id = %s", user_id)
    redis.setex(f"user:{user_id}", 3600, json.dumps(user))  # TTL 1小时
    return user
```

**写操作（先更新 DB，再删缓存）：**
```python
def update_user(user_id, data):
    db.execute("UPDATE users SET ... WHERE id = %s", user_id)
    redis.delete(f"user:{user_id}")  # 删除缓存而非更新
```

**为什么删除而不是更新？**
- 更新缓存可能导致并发问题：线程 A 写 DB → 线程 B 写 DB → 线程 B 写缓存 → 线程 A 写缓存（脏数据）
- 删除缓存后，下次查询会从 DB 加载最新数据，逻辑更简单

**三种缓存模式对比：**

| 模式 | 读 | 写 | 优点 | 缺点 |
|------|----|----|------|------|
| **Cache-Aside** | 应用主导 | 应用主导 | 简单，通用 | 首次miss有穿透 |
| **Read-Through** | 缓存自动加载 | — | 应用简单 | 缓存需要知道如何加载 |
| **Write-Through** | — | 缓存同步更新 | 数据一致性好 | 写延迟增加 |

**缓存常见问题及解决：**

1. **缓存雪崩**：大量 key 同时过期
   - 解决：给过期时间加随机偏移 `TTL + random(0, 300)`

2. **缓存击穿**：热点 key 过期，瞬间大量请求打到 DB
   - 解决：互斥锁 / 永不过期 + 异步更新

3. **缓存穿透**：查询不存在的数据，每次都穿透到 DB
   - 解决：布隆过滤器 / 空值缓存

4. **数据一致性问题**：DB 更新成功，缓存删除失败
   - 解决：重试机制 / 订阅 MySQL binlog 异步更新（canal）

**来源：** [Redis 官方文档 - Data Types](https://redis.io/docs/data-types/)

---

## 题目五：MySQL 主从复制延迟的原因及解决方案？

**🔍 问题描述：**
MySQL 主从架构中，从库延迟越来越严重，可能有哪些原因？如何解决？

**📖 参考答案：**

**主从复制原理回顾：**
```
主库：事务提交 → 写入 binlog → 传输给从库 → 从库写入 relay log → 重放执行
                ↑                                    ↑
            Dump 线程 ←——————————binlog ———————————— I/O 线程
                                                  ↑ 
                                           SQL 线程（重放）
```

**常见延迟原因：**

**1. 大事务导致从库重放时间长**
```sql
-- 反面例子：一个事务中插入 10 万条数据
BEGIN;
INSERT INTO orders (...) VALUES (...);  -- 10万次
COMMIT;  -- 主库很快完成，从库要重放很久
```
- 解决：拆分为小事务，减少单次事务数据量

**2. 从库服务器性能更弱**
- 从库不仅执行查询，还要承担 I/O 线程和 SQL 线程的工作
- 解决：从库规格应 >= 主库，或将读压力分散到多个从库

**3. 从库索引缺失**
- 从库查询走了全表扫描，而主库有索引
- 解决：从库建立与主库相同的索引

**4. 网络延迟（大事务传输）**
- binlog 传输占用主从网络带宽
- 解决：使用 `group_commit` + 压缩 `binlog_transaction_compression`

**5. 从库配置了 `log_slave_updates = ON`**
- 从库同时作为其他从库的主库时，需要记录 binlog，I/O 压力大
- 评估是否必要，如无必要可关闭

**6. 大量 `SELECT` 查询在从库运行**
- 从库同时承担报表、大查询等操作
- 解决：读写分离 + 专用分析从库

**监控延迟：**
```sql
SHOW SLAVE STATUS\G;
-- 查看 Seconds_Behind_Master：延迟秒数
-- 查看 Relay_Log_Pos：重放位置

-- 或使用 pt-heartbeat 工具精确测量
pt-heartbeat -D test --update -h master-server
pt-heartbeat -D test --check -h slave-server
```

**从根本上解决延迟：**
- **MySQL 8.0 MGR（Group Replication）**：多主模式，更低的复制延迟
- **ShardingSphere / Vitess**：分库分表，从根本上减少单库数据量
- **并行复制`：MySQL 5.7+ 支持基于组提交的并行复制，显著降低延迟

**来源：** [MySQL 官方文档 - Replication](https://dev.mysql.com/doc/refman/8.0/en/replication.html)

---

## 📚 更多学习资源

- [MySQL 8.0 Reference Manual](https://dev.mysql.com/doc/refman/8.0/en/)
- [PostgreSQL 16 Documentation](https://www.postgresql.org/docs/16/)
- [Redis 官方文档](https://redis.io/docs/)
- [Percona Toolkit（MySQL 性能诊断工具）](https://www.percona.com/software/percona-toolkit)
