# 数据库性能优化面试题 — 2026-08-03

---分割线---

【题目】MySQL 中 InnoDB 引擎的索引原理是什么？为什么主键索引查询比普通索引快？

【答案】

InnoDB 使用 B+ 树作为索引结构，有以下核心特点：

**B+ 树结构：**
- 非叶子节点只存储索引键值和指针，不存储数据
- 叶子节点包含所有数据，按顺序用双向链表连接
- 每个节点的大小默认为 16KB（一个页）

**主键索引（聚簇索引）：**
- 叶节点直接存储整行数据
- 数据物理存储顺序与主键顺序一致
- 一个表只能有一个聚簇索引

**普通索引（辅助索引/二级索引）：**
- 叶节点存储主键值和索引列值
- 查询时需要"回表"：先在辅助索引中找到主键，再通过主键去聚簇索引查找完整行数据

**主键索引比普通索引快的原因：**
1. 减少磁盘 I/O：主键索引一次定位即可获取全部数据
2. 避免回表：普通索引查到主键后还需再查主键索引（索引覆盖不住时）
3. 缓存命中率高：主键查询的数据更紧凑，缓存效率更高

**覆盖索引优化：**
如果查询的所有列都包含在索引中（如 `SELECT id, name FROM users WHERE name='Tom'`），则不需要回表，直接在辅助索引中返回结果，这种现象称为"索引覆盖"。

📖 来源：https://github.com/CyC2018/CS-Notes

---分割线---

【题目】如何判断一条 SQL 语句是否走索引？为什么有时候明明有索引但 MySQL 却不选择走索引？

【答案】

**判断是否走索引的方法：**
```sql
EXPLAIN SELECT * FROM users WHERE name = 'Tom';
-- key 列显示实际使用的索引
-- type 列显示访问类型（const > eq_ref > ref > range > ALL）
-- Extra 列出现 "Using index condition" 表示使用索引过滤
```

**不走索引的常见原因：**

1. **索引列参与运算**
```sql
-- 不会走索引
SELECT * FROM orders WHERE YEAR(create_time) = 2026;
-- 应该改为
SELECT * FROM orders WHERE create_time >= '2026-01-01' AND create_time < '2027-01-01';
```

2. **索引列使用函数**
```sql
-- 不会走索引
SELECT * FROM users WHERE LEFT(name, 3) = 'Tom';
```

3. **数据类型隐式转换**
```sql
-- phone 是 varchar 类型，传入数字会隐式转换
SELECT * FROM users WHERE phone = 13800138000;  -- 不走索引
SELECT * FROM users WHERE phone = '13800138000'; -- 走索引
```

4. **使用前缀 LIKE 或以通配符开头**
```sql
-- 以 % 开头不走索引
SELECT * FROM users WHERE name LIKE '%om';
-- 以固定值开头会走索引
SELECT * FROM users WHERE name LIKE 'Tom%';
```

5. **数据分布不均匀（MySQL 优化器判断）**
- 如果索引选择性很低（大量重复值），优化器可能选择全表扫描
- 可以用 `SHOW INDEX FROM table` 查看基数（Cardinality）

6. **查询数据量占比过大**
- 超过一定比例（通常 20%~30%）优化器认为全表扫描更快
- 可以用 `FORCE INDEX(idx_name)` 强制指定索引

7. **多表 JOIN 时顺序不当**
- 驱动表选择错误导致无法有效利用索引

📖 来源：https://github.com/CyC2018/CS-Notes

---分割线---

【题目】MySQL 查询语句执行慢的原因有哪些？如何进行 SQL 性能分析与优化？

【答案】

**查询慢的常见原因：**

1. **缺乏索引或索引失效**
2. **单次查询数据量过大**（如 SELECT *）
3. **关联查询太多表**（超过 5-7 张表）
4. **锁竞争严重**（长事务阻塞）
5. **服务器配置不合理**（内存、缓冲区不足）
6. **数据量过大**（未分库分表）
7. **统计信息不准确**（导致优化器选错执行计划）

**性能分析工具：**

```sql
-- 1. EXPLAIN 分析执行计划
EXPLAIN ANALYZE SELECT * FROM orders WHERE user_id = 100;
-- 分析内容：扫描行数、是否使用索引、预估 vs 实际成本

-- 2. 慢查询日志
SHOW VARIABLES LIKE 'slow_query_log';
SET GLOBAL slow_query_log = 'ON';
SET GLOBAL long_query_time = 1;  -- 超过 1 秒记录
-- 查看慢查询文件
SHOW VARIABLES LIKE 'slow_query_log_file';

-- 3. PROFILING
SET profiling = 1;
SELECT * FROM orders;
SHOW PROFILES;  -- 查看每条语句耗时
SHOW PROFILE FOR QUERY 1;  -- 详细各阶段耗时

-- 4. 索引状态检查
SHOW INDEX FROM orders;
ANALYZE TABLE orders;  -- 更新统计信息
```

**核心优化手段：**

| 优化方向 | 具体措施 |
|---------|---------|
| 索引优化 | 添加合适索引、避免索引失效、覆盖索引 |
| SQL 重写 | 避免 SELECT *、减少 JOIN、分解大查询 |
| 业务层 | 分页优化（游标分页）、异步处理 |
| 架构层 | 读写分离、分库分表、引入缓存 |
| 配置调优 | 调整 innodb_buffer_pool_size、max_connections |

**经典优化案例：**
```sql
-- 原始慢查询（分页深度访问）
SELECT * FROM orders ORDER BY id LIMIT 1000000, 10;
-- 优化：使用游标分页（延迟关联）
SELECT t.* FROM orders t 
INNER JOIN (SELECT id FROM orders ORDER BY id LIMIT 1000000, 10) AS tmp 
ON t.id = tmp.id;
```

📖 来源：https://github.com/qq286158530/interview-questions

---分割线---

【题目】MySQL 的存储引擎 InnoDB 和 MyISAM 有什么区别？如何根据业务场景选择？

【答案】

**核心区别对比：**

| 特性 | InnoDB | MyISAM |
|-----|--------|--------|
| 事务支持 | 支持 ACID 事务 | 不支持事务 |
| 行级锁 | 支持行级锁 | 只支持表级锁 |
| 外键约束 | 支持外键 | 不支持外键 |
| 崩溃恢复 | 自动崩溃恢复（redo log） | 需手动修复（myisamchk） |
| 索引结构 | B+ 树（聚簇索引） | B+ 树（非聚簇索引） |
| 全文索引 | 5.6+ 支持 | 原生支持 |
| 存储空间 | 约 2 倍 MyISAM | 较小（压缩格式） |
| 插入性能 | 批量插入略低 | 批量插入性能好 |
| 适用场景 | 事务优先的业务 | 读多写少、不需事务 |

**InnoDB 核心优势：**
- 支持行级锁，并发写入性能好
- 支持 MVCC（多版本并发控制），读写不互斥
- 崩溃恢复能力强，不易丢数据
- 支持外键，保证参照完整性

**MyISAM 适用场景：**
- 日志系统、报表系统等只读或很少更新的场景
- 全文搜索需求（早期版本）
- 追求插入速度（如数据仓库ETL中间表）

**选择建议：**

```
业务场景判断流程：
│─ 是否需要事务？ ──Yes──→ 选择 InnoDB
│
│─ No
│─ 是否需要外键？ ──Yes──→ 选择 InnoDB
│
│─ No
│─ 是否高并发写入？ ──Yes──→ 选择 InnoDB
│
│─ No
│─ 是否以读为主且无需事务？ ──Yes──→ 可选 MyISAM（但建议仍用 InnoDB）
```

**特别注意：** MySQL 8.0+ 已将默认存储引擎从 MyISAM 改为 InnoDB，新项目一律使用 InnoDB。

📖 来源：https://github.com/CyC2018/CS-Notes

---分割线---

【题目】Redis 缓存和 MySQL 数据库如何配合使用？缓存穿透、缓存击穿、缓存雪崩是什么？如何解决？

【答案】

**缓存与数据库配合模式：**

```
读操作：
Client → Redis（命中直接返回）→ 未命中 → MySQL → 回填Redis → 返回

写操作：
Client → MySQL（先写库）→ 删除/更新Redis（Cache Aside 模式）
         注意：不能先删缓存再写库，会造成数据不一致
```

**三大经典问题：**

**1. 缓存穿透（查询不存在的数据）**
- 现象：大量请求查询数据库和缓存都不存在的数据
- 危害：数据库压力剧增，可能被恶意利用

解决方案：
- 布隆过滤器（Bloom Filter）：在缓存层前加过滤层
- 空值缓存：对查询结果为空的数据也缓存（设置短过期时间）
```java
String key = "user:1000";
String value = redis.get(key);
if (value == null) {
    User user = mysql.get(id);
    if (user == null) {
        redis.setex(key, 300, "");  // 缓存空值
    } else {
        redis.set(key, JSON.toJSONString(user));
    }
}
```

**2. 缓存击穿（热点 key 过期）**
- 现象：某个热点 key 过期瞬间，大量并发请求同时穿透到数据库
- 危害：数据库瞬时压力过大

解决方案：
- 互斥锁：只允许一个线程重建缓存
```java
String value = redis.get(key);
if (value == null) {
    String lockKey = "lock:" + key;
    if (redis.setnx(lockKey, "1", 10)) {  // 10秒过期
        User user = mysql.get(id);
        redis.set(key, JSON.toJSONString(user));
        redis.del(lockKey);
    } else {
        Thread.sleep(50);
        return redis.get(key);  // 等待后重试
    }
}
```
- 逻辑过期：热点数据永不过期，用逻辑字段判断是否过期，后台异步更新

**3. 缓存雪崩（大量 key 同时过期）**
- 现象：大量缓存 key 在同一时间集中过期
- 危害：短时间内大量请求打到数据库

解决方案：
- 过期时间随机化：`TTL = baseTTL + random(0, 5分钟)`
- 热点数据永不过期（配合后台更新）
- 多级缓存：本地缓存 + Redis + MySQL
- 熔断/限流：数据库压力过大时拒绝部分请求

**最佳实践总结：**

| 问题 | 解决思路 | 方案 |
|-----|---------|------|
| 穿透 | 拦截不存在请求 | 布隆过滤器 / 空值缓存 |
| 击穿 | 控制并发重建 | 互斥锁 / 逻辑过期 |
| 雪崩 | 分散过期时间 | 随机 TTL / 多级缓存 |

📖 来源：https://github.com/xingshaocheng/architect-awesome
