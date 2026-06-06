# 数据库性能优化面试题集锦

>📅 日期：2026-06-06  
> 🏷️ 标签：MySQL | PostgreSQL | 数据库优化 | 面试题

---

## 题目一：MySQL 索引失效的常见场景有哪些？如何避免？

### 参考答案

**常见索引失效场景：**

1. **使用函数或运算**
   ```sql
   --失效：WHERE YEAR(create_time) = 2026
   -- 正确：WHERE create_time >= '2026-01-01' AND create_time < '2027-01-01'
   ```

2. **使用 LIKE 前导通配符**
   ```sql
   -- 失效：WHERE name LIKE '%张%'
   -- 可用：WHERE name LIKE '张%'
   ```

3. **类型转换**
   ```sql
   -- 失效：WHERE id = '123' (id为INT)
   -- 正确：WHERE id = 123
   ```

4. **OR 连接不同条件**
   ```sql
   -- 失效：WHERE id = 1 OR name = '张三'
   -- 正确：WHERE id = 1 UNION ALL SELECT ... WHERE name = '张三'
   ```

5. **联合索引违背最左前缀原则**
   ```sql
   -- 创建索引：INDEX idx(a,b,c)
   -- 失效：WHERE b = 1
   -- 可用：WHERE a = 1 或 WHERE a = 1 AND b = 1
   ```

6. **使用 NOT、<>、!=**
   ```sql
   -- 失效：WHERE status !=1
   -- 替代：业务层面处理，或用范围查询
   ```

**避免策略：**
- 保持 SQL 简洁，避免函数包裹索引列
- 建立合适的索引顺序，遵循最左前缀原则
- 使用覆盖索引减少回表
- 开启慢查询日志定期分析

> 📚 来源：https://developer.aliyun.com/article/1517610

---

## 题目二：Explain 执行计划中各个字段的含义是什么？如何根据它优化 SQL？

### 参考答案

**Explain关键字段解析：**

| 字段 | 含义 | 优化关注点 |
|------|------|-----------|
| **type** | 访问类型 | 最好到最差：system > const > eq_ref > ref > range > index > ALL |
| **key** | 实际使用索引 | 必须有值，否则未用到索引 |
| **rows** | 估算扫描行数 | 越大越需要优化 |
| **Extra** | 附加信息 | Using filesort/Using temporary 需要优化 |

**优化方向：**
1. **type 为 ALL**：全表扫描，需建索引
2. **rows 过大**：通过统计信息调整，或建索引
3. **Extra 显示 Using filesort**：利用索引消除排序
4. **Extra 显示 Using temporary**：优化 GROUP BY/ORDER BY

```sql
-- 示例分析
EXPLAIN SELECT * FROM orders WHERE status = 1 ORDER BY create_time DESC;
-- 若 type=ALL, rows=100000, Extra=Using filesort → 需要优化
-- 添加索引：ALTER TABLE orders ADD INDEX idx_status_time(status, create_time);
```

> 📚 来源：https://juejin.cn/post/6844903656855863309

---

## 题目三：MySQL InnoDB 与 PostgreSQL 的缓存机制有何区别？如何调优？

### 参考答案

**MySQL InnoDB 缓存机制：**
- **Buffer Pool**：InnoDB 自带缓存池，默认128MB，可调整到可用内存的 60-80%
- 缓存数据页（16KB/page）、索引页、插入缓冲、锁对象等
- 使用 LRU-K 算法淘汰冷数据

**PostgreSQL 缓存机制：**
- **Shared Buffers**：需手动配置，默认 128MB，建议设为内存的 25%
- **OS Cache**：依赖操作系统缓存，Linux 会自动缓存文件页
- **walbuffers、effective_cache_size**：辅助缓存参数

**调优建议：**

```ini
# MySQL my.cnf
innodb_buffer_pool_size = 8G  # 建议设为可用内存的60-80%
innodb_log_file_size = 1G     # 预写日志文件
innodb_flush_log_at_trx_commit = 2  # 折中方案：性能与安全平衡
```

```ini
# PostgreSQL postgresql.conf
shared_buffers = 8GB # 建议设为内存的25%
effective_cache_size = 24GB   #  optimizer用，影响计划选择
work_mem = 256MB # 排序/哈希操作的内存
```

> 📚 来源：https://www.postgresql.org/docs/current/runtime-config-resource.html

---

## 题目四：如何设计一个高效的数据库分页方案？LIMIT 偏移量过大有什么性能问题？

### 参考答案

**问题分析：**
```sql
-- 低效：偏移量越大越慢
SELECT * FROM orders ORDER BY id LIMIT 1000000, 10;
-- 原因：MySQL先扫描前1000010行，再丢弃前1000000行
```

**优化方案：**

**方案一：基于上一页ID的游标分页**
```sql
-- 第一页
SELECT * FROM orders ORDER BY id LIMIT 10;
-- 下一页：传入上一页最后一条的ID
SELECT * FROM orders WHERE id > last_id ORDER BY id LIMIT 10;
-- 时间复杂度：O(1)，不受偏移量影响
```

**方案二：延迟关联**
```sql
-- 先查索引，再关联回原表
SELECT t.* FROM orders t
INNER JOIN (
    SELECT id FROM orders ORDER BY id LIMIT 1000000, 10
) AS tmp ON t.id = tmp.id;
```

**方案三：使用标签记录法**
```sql
-- 用户翻页时记录当前页最后一条的索引值
-- 避免OFFSET计算，快速定位
```

**通用原则：**
- 数据量 < 10000 条：直接 LIMIT
- 数据量 10000-100000：优化 SQL + 索引
- 数据量 > 100000：必须用游标分页

> 📚 来源：https://tech.meituan.com/2016/03/24/mysql-index.html

---

## 题目五：数据库连接池的核心参数有哪些？如何根据业务场景调优？

### 参考答案

**核心参数解析：**

| 参数 | 说明 | 建议值 |
|------|------|--------|
| **initialSize** | 初始连接数 | 5-10 |
| **maxActive** | 最大活跃连接数 | 根据QPS调，一般50-200 |
| **maxIdle** | 最大空闲连接数 | 与maxActive接近 |
| **minIdle** | 最小空闲连接数 | 5-10 |
| **maxWait** | 最大等待时间(ms) | 3000-5000 |
| **testWhileIdle** | 空闲时检测有效性 | true |
| **validationQuery** | 连接有效性检测SQL | SELECT1 |

**调优策略：**

**高并发短连接场景：**
```properties
initialSize=10
maxActive=200
minIdle=10
maxWait=3000
```

**低并发长连接场景：**
```properties
initialSize=20
maxActive=50
minIdle=20
maxWait=60000
```

**常见问题排查：**
1. **连接超时**：maxWait 过小或慢查询占连接
2. **连接泄漏**：确保 finally 中关闭连接
3. **连接不足**：监控 activeSize，调大 maxActive

```java
// Druid连接池配置示例
DruidDataSource dataSource = new DruidDataSource();
dataSource.setInitialSize(10);
dataSource.setMaxActive(100);
dataSource.setMinIdle(10);
dataSource.setMaxWait(3000);
dataSource.setValidationQuery("SELECT 1");
dataSource.setTestWhileIdle(true);
```

> 📚 来源：https://github.com/alibaba/druid/wiki/%E9%85%8D%E7%BD%AE

---

## 📖 更多学习资源

- [MySQL高性能优化规范](https://www.jianshu.com/p/4dcae2532f93)
- [PostgreSQL Performance Best Practices](https://www.postgresql.org/docs/current/perf-opt.html)

---

*本面试题集由 AI 自动整理生成，如有问题欢迎指正*