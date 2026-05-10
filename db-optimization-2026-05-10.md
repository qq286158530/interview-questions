# 数据库性能优化面试题

> 📅 日期：2026-05-10  
> 🏷️ 标签：MySQL | PostgreSQL | 性能优化 | 索引 | 查询优化  
> 📚 来源：综合整理自网络经典面试题

---

## 题目一：索引失效的场景有哪些？如何避免？

### 参考答案

**常见的索引失效场景：**

1. **使用函数或计算**
   ```sql
   -- 错误示例：索引列参与运算
   SELECT * FROM orders WHERE YEAR(create_time) = 2026;
   
   -- 正确示例：保留索引列的独立性
   SELECT * FROM orders WHERE create_time >= '2026-01-01' AND create_time < '2027-01-01';
   ```

2. **隐式类型转换**
   ```sql
   -- phone 是 varchar 类型，但传入数字
   SELECT * FROM users WHERE phone = 13800138000; -- 触发隐式转换，索引失效
   ```

3. **前导模糊查询**
   ```sql
   -- 错误示例：%开头导致全表扫描
   SELECT * FROM users WHERE name LIKE '%张%';
   
   -- 正确示例：可以用全文索引或考虑 ElasticSearch
   SELECT * FROM users WHERE name LIKE '张%'; -- 索引仍可用
   ```

4. **OR 连接条件**
   ```sql
   -- OR 导致索引失效（除非两边都有索引且类型一致）
   SELECT * FROM users WHERE name = '张三' OR age = 25;
   -- 建议：拆分为 UNION 或确保 OR 两边都有合适索引
   ```

5. **不符合最左前缀原则**
   ```sql
   -- 联合索引 (a, b, c)，下面的查询无法使用索引
   WHERE b = 1  -- 跳过了 a
   WHERE c = 1  -- 跳过了 a 和 b
   ```

6. **IN 子句包含过多值**
   - IN 中数据量过大时，MySQL 可能放弃索引而选择全表扫描

**避免策略：**
- EXPLAIN 分析执行计划，关注 type、key、rows
- 确保字段类型匹配，避免隐式转换
- 避免在 WHERE 中对索引列使用函数
- 设计索引时考虑查询模式，遵循最左前缀原则

---

## 题目二：Explain 执行计划中各个字段的含义是什么？

### 参考答案

```sql
EXPLAIN SELECT * FROM orders WHERE user_id = 100;
```

| 字段 | 含义 | 正常值参考 |
|------|------|------------|
| **id** | SELECT 查询的编号，id 越大越先执行 | 数字 |
| **select_type** | 查询类型 | SIMPLE（简单查询）、PRIMARY（最外层）、SUBQUERY（子查询）、DERIVED（派生表） |
| **type** | 访问方式，越靠前越好 | `const > eq_ref > ref > range > index > ALL`（从左到右性能递减） |
| **possible_keys** | 可能使用的索引 | 显示可供选择的索引列表 |
| **key** | 实际使用的索引 | NULL 表示没用索引 |
| **key_len** | 索引使用的字节数 | 越大说明索引覆盖越完整 |
| **rows** | 估算需要扫描的行数 | 越小越好 |
| **Extra** | 附加信息 | Using index（覆盖索引）、Using filesort（需额外排序）、Using temporary（需临时表） |

**重点关注：**
- **type 为 ALL**：`SELECT *` 且无 WHERE 条件，需要优化
- **key 为 NULL**：没有索引，需要考虑加索引
- **Extra 包含 Using filesort**：需要额外排序，性能差，建议优化
- **Extra 包含 Using temporary**：使用了临时表，考虑优化查询或增加索引

---

## 题目三：MySQL InnoDB 和 MyISAM 存储引擎的区别？如何选择？

### 参考答案

| 特性 | InnoDB | MyISAM |
|------|--------|--------|
| **事务支持** | ✅ 支持事务（ACID） | ❌ 不支持 |
| **锁粒度** | 行级锁 + 间隙锁 | 表级锁 |
| **外键支持** | ✅ 支持 | ❌ 不支持 |
| **MVCC** | ✅ 支持（高并发） | ❌ 不支持 |
| **崩溃恢复** | 自动恢复 | 需手动修复 |
| **全文索引** | 5.6+ 支持 | ✅ 原生支持 |
| **COUNT(*)** | 全表扫描 | 保存了计数器，快 |
| **适用场景** | 高并发、事务需求、数据可靠性 | 读多写少、不需要事务、日志表 |

**选择建议：**
- **需要事务和行级锁 → InnoDB**
- **追求高写入性能且不需要事务 → MyISAM**（已逐渐被淘汰）
- **需要全文搜索且是 MySQL 5.5 及以下 → MyISAM**（新版本推荐 InnoDB + 全文索引）

---

## 题目四：数据库慢查询如何排查和优化？

### 参考答案

**排查步骤：**

1. **开启慢查询日志**
   ```sql
   -- 查看慢查询配置
   SHOW VARIABLES LIKE 'slow_query%';
   SHOW VARIABLES LIKE 'long_query_time';
   
   -- 开启慢查询日志（临时）
   SET GLOBAL slow_query_log = 'ON';
   SET GLOBAL long_query_time = 2; -- 超过2秒记录
   ```

2. **使用 EXPLAIN 分析**
   ```sql
   EXPLAIN SELECT ...;  -- 重点关注 type、key、rows、Extra
   ```

3. **使用 Show Profile 分析**
   ```sql
   SET profiling = 1;
   执行你的查询;
   SHOW PROFILES;
   SHOW PROFILE FOR QUERY 1;
   ```

4. **查看锁等待**
   ```sql
   SHOW ENGINE INNODB STATUS; -- 查看死锁和锁等待信息
   SELECT * FROM information_schema.INNODB_LOCK_WAITS; -- 锁等待
   ```

**优化手段：**

| 优化方向 | 具体措施 |
|----------|----------|
| **索引优化** | 确认查询走了索引，添加合适索引 |
| **查询优化** | 避免 SELECT *，减少 JOIN，分解大查询 |
| **分页优化** | 使用游标分页（WHERE id > last_id）替代 OFFSET |
| **表结构优化** | 适当冗余字段，减少 JOIN |
| **架构优化** | 读写分离、分库分表、引入缓存 |

**案例：分页优化**
```sql
-- 慢（OFFSET 大时性能骤降）
SELECT * FROM orders LIMIT 100000, 20;

-- 快（游标方式）
SELECT * FROM orders WHERE id > 100000 LIMIT 20;
```

---

## 题目五：PostgreSQL 中如何进行查询调优？有哪些常用技巧？

### 参考答案

**1. 使用 EXPLAIN ANALYZE 分析执行计划**
```sql
EXPLAIN (ANALYZE, BUFFERS, FORMAT TEXT) SELECT * FROM users WHERE ...;
```

**2. 创建合适的索引**
```sql
-- 单列索引
CREATE INDEX idx_users_name ON users(name);

-- 联合索引（注意顺序）
CREATE INDEX idx_orders_user_status ON orders(user_id, status);

-- 表达式索引（避免全表扫描）
CREATE INDEX idx_users_created ON users((DATE(created_at)));

-- 部分索引（只索引符合条件的行）
CREATE INDEX idx_orders_active ON orders(user_id) WHERE status = 'active';
```

**3. 统计分析信息**
```sql
-- 更新统计信息（数据变化大后执行）
ANALYZE users;

-- 查看表统计信息
SELECT schemaname, tablename, n_live_tup, n_dead_tup FROM pg_stat_user_tables;
```

**4. 常用调优参数**
```sql
-- 查看当前配置
SHOW all;

-- 关键参数调优（需reload）
-- shared_buffers：缓存大小，建议为系统内存的1/4
-- work_mem：排序/哈希操作内存
-- effective_cache_size：操作系统缓存预估
-- random_page_cost：随机页访问成本（SSD设为4，机械盘设为1.1）
```

**5. 识别问题查询**
```sql
-- 查看长时间运行的查询
SELECT pid, now() - query_start AS duration, query
FROM pg_stat_activity
WHERE state != 'idle' AND now() - query_start > interval '5 minutes';

-- 终止问题查询
SELECT pg_cancel_backend(pid);  -- 温柔地取消
SELECT pg_terminate_backend(pid); -- 强制终止
```

**6. PostgreSQL 特有优化**
- **BRIN 索引**：适合自然有序的大表（如按时间排序的日志表），比 B-tree 小很多
- **覆盖索引**：确保查询所需字段都在索引中，避免回表
- **JSONB 索引**：对 JSON 字段使用 GIN 索引加速查询

---

## 📚 参考来源

1. [MySQL索引失效的典型场景](https://dev.mysql.com/doc/refman/8.0/en/mysql-indexes.html)
2. [EXPLAIN Output Format - MySQL官方文档](https://dev.mysql.com/doc/refman/8.0/en/explain-output.html)
3. [InnoDB vs MyISAM - MySQL引擎对比](https://dev.mysql.com/doc/refman/8.0/en/innodb-introduction.html)
4. [PostgreSQL EXPLAIN分析](https://www.postgresql.org/docs/current/using-explain.html)
5. [PostgreSQL查询优化技巧](https://www.postgresql.org/docs/current/performance-tips.html)