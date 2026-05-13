# 数据库性能优化面试题

> 发布日期：2026-05-13
> 标签：MySQL | PostgreSQL | 性能优化 | 索引 | 查询优化

---

## 题目一：索引失效的场景有哪些？如何避免？

### 参考答案

**常见索引失效场景：**

1. **使用函数或运算**
   ```sql
   -- 索引失效
   SELECT * FROM users WHERE YEAR(created_at) = 2025;
   -- 正确写法
   SELECT * FROM users WHERE created_at >= '2025-01-01' AND created_at < '2026-01-01';
   ```

2. **类型转换**
   ```sql
   -- phone 是 varchar，传入数字会导致全表扫描
   SELECT * FROM users WHERE phone = 13800138000;
   -- 正确写法
   SELECT * FROM users WHERE phone = '13800138000';
   ```

3. **LIKE 以前导通配符开头**
   ```sql
   -- 索引失效
   SELECT * FROM users WHERE name LIKE '%张%';
   -- 可以使用索引
   SELECT * FROM users WHERE name LIKE '张%';
   ```

4. **OR 连接不同类型列**
   ```sql
   -- 索引失效（user_id 是 int，name 是 varchar）
   SELECT * FROM users WHERE user_id = 1 OR name = '张三';
   ```

5. **不等于比较**
   ```sql
   -- 索引失效
   SELECT * FROM users WHERE status != 1;
   ```

6. **NOT IN / NOT EXISTS**
   ```sql
   -- 索引失效
   SELECT * FROM users WHERE id NOT IN (1, 2, 3);
   ```

7. **联合索引违反最左前缀原则**
   ```sql
   -- 假设 index(a, b, c)
   -- 失效场景
   SELECT * FROM t WHERE b = 1;
   SELECT * FROM t WHERE c = 1;
   ```

**如何避免：**
- 避免在 WHERE 条件的列上使用函数或运算
- 类型必须严格匹配，避免隐式类型转换
- 尽量使用前导通配符或改用全文索引
- 将 OR 条件拆分为 UNION
- 使用覆盖索引减少回表

---

## 题目二：Explain 执行计划中各个字段的含义是什么？如何根据它优化SQL？

### 参考答案

```sql
EXPLAIN SELECT * FROM users WHERE name = '张三';
```

**关键字段解释：**

| 字段 | 含义 |
|------|------|
| `type` | 关联类型，system > const > eq_ref > ref > range > index > ALL |
| `key` | 实际使用的索引 |
| `rows` | 预估扫描行数，越少越好 |
| `Extra` | 额外信息，Using filesort / Using temporary / Using index 表示有问题 |

**常见优化方向：**

- `type = ALL`（全表扫描）：必须加索引
- `key = NULL` 且 `rows` 过大：检查是否遗漏索引
- `Extra = Using filesort`：filesort 意味着在内存或磁盘排序，加索引消除
- `Extra = Using temporary`：临时表创建，尝试优化查询结构

```sql
-- 案例优化
-- Before: 全表 + filesort
EXPLAIN SELECT * FROM orders WHERE status = 1 ORDER BY created_at DESC;

-- After: 添加索引
ALTER TABLE orders ADD INDEX idx_status_created(status, created_at);
```

---

## 题目三：MySQL InnoDB 和 MyISAM 存储引擎的区别？如何选择？

### 参考答案

| 特性 | InnoDB | MyISAM |
|------|--------|--------|
| 事务支持 | ✅ 支持 | ❌ 不支持 |
| 行锁 | ✅ 支持 | ❌ 表锁 |
| 外键 | ✅ 支持 | ❌ 不支持 |
| 全文索引 | ✅ 5.6+支持 | ✅ 原生支持 |
| 表空间 | 较大 | 较小 |
| 崩溃恢复 | 自动恢复 | 较差 |
| MVCC | ✅ 支持 | ❌ 不支持 |
| 适用场景 | 写多、事务需求 | 读多、表统计 |

**如何选择：**
- 需要事务、行锁、外键 → InnoDB（几乎所有场景的默认选择）
- 只读/统计为主的表，且不需要事务 → MyISAM 可考虑（但越来越少用）
- 现代生产环境几乎全部使用 InnoDB

---

## 题目四：什么是慢查询？如何定位并优化慢查询？

### 参考答案

**慢查询（Slow Query）**：执行时间超过 `long_query_time` 阈值的 SQL。

**定位步骤：**

1. **开启慢查询日志**
   ```sql
   SET GLOBAL slow_query_log = 'ON';
   SET GLOBAL long_query_time = 1; -- 超过1秒记录
   SET GLOBAL slow_query_log_file = '/var/log/mysql/slow.log';
   ```

2. **使用 pt-query-digest 分析**
   ```bash
   pt-query-digest /var/log/mysql/slow.log
   ```

3. **根据分析结果定位 TOP SQL**

**优化手段：**

1. **加索引**：分析 WHERE / JOIN / ORDER BY 字段
2. **改写SQL**：避免 SELECT *，减少子查询，用 JOIN 替代
3. **分页优化**：延迟关联 + 覆盖索引
   ```sql
   -- 低效
   SELECT * FROM orders ORDER BY id LIMIT 1000000, 10;
   -- 高效
   SELECT * FROM orders o
   INNER JOIN (SELECT id FROM orders ORDER BY id LIMIT 1000000, 10) t
   ON o.id = t.id;
   ```
4. **读写分离**：将读流量打到从库
5. **业务层优化**：加缓存、减少查询频率

---

## 题目五：MySQL 主从复制原理是什么？延迟的原因有哪些？如何优化？

### 参考答案

**主从复制原理（Binlog 复制）：**
1. 主库将写操作记录到 Binlog
2. 从库 I/O 线程请求主库 Binlog，写入本地 Relay Log
3. 从库 SQL 线程重放 Relay Log，完成数据同步

**常见延迟原因：**
- 从库配置低，SQL 线程重放速度跟不上主库写入速度
- 大事务执行（主库几秒，从库需要更长时间重放）
- 从库同时承担大量查询，CPU/IO 争抢
- 网络延迟
- 缺少合理索引，导致从库重放时全表扫描

**优化方案：**

```sql
-- 1. 使用并行复制（MySQL 5.7+）
STOP SLAVE;
SET GLOBAL slave_parallel_workers = 8; -- 根据CPU核心数设置
START SLAVE;

-- 2. 配置多源复制，分散写入压力

-- 3. 业务层优化：读写分离 + 延迟感知的路由
-- 对实时性要求高的读走主库

-- 4. 监控延迟
SHOW SLAVE STATUS\G;
-- 关注 Seconds_Behind_Master
```

---

## 参考来源

- [MySQL 索引失效场景 - 掘金](https://juejin.cn)
- [Explain 执行计划详解 - 美团技术博客](https://tech.meituan.com)
- [MySQL InnoDB vs MyISAM 对比 - 廖雪峰的博客](https://www.liaoxuefeng.com)
- [慢查询优化指南 - 阿里巴巴数据库内核月报](http://mysql.taobao.org)
- [MySQL 主从复制原理与优化 - 简书](https://www.jianshu.com)

---

_*本面试题库由 GitHub 机器人自动维护*_
_📚 GitHub: https://github.com/qq286158530/interview-questions_