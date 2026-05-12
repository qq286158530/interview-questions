# 数据库性能优化面试题

> 日期：2026-05-12
> 来源：整理自互联网高质量技术文章

---

## 题目一：MySQL 索引失效的场景有哪些？如何避免？

### 参考答案

**索引失效的常见场景：**

1. **使用函数或表达式**
   ```sql
   -- 失效：WHERE YEAR(create_time) = 2026
   -- 有效：WHERE create_time >= '2026-01-01' AND create_time < '2027-01-01'
   ```

2. **左模糊匹配 LIKE '%xxx'**
   ```sql
   -- 失效：WHERE name LIKE '%张'
   -- 有效：WHERE name LIKE '张%'
   ```

3. **类型转换**
   ```sql
   -- 失效：WHERE phone = 13800138000 (phone 是 VARCHAR 类型)
   -- 有效：WHERE phone = '13800138000'
   ```

4. **使用 OR 连接条件**
   ```sql
   -- 失效：WHERE name = '张三' OR age = 20
   -- 改善：为 age 也建立索引，或拆分为 UNION
   ```

5. **联合索引未遵循最左前缀原则**
   ```sql
   -- 建立 INDEX idx(a,b,c)
   -- 失效：WHERE b = 1
   -- 有效：WHERE a = 1 或 WHERE a = 1 AND b = 2
   ```

6. **WHERE 中使用 !=、<>、NOT IN**
   ```sql
   -- 失效：WHERE status != 'active'
   ```

7. **范围查询放在后面**
   ```sql
   -- 失效：WHERE age > 20 AND name = '张三'
   -- 有效：WHERE name = '张三' AND age > 20（范围列放到最后）
   ```

**避免策略：**
- 分析执行计划（EXPLAIN），关注 `type`、`key`、`Extra`
- 避免在索引列上使用函数，优先将计算移到比较值侧
- 设计多列索引时，考虑列的区分度和查询频率
- 对于 OR 条件，尽量改写为 UNION ALL 或 JOIN

---

## 题目二：Explain 执行计划有哪些关键字段？如何解读？

### 参考答案

```sql
EXPLAIN SELECT * FROM orders WHERE user_id = 100 AND status = 'paid';
```

**关键字段解读：**

| 字段 | 含义 | 判断标准 |
|------|------|----------|
| **type** | 连接类型 | `const/eq_ref/ref/range/index/all`，最好达到 `ref` 级别，忌 `all` |
| **key** | 实际使用的索引 | 应与设计预期一致 |
| **rows** | 预估扫描行数 | 越小越好 |
| **Extra** | 额外信息 | 出现 `Using filesort/Using temporary` 表示效率低 |

**type 从好到差排序：**
```
const > eq_ref > ref > range > index > ALL
```

**Extra 常见值及处理：**

| Extra | 含义 | 优化方向 |
|-------|------|----------|
| `Using index` | 覆盖索引，无需回表 | 很好，继续保持 |
| `Using index condition` | 索引下推 | 正常 |
| `Using where` | 需要在存储引擎后过滤 | 检查是否索引覆盖 |
| `Using filesort` | 需额外排序 | 添加合适索引消除 |
| `Using temporary` | 需建临时表 | 优化 SQL 或增加索引 |
| `Using join buffer` | 嵌套循环读取大量数据 | 增加索引或改写 JOIN |

**实战技巧：**
- `rows` 特别大（如 > 10000）需考虑优化
- `key_len` 可判断索引被使用的程度
- `filtered` 表示返回行占总扫描行的百分比，越大越好

---

## 题目三：MySQL InnoDB 与 MyISAM 的区别？如何选择？

### 参考答案

| 特性 | InnoDB | MyISAM |
|------|--------|--------|
| **事务支持** | 支持 ACID | 不支持 |
| **行锁** | 支持行锁，并发好 | 只支持表锁 |
| **外键** | 支持 | 不支持 |
| **崩溃恢复** | 自动恢复 | 需手动 repair |
| **全文索引** | 5.6+ 支持 | 支持 |
| **COUNT(*)** | 全表扫描 | 保存数值，极快 |
| **存储结构** | 聚簇索引（数据文件和索引在一起） | 非聚簇索引（索引文件与数据文件分离） |
| **适用场景** | 高并发、事务需求、数据可靠性 | 读多写少、不需要事务 |

**InnoDB 为什么是默认引擎？**
1. 支持行锁，高并发下性能优秀
2. 支持事务，满足业务数据一致性需求
3. 支持崩溃自动恢复，数据安全有保障
4. 支持外键约束，保证引用完整性

**何时可选 MyISAM：**
- 日志系统、报表系统等只读/少写场景
- 全文检索需求且版本 < 5.6
- 对 COUNT(*) 查询性能要求极高

**实际建议：** 新项目默认 InnoDB，有特殊需求再评估切换。

---

## 题目四：如何优化大表分页查询（LIMIT offset, size）？

### 参考答案

**问题本质：**
```sql
SELECT * FROM orders LIMIT 1000000, 10
```
offset 越大，MySQL 需要扫描前 1000010 行，丢弃前 1000000 行，效率极低。

**优化方案：**

**方案一：主键 + 延迟关联**
```sql
-- 原始（慢）
SELECT * FROM orders ORDER BY id LIMIT 1000000, 10;

-- 优化（快）：先查主键，再关联
SELECT o.* FROM orders o
INNER JOIN (
    SELECT id FROM orders
    WHERE id > 1000000
    ORDER BY id
    LIMIT 10
) t ON o.id = t.id;
```

**方案二：游标分页（推荐）**
```sql
-- 第一页
SELECT * FROM orders ORDER BY id LIMIT 10;
-- 假设最后一行的 id = 12345

-- 第二页：传入上次的最大 id
SELECT * FROM orders
WHERE id > 12345
ORDER BY id LIMIT 10;
```

**方案三：覆盖索引 + 延迟关联**
```sql
SELECT * FROM orders
INNER JOIN (
    SELECT id FROM orders
    ORDER BY id
    LIMIT 1000000, 10
) t USING(id);
```

**根本原则：**
- 不要跳页读取，优先使用连续游标
- 利用主键或索引的有序性
- 深分页场景必须改写为基于主键范围的查询

---

## 题目五：PostgreSQL 与 MySQL 在性能优化上的主要差异？

### 参考答案

**1. 查询优化器**

| 方面 | MySQL | PostgreSQL |
|------|-------|------------|
| 优化器 | 基于成本（CBO），较简单 | 基于成本 + 规则，强大 |
| 统计信息 | 简化的直方图 | 丰富的统计信息（MCV、相关性） |
| 优化提示 | 支持 Hint 注释 | 一般不推荐 Hint，靠统计信息 |

**2. 索引类型**

| 索引类型 | MySQL | PostgreSQL |
|----------|-------|------------|
| B-Tree | ✅ 默认 | ✅ 默认 |
| Hash | ✅ | ✅ |
| GiST | ❌ | ✅（全文检索、几何类型） |
| GIN | ❌ | ✅（数组、全文搜索） |
| BRIN | ❌ | ✅（超大型表，块级索引） |
| 表达式索引 | 部分支持 | ✅ 完全支持 |
| 部分索引 | ❌ | ✅ |

**3. 连接（JOIN）策略**

PostgreSQL 支持更多 JOIN 类型：
```sql
-- PostgreSQL 特有：LATERAL JOIN，适合子查询依赖外部行
SELECT u.name, o.total
FROM users u
LEFT JOIN LATERAL (
    SELECT sum(amount) as total FROM orders
    WHERE user_id = u.id
) o ON true;
```

**4. MVCC 与并发控制**

| 方面 | MySQL (InnoDB) | PostgreSQL |
|------|----------------|------------|
| 实现 | 回滚段（Undo） | 版本链 |
| 脏读 | 不可能 | 不可能 |
| 读已提交 | ✅ 默认 | ✅ 默认，可配置 |
| 可重复读 | ✅ 默认 | ❌（MySQL 默认） |
| 序列化 | ✅ | ✅ |

**5. 实际优化建议对比**

```sql
-- PostgreSQL：善于利用表达式索引优化函数查询
CREATE INDEX idx ON orders (EXTRACT(YEAR FROM create_time));
SELECT * FROM orders WHERE EXTRACT(YEAR FROM create_time) = 2026;

-- PostgreSQL：善于利用部分索引
CREATE INDEX idx ON logs WHERE status = 'error';
SELECT * FROM logs WHERE status = 'error'; -- 命中部分索引

-- PostgreSQL：BRIN 索引适合超大型日志表
CREATE INDEX idx ON logs USING BRIN(create_time);
-- 存储空间极小，适合按时间顺序的表
```

**总结：** PostgreSQL 在高级特性（表达式索引、部分索引、LATERAL、统计信息）上更灵活，适合复杂查询场景；MySQL 在简单 OLTP 场景下更省心。

---

> 📚 参考来源：
> - MySQL 索引失效场景：https://zhuanlan.zhihu.com/p/590593200
> - Explain 执行计划详解：https://xiaolincoding.com/mysql/base/cluster.html
> - InnoDB vs MyISAM 对比：https://www.nowcoder.com/feed/main/detail/8e7d2f96c3d74c5da6d66a60e7b6d20e
> - 大表分页优化：https://www.cnblogs.com/wupeiqi/p/11977110.html
> - PostgreSQL vs MySQL：https://habr.com/en/articles/