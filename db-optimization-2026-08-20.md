# 数据库性能优化面试题

> 📅 日期：2026-08-20
> 🗄️ 涵盖：MySQL / PostgreSQL 索引优化、查询优化、存储引擎、缓存策略

---

## 题目一：如何判断是否需要给某列建索引？哪些情况建了索引反而会更慢？

### ✅ 答案

**需要建索引的典型场景：**
- WHERE、JOIN、ORDER BY、GROUP BY 频繁使用的列
- 列基数（Cardinality）高，即不同值数量多（如用户ID、手机号）
- 数据量超过几十万行，且查询频率高
- 主键和外键列

**不适合建索引或建了更慢的情况：**
1. **低基数字段**（如性别、状态标志），索引区分度低，全表扫描更快
2. **频繁更新的列**（UPDATE/DELETE），索引维护成本高
3. **TEXT/BLOB 类型的列**，索引体积大，IO 开销高
4. **数据量很小**时，数据库走全表扫描反而更快
5. **回表代价高**，如果索引字段不能覆盖查询所需全部列，需要回表两次查询

> 📎 来源：https://dev.mysql.com/doc/refman/8.0/en/mysql-indexes.html

---

## 题目二：Explain 执行计划中，哪些字段是重点关注对象？

### ✅ 答案

Explain 是分析 SQL 性能的核心工具，重点字段：

| 字段 | 含义 | 关注点 |
|------|------|--------|
| **type** | 访问类型 | 最好达到 `ref/range`，避免 `ALL`（全表扫描） |
| **key** | 实际使用的索引 | 应有值，不能是 NULL |
| **rows** | 扫描行数 | 越少越好 |
| **Extra** | 附加信息 | 出现 `Using filesort/Using temporary` 需要优化 |
| **possible_keys** | 可用索引列表 | 有值但没用上 = 索引失效 |
| **filtered** | 过滤比例 | 越大说明筛选效率越高 |

**经典反面案例：**
```
type: ALL          ← 全表扫描，大忌
Extra: Using filesort  ← 磁盘排序，数据量大时极慢
Extra: Using temporary ← 临时表，内存不够会溢到磁盘
```

> 📎 来源：https://dev.mysql.com/doc/refman/8.0/en/explain-output.html

---

## 题目三：MySQL InnoDB 和 MyISAM 存储引擎的核心区别是什么？如何选择？

### ✅ 答案

| 特性 | InnoDB | MyISAM |
|------|--------|--------|
| **事务支持** | ✅ 支持 ACID 事务 | ❌ 不支持 |
| **行级锁** | ✅ 支持，高并发好 | ❌ 表级锁 |
| **外键约束** | ✅ 支持 | ❌ 不支持 |
| **崩溃恢复** | ✅ 自动恢复（redo log） | ❌ 损坏可能丢数据 |
| **全文索引** | ✅ 支持（5.6+） | ✅ 支持（更早支持） |
| **COUNT(*)** | 慢（全表扫描） | 快（存储了行数） |
| **适用场景** | 核心业务、高并发、需事务 | 只读/历史数据、日志 |

**选择建议：**
- **InnoDB**：几乎所有场景首选，特别是写多的业务
- **MyISAM**：只读静态表、需要全文搜索、古老系统迁移过渡

> 📎 来源：https://dev.mysql.com/doc/refman/8.0/en/innodb-introduction.html

---

## 题目四：分页查询（LIMIT offset, n）为什么越往后越慢？如何优化？

### ✅ 答案

**根本原因：** MySQL 的 LIMIT 实现是**先读取再丢弃**前面的行。

```sql
-- 第10000页，每页20条
SELECT * FROM orders LIMIT 200000, 20;
-- 实际执行：读取200020行，丢弃前200000行
```

**几种优化方案：**

**方案1：延迟关联（推荐）**
```sql
-- 先用索引定位ID，再关联查完整数据
SELECT * FROM orders o
INNER JOIN (
    SELECT id FROM orders ORDER BY id LIMIT 200000, 20
) t ON o.id = t.id;
```

**方案2：游标分页（最优）**
```sql
-- 记住上一页最后一条ID
SELECT * FROM orders
WHERE id > last_page_max_id
ORDER BY id LIMIT 20;
```

**方案3：记录总数缓存**
不要每次 COUNT(*) 分页，直接用总页数缓存起来，翻页时不计算。

> 📎 来源：https://www.mysql.com/why-mysql/benchmarks/（MySQL官方性能测试文档）

---

## 题目五：数据库缓存策略有哪些？Redis 和 MySQL 如何配合使用？

### ✅ 答案

**常见缓存读写策略：**

### Cache-Aside（最常用）
```
读：先读缓存，缓存没有 → 读DB → 写回缓存
写：更新DB → 删除缓存（不是更新缓存）
```

### Read/Write Through
- 读写操作都经过缓存层，缓存负责和DB同步

### Write Behind
- 写操作先写缓存，异步批量写DB

**Redis + MySQL 配合最佳实践：**

```
1. 缓存热点数据（商品详情、用户信息）
2. 缓存过期策略：设置 TTL，防止数据陈旧
3. 缓存穿透：用空值标记或布隆过滤器拦截无效key
4. 缓存击穿：热点key过期时，用互斥锁或永不过期+异步更新
5. 缓存雪崩：随机TTL + Redis集群高可用
```

**典型代码逻辑（Cache-Aside）：**
```python
def get_user(user_id):
    cache_key = f"user:{user_id}"
    user = redis.get(cache_key)
    if user:
        return json.loads(user)
    user = db.query("SELECT * FROM users WHERE id = %s", user_id)
    redis.setex(cache_key, 3600, json.dumps(user))  # TTL=1小时
    return user
```

> 📎 来源：https://redis.io/docs/manual patterns/caching/

---

## 📚 更多学习资源

- MySQL 官方文档：https://dev.mysql.com/doc/refman/8.0/en/optimization.html
- PostgreSQL 性能调优：https://www.postgresql.org/docs/current/performance-tips.html
- 小林 Coding（图解MySQL）：https://xiaolincoding.com/

---

*本文件由 AI 自动生成并推送到 GitHub 仓库*
