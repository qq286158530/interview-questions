# 数据库性能优化面试题

> 日期：2026-08-17
> 来源：整理自经典面试题库

---

## 题目1：MySQL索引失效的场景有哪些？如何避免？

### 参考答案

**索引失效的常见场景：**

1. **使用函数或运算**
   - `WHERE YEAR(create_time) = 2024` — 对索引列使用函数
   - `WHERE id + 1 = 10` — 对索引列进行运算

2. **类型转换**
   - `WHERE phone = 13800138000`（phone是varchar类型，但传入数字）
   - 隐式类型转换导致索引失效

3. **模糊查询以%开头**
   - `WHERE name LIKE '%张三'` — 前导通配符无法使用索引
   - `WHERE name LIKE '张%'` — 可以使用索引

4. **OR连接条件**
   - `WHERE id = 1 OR name = '张三'` — 涉及非索引列时全表扫描
   - 解决方案：为OR两边的列都建立索引

5. **不等于比较**
   - `WHERE status != 1` — 通常无法使用索引
   - `WHERE status NOT IN (1,2)` — 同理

6. **NULL值判断**
   - `WHERE age IS NULL` — 部分引擎可使用索引
   - `WHERE age IS NOT NULL` — 通常无法使用索引

7. **联合索引不遵循最左前缀原则**
   - 创建了`(a,b,c)`联合索引，但查询只有`WHERE b = 1`

**如何避免：**
- 避免在索引列上使用函数、运算
- 确保类型一致
- 使用覆盖索引
- 合理设计索引，考虑查询模式

---

## 题目2：如何排查和优化慢查询？

### 参考答案

**排查步骤：**

1. **开启慢查询日志**
   ```sql
   -- 查看慢查询配置
   SHOW VARIABLES LIKE 'slow_query%';
   SHOW VARIABLES LIKE 'long_query_time';
   
   -- 开启慢查询日志
   SET GLOBAL slow_query_log = 'ON';
   SET GLOBAL long_query_time = 1; -- 超过1秒记录
   ```

2. **使用EXPLAIN分析**
   ```sql
   EXPLAIN SELECT * FROM orders WHERE status = 1;
   ```
   关键字段：
   - `type`: 性能从优到差 const > eq_ref > ref > range > index > ALL
   - `key`: 实际使用的索引
   - `rows`: 扫描行数，越少越好
   - `Extra`: Using filesort、Using temporary需优化

3. **使用SHOW PROFILE**
   ```sql
   SET profiling = 1;
   -- 执行查询
   SHOW PROFILES;
   SHOW PROFILE FOR QUERY 1;
   ```

**优化方法：**

1. **索引优化**
   - 添加合适索引
   - 使用覆盖索引避免回表

2. **SQL语句优化**
   - 避免SELECT *
   - 分解大查询
   - 批量操作代替循环

3. **表结构优化**
   - 字段类型要合适
   - 适当冗余减少JOIN
   - 分库分表

4. **配置优化**
   - 调整buffer pool大小
   - 优化连接池参数

---

## 题目3：InnoDB与MyISAM存储引擎的区别？如何选择？

### 参考答案

| 特性 | InnoDB | MyISAM |
|------|--------|--------|
| 事务支持 | 支持ACID事务 | 不支持 |
| 锁粒度 | 行级锁 | 表级锁 |
| 外键 | 支持 | 不支持 |
| 全文索引 | 5.6+支持 | 支持 |
| 索引结构 | B+Tree | B+Tree |
| 崩溃恢复 | 自动恢复 | 需手动修复 |
| 存储空间 | 约2倍 | 较小 |
| 适用场景 | 写密集、事务需求 | 读密集、表统计 |

**选择建议：**

- **选择InnoDB：**
  - 需要事务支持（转账、订单等）
  - 并发写入量大
  - 需要行级锁
  - 数据可靠性要求高
  - MySQL 5.7+默认引擎

- **选择MyISAM：**
  - 只读的静态数据
  - 全文搜索需求
  - 表级锁可接受
  - 存储空间紧张

**实际选择：** 现代MySQL应用中，InnoDB几乎是唯一选择，除非有特殊原因。

---

## 题目4：数据库缓存策略有哪些？如何设计多级缓存？

### 参考答案

**缓存策略分类：**

1. **Cache Aside（旁路缓存）**
   ```
   读：先读缓存，缓存未命中读数据库，写入缓存
   写：先写数据库，删除缓存（而非更新）
   ```
   - 最常用策略
   - 缺点：可能缓存不一致

2. **Read Through**
   - 应用程序只和缓存交互
   - 缓存负责从数据库加载数据

3. **Write Through**
   - 写入时同步更新缓存和数据库
   - 保证一致性，但写入较慢

4. **Write Behind**
   - 异步写入，先更新缓存，定期写入数据库
   - 性能最高，但可能丢数据

**多级缓存设计：**

```
客户端 -> CDN -> Nginx本地缓存 -> Redis -> MySQL Query Cache
```

1. **本地缓存（进程内）**
   - Caffeine/Guava Cache
   - 热点数据，毫秒级访问
   - 缺点：各实例独立，无法同步

2. **分布式缓存（Redis/Memcached）**
   - 跨进程共享
   - 建议数据量：GB~TB级
   - 注意：网络延迟

3. **数据库Query Cache**（MySQL 8.0已移除）
   - 已不推荐使用

**缓存问题解决方案：**

| 问题 | 解决方案 |
|------|----------|
| 缓存穿透 | 布隆过滤器/空值缓存 |
| 缓存击穿 | 互斥锁/热点数据永不过期 |
| 缓存雪崩 | 过期时间随机化/多级缓存 |

---

## 题目5：PostgreSQL与MySQL在性能优化上的主要区别是什么？

### 参考答案

**架构差异：**

1. **MVCC实现**
   - MySQL InnoDB：回滚段（Undo Log）
   - PostgreSQL：多版本快照，无须回滚段

2. **并发控制**
   - MySQL：表级锁+行级锁（InnoDB）
   - PostgreSQL：MVCC+SSI隔离级别，支持真正并发

**性能优化差异：**

| 方面 | MySQL | PostgreSQL |
|------|-------|------------|
| 索引类型 | B-Tree、哈希、全文 | B-Tree、哈希、全文、GiST、SP-GiST、GIN、BRIN |
| 分区表 | 有限支持 | 声明式分区更强大 |
| 并行查询 | 有限 | 支持并行_seqscan、_hashjoin等 |
| 物化视图 | 不支持 | 支持，大数据量预计算 |
| JSON支持 | JSON函数 | 原生JSONB，性能更好 |

**PostgreSQL独特优势：**

1. **索引丰富**
   ```sql
   -- 表达式索引
   CREATE INDEX idx ON orders (LOWER(customer_name));
   -- 部分索引
   CREATE INDEX idx ON orders (id) WHERE status = 1;
   ```

2. **并行查询**
   ```sql
   EXPLAIN SELECT * FROM large_table ORDER BY id;
   -- 显示 Parallel Seq Scan
   ```

3. **物化视图**
   ```sql
   CREATE MATERIALIZED VIEW sales_summary AS
   SELECT date, SUM(amount) FROM orders GROUP BY date;
   REFRESH MATERIALIZED VIEW sales_summary;
   ```

4. **外部表包装器**
   - 可直接查询其他数据库或CSV文件

**选择建议：**
- 传统Web应用、结构简单：MySQL
- 复杂查询、数据量大、需强一致性：PostgreSQL
- GIS应用：PostgreSQL + PostGIS
- 需要MySQL兼容：MySQL

---

> 📚 更多面试题请访问：[GitHub - interview-questions](https://github.com/qq286158530/interview-questions)
