# 数据库性能优化面试题

> 📅 日期：2026-08-19
> 🔧 技术栈：MySQL / PostgreSQL
> 📚 知识点：索引优化、查询优化、存储引擎、缓存策略

---

## 题目一：如何定位并优化慢查询？

### 问题描述
一个使用 InnoDB 引擎的 MySQL 表有 1000 万条数据，前端接口响应时间超过 5 秒，如何定位原因并优化？

### 参考答案

**定位慢查询的方法：**

```sql
-- 1. 开启慢查询日志
SET GLOBAL slow_query_log = 'ON';
SET GLOBAL long_query_time = 1;  -- 超过1秒记录
SET GLOBAL slow_query_log_file = '/var/log/mysql/slow.log';

-- 2. 查看慢查询日志
SHOW VARIABLES LIKE 'slow_query%';

-- 3. 使用 EXPLAIN 分析查询
EXPLAIN SELECT * FROM orders WHERE status = 1 AND create_time > '2026-01-01';

-- 4. 使用 PROFILE 分析执行过程
SET profiling = 1;
SELECT * FROM orders WHERE order_no = 'ABC123';
SHOW PROFILES;
```

**常见优化手段：**

| 优化方向 | 具体措施 |
|---------|---------|
| 索引优化 | 为 WHERE/ORDER BY/JOIN 条件创建合适索引 |
| SQL 重写 | 避免 SELECT *，减少子查询，使用 JOIN 替代 |
| 分页优化 | 使用游标分页，避免大偏移量 |
| 表结构 | 垂直分表、水平分表、分区表 |
| 配置调整 | 调整 innodb_buffer_pool_size、tmp_table_size |

**关键原则：**
-遵循最左前缀原则创建复合索引
-避免在索引列上使用函数或运算
-区分度高的列放在复合索引前面
-使用覆盖索引避免回表查询

**来源：** [腾讯云数据库慢查询优化指南](https://cloud.tencent.com/document/product/236/72591)

---

## 题目二：InnoDB 与 MyISAM 引擎的核心区别？如何选择？

### 问题描述
在设计表结构时，应该如何选择存储引擎？两者在性能上有何差异？

### 参考答案

**核心区别对比：**

| 特性 | InnoDB | MyISAM |
|-----|--------|--------|
| 事务支持 | ✅ 支持 ACID 事务 | ❌ 不支持 |
| 行级锁 | ✅ 支持 | ❌ 只支持表级锁 |
| 外键约束 | ✅ 支持 | ❌ 不支持 |
| 崩溃恢复 | ✅ 自动恢复 | ❌ 需手动修复 |
| 全文索引 | ⚠️ 5.6+支持 | ✅ 原生支持 |
| 索引缓存 | 共享存储 | 独立缓存 |
|  COUNT(*) | 全表扫描 | 优化存储 |

**性能差异场景：**

```sql
-- InnoDB 适用场景
-- 1. 需要事务支持的业务系统
-- 2. 并发写入量大（行级锁优势）
-- 3. 数据一致性要求高
-- 4. 表主键查询频繁

-- MyISAM 适用场景
-- 1. 只读或极少更新的数据仓库
-- 2. 全文检索需求（5.6之前版本）
-- 3. 静态报表统计（COUNT(*) 更快）
-- 4. 空间数据类型（GIS）
```

**MySQL 8.0+ 重要变化：**
- MyISAM 只读静态表，InnoDB 成为默认引擎
- InnoDB 支持多版本并发控制（MVCC）
- 优先选择 InnoDB 已成为行业最佳实践

**来源：** [MySQL 官方文档 - InnoDB 存储引擎](https://dev.mysql.com/doc/refman/8.0/en/innodb-introduction.html)

---

## 题目三：什么是索引下推？它如何提升查询性能？

### 问题描述
MySQL 5.6 引入了索引下推（Index Condition Pushdown），这个特性是如何工作的？

### 参考答案

**索引下推原理：**

索引下推是 MySQL 5.6 针对二级索引的优化，**在索引遍历过程中过滤数据**，减少回表次数。

**无索引下推的执行流程：**
```
查询：SELECT * FROM user WHERE name LIKE '张%' AND age = 25;

1. 使用 idx_name 索引找到所有 name LIKE '张%' 的主键 ID
2. 根据主键 ID 回表查询完整行数据
3. 在 Server 层过滤 age = 25
```

**有索引下推的执行流程：**
```
复合索引：(name, age)

1. 使用 idx_name_age 索引找到 name LIKE '张%' 的记录
2. **在索引层同时过滤 age = 25**（不需要回表）
3. 只将满足全部条件的记录回表查询
```

**验证方法：**
```sql
EXPLAIN SELECT * FROM user WHERE name LIKE '张%' AND age = 25;

-- Extra 列显示 Using index condition 表示启用索引下推
-- Extra 列显示 Using index 表示使用覆盖索引
```

**性能提升原因：**
- 减少回表次数，降低磁盘 I/O
- 减少 Server 层与 Engine 层数据传输
- 在 InnoDB 引擎层直接完成数据过滤

**来源：** [MySQL 索引下推详解](https://juejin.cn/post/6844904060623290382)

---

## 题目四：PostgreSQL 的 MVCC 机制如何实现？与 MySQL 有何区别？

### 问题描述
PostgreSQL 使用 MVCC（多版本并发控制）实现事务隔离，它是如何工作的？与 MySQL InnoDB 的 MVCC 有何不同？

### 参考答案

**PostgreSQL MVCC 实现原理：**

PostgreSQL 通过 **ACID 原则中的 Isolation** 实现并发控制，每个事务看到的是数据库的某个一致性快照。

```sql
-- PostgreSQL 事务隔离级别
SHOW TRANSACTION ISOLATION LEVEL;  -- 默认: Read Committed

-- 事务快照机制
BEGIN;
SELECT txid_current_snapshot();  -- 查看当前事务快照
-- 返回格式: 最小xid:最大xid:已提交xid列表
-- 例: 100:105:100,102,104
```

**关键数据结构：**

| 组件 | 作用 |
|-----|------|
| xmin | 插入这条记录的事务ID |
| xmax | 删除/更新这条记录的事务ID |
| t_xmin | 事务提交时间 |
| t_xmax | 事务结束时间 |
| ctid | 元组物理位置标识 |

**与 MySQL InnoDB 的区别：**

| 维度 | PostgreSQL MVCC | MySQL InnoDB MVCC |
|-----|----------------|------------------|
| 实现方式 | 多版本元组（行级） | Undo 日志链 + ReadView |
| 垃圾回收 | VACUUM 进程 | InnoDB Purge 线程 |
| 可见性判断 | 基于事务ID和快照 | 基于事务ID和Undo链 |
| 索引处理 | 索引不存储版本信息 | 聚簇索引包含事务ID |
| 典型问题 | VACUUM 不及时导致膨胀 | Long Transaction 阻塞 Purge |

**PostgreSQL 优化建议：**
```sql
-- 监控垃圾元组
SELECT relname, n_dead_tup, n_live_tup, 
       round(n_dead_tup * 100.0 / (n_live_tup + n_dead_tup), 2) AS dead_ratio
FROM pg_stat_user_tables
ORDER BY n_dead_tup DESC;

-- 配置自动清理
ALTER SYSTEM SET autovacuum = on;
ALTER SYSTEM SET autovacuum_vacuum_threshold = 50;
```

**来源：** [PostgreSQL 官方文档 - MVCC](https://www.postgresql.org/docs/current/mvcc.html)

---

## 题目五：如何设计分库分表方案？ShardingSphere 如何实现？

### 问题描述
当单表数据量超过千万级时，需要考虑分库分表，具体如何设计？ShardingSphere 等中间件如何实现？

### 参考答案

**分库分表策略：**

```
┌─────────────────────────────────────────────────┐
│                    逻辑库/逻辑表                   │
├─────────────────┬───────────────────────────────┤
│   分片节点 0     │        分片节点 1              │
│  ┌───────────┐  │   ┌───────────┐               │
│  │  user_0   │  │   │  user_1   │               │
│  │  order_0  │  │   │  order_1  │               │
│  └───────────┘  │   └───────────┘               │
└─────────────────┴───────────────────────────────┘
```

**分片算法：**

```sql
-- 1. 哈希分片：user_id % 4
-- 数据均匀，但扩缩容成本高

-- 2. 范围分片：按时间/ID区间
-- 查询友好，但可能热点集中

-- 3. 一致性哈希：解决扩容数据迁移问题
-- 虚拟节点环，迁移量可控

-- 4. 枚举/自定义分片
-- 按地域/业务类型分片
```

**ShardingSphere-JDBC 配置示例：**

```yaml
spring:
  shardingsphere:
    rules:
      sharding:
        tables:
          t_order:
            actual-data-nodes: ds_${0..1}.t_order_${0..15}
            table-strategy:
              standard:
                sharding-column: order_id
                sharding-algorithm-name: mod
            key-generate-strategy:
              column: order_id
              key-generator-name: snowflake
        sharding-algorithms:
          mod:
            type: INLINE
            props:
              algorithm-expression: t_order_${order_id % 16}
```

**跨分片查询解决方案：**

| 方案 | 适用场景 | 复杂度 |
|-----|---------|--------|
| 广播表 | 字典表、低频全量查询 | ⭐ |
| 绑定表 | 主从表关联查询 | ⭐⭐ |
| 联邦查询 | 跨库 SQL 支持 | ⭐⭐⭐ |
| 异构索引 | 异步同步 Elasticsearch | ⭐⭐ |
| 分页查询 | 假分页（限制返回条数）| ⭐⭐ |

**注意事项：**
- 分片键选择要避免数据倾斜
- 事务边界需要用分布式事务框架（如 Seata）
- 跨分片排序/聚合需要先拉取再内存处理
- 预留扩容空间，避免频繁重分片

**来源：** [ShardingSphere 官方文档](https://shardingsphere.apache.org/document/current/cn/overview/)

---

## 📚 更多资源

- [MySQL 8.0 Reference Manual](https://dev.mysql.com/doc/refman/8.0/en/)
- [PostgreSQL 15 Documentation](https://www.postgresql.org/docs/15/)
- [阿里云数据库内核月报](https://www.aliyun.com/product/drds)
- [字节跳动数据库技术博客](https://blog.doubao.com/)

---

*本面试题库由 GitHub 仓库 [qq286158530/interview-questions](https://github.com/qq286158530/interview-questions) 维护*
