# 数据库性能优化面试题（2026-05-07）

> 本文档整理了 5 道高质量的数据库性能优化面试题，涵盖 MySQL 索引优化、查询优化、存储引擎、缓存策略等核心知识点。

---

## 题目一：为什么 MySQL InnoDB 选择 B+Tree 作为索引的数据结构？

### 参考答案

MySQL InnoDB 选择 B+Tree 作为索引数据结构，主要有以下几个原因：

**1. B+Tree vs B Tree**
- B+Tree 只在叶子节点存储数据，而 B 树的非叶子节点也要存储数据，所以 B+Tree 的单个节点的数据量更小，在相同的磁盘 I/O 次数下，能查询更多的节点。
- B+Tree 叶子节点采用双链表连接，适合 MySQL 中常见的基于范围的顺序查找，而 B 树无法做到这一点。

**2. B+Tree vs 二叉树**
- 对于有 N 个叶子节点的 B+Tree，其搜索复杂度为 O(logdN)，其中 d 表示节点允许的最大子节点个数（实际应用中 d 值大于 100）。
- 即使数据达到千万级别，B+Tree 的高度依然维持在 3~4 层左右，一次查询只需要 3~4 次磁盘 I/O。
- 二叉树的搜索复杂度为 O(logN)，每个父节点只有 2 个子节点，检索到目标数据经历的磁盘 I/O 次数更多。

**3. B+Tree vs Hash**
- Hash 在做等值查询时效率很高，搜索复杂度为 O(1)。
- 但 Hash 表不适合做范围查询，只适合等值查询，而 B+Tree 索引应用场景更广泛。

**4. B+Tree 的实际优势**
- 叶子节点形成双向链表，便于范围查询
- 非叶子节点只存索引，扇出率高，树高低
- 适合磁盘预读和缓存

> 📚 来源：[小林 coding - MySQL 索引常见面试题](https://xiaolincoding.com/mysql/index/index_interview.html)

---

## 题目二：什么是聚簇索引和二级索引？两者有什么区别？

### 参考答案

**聚簇索引（Clustered Index）**
- 聚簇索引的叶子节点存储的是完整的行数据，即实际数据。
- InnoDB 中，主键索引就是聚簇索引。
- 每张表只有一个聚簇索引（因为数据只能按一种方式排序存储）。

**二级索引（Secondary Index / 辅助索引）**
- 二级索引的叶子节点存储的是主键值，而不是实际数据。
- 创建的普通索引、唯一索引、前缀索引都属于二级索引。

**两者的区别**

| 区别 | 聚簇索引 | 二级索引 |
|------|---------|---------|
| 叶子节点存储内容 | 完整行数据 | 主键值 |
| 查询效率 | 直接返回数据 | 可能需要回表 |
| 数量 | 每张表只有一个 | 可以有多个 |
| 建立方式 | 主键自动创建 | 手动创建 |

**回表与覆盖索引**
- 使用二级索引查询时，如果查询的数据不在二级索引中，需要先查到主键值，再回表查主键索引获取完整数据，这个过程叫**回表**。
- 如果要查询的字段恰好都在二级索引的叶子节点上，则不需要回表，直接返回数据，这叫**覆盖索引**。

> 📚 来源：[小林 coding - MySQL 索引常见面试题](https://xiaolincoding.com/mysql/index/index_interview.html)

---

## 题目三：联合索引的最左匹配原则是什么？哪些情况会导致索引失效？

### 参考答案

**最左匹配原则**
联合索引（a, b, c）的 B+Tree 先按 a 排序，a 相同时按 b 排序，b 相同时按 c 排序。因此，索引的生效必须从最左边开始连续匹配。

**能匹配上联合索引的查询：**
```sql
WHERE a = 1                    -- ✅ 使用索引
WHERE a = 1 AND b = 2          -- ✅ 使用索引
WHERE a = 1 AND b = 2 AND c = 3 -- ✅ 使用索引
```

**无法匹配联合索引的查询（索引失效）：**
```sql
WHERE b = 2                    -- ❌ 索引失效（没有a）
WHERE c = 3                    -- ❌ 索引失效（没有a、b）
WHERE b = 2 AND c = 3          -- ❌ 索引失效（没有a）
```

**范围查询的特殊情况**
- `WHERE a > 1 AND b = 2`：a 用到索引，b 无法利用索引（因为 a > 1 的范围内 b 无序）
- `WHERE a >= 1 AND b = 2`：a 和 b 都能用到索引（因为 a = 1 时 b 是有序的）
- `WHERE a BETWEEN 2 AND 8 AND b = 2`：a 和 b 都能用到索引
- `WHERE a LIKE 'j%' AND b = 2`：a 和 b 都能用到索引（前缀匹配不会中断）

**索引下推优化（MySQL 5.6+）**
在联合索引遍历过程中，对索引中包含的字段先做判断，直接过滤掉不满足条件的记录，减少回表次数。例如 `WHERE a > 1 AND b = 2`，会在联合索引内先判断 b 是否等于 2，再决定是否回表。

**导致索引失效的其他情况：**
1. 使用左模糊或左右模糊匹配 `LIKE %xx`、`LIKE %xx%`
2. 在索引列上使用计算、函数、类型转换
3. OR 前后条件列不都是索引列
4. 字符串不加单引号导致隐式类型转换

> 📚 来源：[小林 coding - MySQL 索引常见面试题](https://xiaolincoding.com/mysql/index/index_interview.html)

---

## 题目四：如何优化索引？有哪些实用的索引优化技巧？

### 参考答案

**1. 前缀索引优化**
对于大字符串字段，只使用前几个字符建立索引，可以减小索引体积，提高查询速度。

```sql
-- 对 name 字段的前 6 个字符建立索引
ALTER TABLE user ADD KEY (name(6));
```

局限性：无法用于 ORDER BY 和 GROUP BY，也无法用于覆盖索引。

**2. 覆盖索引优化**
让 SQL 查询的所有字段都存在于索引中，避免回表操作。

```sql
-- 如果经常查询商品名称和价格，建立联合索引 (product_id, name, price)
-- 这样查询时不需要回表
SELECT name, price FROM products WHERE product_id = 100;
```

**3. 主键自增优化**
使用自增主键，每次插入数据都是追加操作，不需要移动数据，不会发生页分裂，插入效率高。

非自增主键插入时可能插入到数据页中间，导致页分裂和大量内存碎片，影响查询效率。

**4. 索引列设为 NOT NULL**
- 可为 NULL 的列使索引统计和值比较更复杂
- NULL 值会占用额外存储空间（InnoDB 行格式至少用 1 字节存储 NULL 值列表）

**5. 联合索引字段顺序优化**
- 把区分度大的字段排在前面，这样更有可能被更多 SQL 使用到
- 区分度 = 字段不同值个数 / 表总行数
- 避免在区分度低的字段（如性别）上建索引

**6. 利用索引进行排序**
如果查询需要 ORDER BY 字段，可以建立联合索引利用索引的有序性，避免文件排序（Using filesort）。

> 📚 来源：[小林 coding - MySQL 索引常见面试题](https://xiaolincoding.com/mysql/index/index_interview.html)

---

## 题目五：MySQL 执行计划（EXPLAIN）中有哪些关键字段？如何通过它判断查询性能？

### 参考答案

**EXPLAIN 常见字段解读**

| 字段 | 含义 |
|------|------|
| `type` | 数据扫描类型，从高到低：const > eq_ref > ref > range > index > ALL |
| `key` | 实际使用的索引，为 NULL 表示未使用索引 |
| `key_len` | 索引使用的字节数，越大说明使用的索引字段越多 |
| `rows` | 预计扫描的行数，越少越好 |
| `Extra` | 附加信息，Using filesort/Using temporary 表示效率低 |

**type 字段详解（效率从高到低）：**
- `const`：主键或唯一索引与常量值比较，只返回一条记录
- `eq_ref`：多表联查，主键或唯一索引作为关联条件
- `ref`：非唯一索引扫描，返回多条记录
- `range`：索引范围扫描（<、>、in、between 等）
- `index`：全索引扫描（不读数据，只读索引）
- `ALL`：全表扫描，性能最差

**Extra 字段关键指标：**
- `Using filesort`：无法利用索引完成排序，使用了文件排序，效率低，应尽量避免
- `Using temporary`：使用了临时表保存中间结果，常见于 ORDER BY 和 GROUP BY，效率低，应避免
- `Using index`：使用了覆盖索引，不需要回表，效率高
- `Using index condition`：使用了索引下推优化

**优化目标：**
1. 让 type 尽量达到 ref 或 eq_ref 级别，避免 ALL 和 index
2. Extra 中避免出现 Using filesort 和 Using temporary
3. 尽量出现 Using index（覆盖索引）

> 📚 来源：[小林 coding - MySQL 索引常见面试题](https://xiaolincoding.com/mysql/index/index_interview.html)

---

## 延伸阅读

- [MySQL 索引底层原理详解](https://xiaolincoding.com/mysql/index/index_interview.html)
- [PostgreSQL 官方性能优化文档](https://www.postgresql.org/docs/current/performance-tips.html)
- [小林 coding - 图解 MySQL](https://xiaolincoding.com/mysql/base/transaction.html)

---

*本文档由 AI 自动整理生成，每日更新*
