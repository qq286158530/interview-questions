# 数据库性能优化面试题集

> 📅 日期：2026-06-09  
> 🏷️ 分类：MySQL / PostgreSQL / 数据库性能优化  
> 👨‍💻 适合：后端工程师、数据库管理员、DBA

---

## 题目一：MySQL 索引优化——如何定位并解决线上慢查询？

### 题目
线上某条 SQL 查询执行时间超过 3 秒，已影响业务响应。请描述你完整的排查和优化思路。

### 答案

**第一步：定位慢查询**

```sql
-- 开启慢查询日志
SET GLOBAL slow_query_log = 'ON';
SET GLOBAL long_query_time = 1; -- 超过1秒记录
SET GLOBAL slow_query_log_file = '/var/log/mysql/slow.log';

-- 查看当前会话慢查询状态
SHOW VARIABLES LIKE 'slow_query%';

-- 通过 performance_schema 实时分析
SELECT * FROM performance_schema.events_statements_summary_by_digest 
ORDER BY SUM_TIMER_WAIT DESC LIMIT 10;
```

**第二步：EXPLAIN 分析执行计划**

```sql
EXPLAIN FORMAT=JSON SELECT ... FROM ...
-- 重点关注：
-- - type: 至少达到 ref/index range，避免 ALL（全表扫描）
-- - key: 确认使用了索引
-- - rows: 预估扫描行数，越少越好
-- - Extra: 避免 Using filesort、Using temporary
```

**第三步：常见优化手段**

| 场景 | 优化方案 |
|------|---------|
| 索引失效 | 避免隐式类型转换，例如 `phone VARCHAR(20)` 查询时不用加引号 |
| 回表查询过多 | 覆盖索引：`SELECT id,name WHERE phone='138...'`，加联合索引 |
| 深分页 | 改用基于主键的范围查询：`WHERE id > last_id LIMIT 20` |
| JOIN 顺序错误 | 用 `STRAIGHT_JOIN` 强制驱动表，或加 hint |
| 统计信息过期 | `ANALYZE TABLE tb_name` 更新统计信息 |

**第四步：验证优化效果**

```sql
-- 对比优化前后的执行时间
SELECT SQL_NO_CACHE ... -- 禁用查询缓存
```

**来源：** [美团技术团队 - MySQL索引原理及优化](https://tech.meituan.com/2014/06/30/mysql-index.html)

---

## 题目二：InnoDB 与 MyISAM 存储引擎的核心差异？如何选型？

### 题目
项目开发中，MySQL 5.7 默认存储引擎是 InnoDB，而历史系统仍有 MyISAM 表。在什么场景下应该选择哪种引擎？

### 答案

### 核心差异对比

| 特性 | InnoDB | MyISAM |
|------|--------|--------|
| 事务支持 | ✅ 支持 ACID 事务 | ❌ 不支持 |
| 行级锁 | ✅ 支持，高并发 | ❌ 仅表级锁 |
| 外键约束 | ✅ 支持 | ❌ 不支持 |
| 崩溃恢复 | ✅ 自动恢复（redo log） | ❌ 需手动修复 |
| 全文索引 | ✅ 5.6+支持 | ✅ 支持 |
| 存储空间 | 约 2x MyISAM | 较小（压缩比高） |
| 写入性能 | 写优化 | 读略快 |

### 选型建议

**选 InnoDB：**
- 有事务需求（转账、订单、库存扣减）
- 高并发写入场景（>100并发）
- 数据可靠性要求高（如金融、订单系统）
- 需要外键约束保证引用完整性

**选 MyISAM：**
- 以读为主、极少更新的场景（如配置表、日志报表）
- 全文检索需求，且版本 < 5.6
- 追求极致的查询 QPS，放弃事务

### 实战建议

```sql
-- 查看当前引擎
SHOW CREATE TABLE tb_nameG;

-- 修改引擎
ALTER TABLE tb_name ENGINE = InnoDB;

-- 设置默认引擎
SET GLOBAL default_storage_engine = InnoDB;
```

**来源：** [阿里巴巴数据库内核月报 - InnoDB vs MyISAM](https://mysql.taobao.org/monthly/)

---

## 题目三：PostgreSQL 如何优化大表分页查询？深度分页优化策略？

### 题目
PostgreSQL 中 `SELECT * FROM orders ORDER BY id LIMIT 100 OFFSET 100000` 执行非常慢，如何优化？

### 答案

### 问题根源
`OFFSET` 越大，数据库仍需扫描前面的所有行，时间复杂度 O(n)。

### 优化策略

**策略一：游标分页（推荐）**

```sql
-- 基于主键的范围查询，性能稳定
SELECT * FROM orders 
WHERE id > last_id  -- last_id 为上一页最后一条的 id
ORDER BY id 
LIMIT 100;

-- 带条件的游标分页
SELECT * FROM orders 
WHERE id > last_id AND created_at > '2026-01-01'
ORDER BY id 
LIMIT 100;
```

**策略二：覆盖索引**

```sql
-- 创建覆盖索引，包含查询涉及的所有列
CREATE INDEX idx_orders_covered ON orders(created_at, id) 
INCLUDE (status, amount, customer_id);

-- 查询直接走索引，无需回表
SELECT id, status, amount FROM orders 
WHERE created_at > '2026-01-01' 
ORDER BY created_at 
LIMIT 100;
```

**策略三：延迟关联**

```sql
-- 先通过索引定位主键，再关联取完整数据
SELECT o.* FROM orders o
INNER JOIN (
    SELECT id FROM orders 
    WHERE created_at > '2026-01-01' 
    ORDER BY created_at 
    LIMIT 100 OFFSET 100000
) AS t ON o.id = t.id;
```

**策略四：分桶分页**

```sql
-- 按日期/月份分桶，用户先选月份再翻页
SELECT * FROM orders 
WHERE created_at BETWEEN '2026-06-01' AND '2026-06-30'
ORDER BY id 
LIMIT 100 OFFSET 500;

-- 配合分区表效果更佳
CREATE TABLE orders (
    id BIGSERIAL, ...
) PARTITION BY RANGE (created_at);
```

### 性能对比

| 方式 | 1000万数据 Offset=100000 | 性能 |
|------|--------------------------|------|
| 原始 OFFSET | 扫描10万行 | ~2000ms |
| 游标分页 | 扫描100行 | ~5ms |
| 覆盖索引 | 索引扫描 | ~3ms |

**来源：** [PostgreSQL中文文档 - 分页优化](https://www.postgresql.org/docs/current/using-explain.html)

---

## 题目四：MySQL/PostgreSQL 缓存策略——如何设计多级缓存架构？

### 题目
高并发系统如何设计缓存层，实现 Redis + 应用本地缓存 + 数据库的多级缓存？重点注意哪些问题？

### 答案

### 多级缓存架构图

```
┌─────────┐   ┌──────────┐   ┌────────────┐   ┌────────┐
│  用户   │──▶│ CDN/Nginx │──▶│ Local Cache │──▶│  Redis │──▶│  DB   │
└─────────┘   │  (Lua)   │   │ (Caffeine) │   │        │   └────────┘
              └──────────┘   └────────────┘   └────────┘
                    10ms          1ms            0.5ms      5ms
```

### 实战代码：Spring Boot 多级缓存

```java
@Service
public class ProductService {
    
    // L1: 本地缓存（Guava/Caffeine），TTL 30秒
    private final Cache<String, Product> localCache = 
        Caffeine.newBuilder()
            .maximumSize(10_000)
            .expireAfterWrite(30, TimeUnit.SECONDS)
            .build();
    
    // L2: Redis 分布式缓存，TTL 5分钟
    @Autowired private RedisTemplate<String, Product> redisTemplate;
    
    public Product getProduct(Long id) {
        String key = "product:" + id;
        
        // L1 查询
        Product product = localCache.getIfPresent(key);
        if (product != null) return product;
        
        // L2 查询
        product = redisTemplate.opsForValue().get(key);
        if (product != null) {
            localCache.put(key, product); // 写入L1
            return product;
        }
        
        // L3 查询数据库
        product = productMapper.selectById(id);
        if (product != null) {
            redisTemplate.opsForValue().set(key, product, 5, TimeUnit.MINUTES);
            localCache.put(key, product);
        }
        return product;
    }
    
    // 数据变更时清除缓存（CacheAside 模式）
    public void updateProduct(Product product) {
        productMapper.update(product);
        String key = "product:" + product.getId();
        localCache.invalidate(key);
        redisTemplate.delete(key); // 延迟双删
    }
}
```

### 重点注意事项

| 问题 | 解决方案 |
|------|---------|
| 缓存穿透 | 布隆过滤器 or 空值缓存（TTL 短） |
| 缓存击穿 | 互斥锁（Redis SETNX）或热点数据永不过期 |
| 缓存雪崩 | 随机 TTL + 热点数据预加载 |
| 数据一致性 | 延迟双删、订阅 Binlog（Canal）异步更新 |
| 缓存容量 | 合理评估大小，避免 OOM |

**来源：** [阿里云数据库技术 - Redis缓存设计最佳实践](https://help.aliyun.com/document_detail/159481.html)

---

## 题目五：如何监控和优化 PostgreSQL 慢查询与连接池？

### 题目
PostgreSQL 数据库 CPU 飙高、连接数接近上限，请给出完整的排查和优化流程。

### 答案

### 第一步：排查当前连接和慢查询

```sql
-- 查看当前连接数
SELECT count(*) FROM pg_stat_activity;

-- 查看所有连接详情
SELECT pid, usename, query_start, state, query 
FROM pg_stat_activity 
ORDER BY query_start;

-- 查看等待锁的连接
SELECT * FROM pg_stat_activity 
WHERE wait_event_type = 'Lock';

-- 查看最慢的查询
SELECT query, calls, total_time, mean_time 
FROM pg_stat_statements 
ORDER BY total_time DESC LIMIT 10;
```

### 第二步：定位性能瓶颈

```sql
-- 查看索引使用情况
SELECT schemaname, tablename, indexname, idx_scan, idx_tup_read 
FROM pg_stat_user_indexes 
ORDER BY idx_scan DESC LIMIT 20;

-- 查看表膨胀（需要插件 pgstattuple）
SELECT * FROM pgstattuple_approx('orders');

-- 分析慢查询执行计划
EXPLAIN (ANALYZE, BUFFERS, FORMAT TEXT) 
SELECT * FROM orders WHERE status = 'pending';
```

### 第三步：优化连接池（PgBouncer）

```ini
; pgbouncer.ini
[databases]
mydb = host=127.0.0.1 port=5432 dbname=mydb

[pgbouncer]
pool_mode = transaction          ; 推荐事务模式
max_client_conn = 1000
default_pool_size = 50           ; 每个库50个连接
min_pool_size = 10
server_idle_timeout = 600
query_timeout = 30               ; 超过30秒自动取消
```

```sql
-- PostgreSQL 配置优化
ALTER SYSTEM SET max_connections = 200;        -- 最大连接数
ALTER SYSTEM SET shared_buffers = '4GB';       -- 共享内存（不超过服务器75%）
ALTER SYSTEM SET effective_cache_size = '12GB';-- 估计可用缓存（服务器内存75%）
ALTER SYSTEM SET work_mem = '64MB';            -- 单个查询工作内存
ALTER SYSTEM SET maintenance_work_mem = '1GB'; -- 维护操作内存
ALTER SYSTEM SET random_page_cost = 1.1;       -- SSD设为1.1，HDD设为4.0

SELECT pg_reload_conf();
```

### 第四步：持续监控

```sql
-- 创建监控视图
CREATE VIEW pg_monitor AS
SELECT 
    schemaname,
    tablename,
    seq_scan, 
    idx_scan,
    n_tup_ins + n_tup_upd + n_tup_del as total_writes,
    n_live_tup, n_dead_tup,
    last_vacuum, last_autovacuum
FROM pg_stat_user_tables
WHERE schemaname = 'public';

-- 定期查看死元组
SELECT schemaname, tablename, n_dead_tup, last_autovacuum
FROM pg_stat_user_tables
WHERE n_dead_tup > 10000
ORDER BY n_dead_tup DESC;
```

### 常见问题处理

| 症状 | 可能原因 | 解决方案 |
|------|---------|---------|
| 连接数爆满 | 连接泄漏、未关闭事务 | 检查 `idle in transaction` 连接 + 调大 max_connections |
| CPU 100% | 慢查询、未建索引 | EXPLAIN 分析 + 加索引 |
| 内存飙升 | work_mem 过大、排序溢出 | 降低 work_mem + 加内存 |
| IO 飙升 | 频繁写日志、垃圾回收 | 调大 wal_buffers + VACUUM |

**来源：** [PostgreSQL官方文档 - pg_stat_statements](https://www.postgresql.org/docs/current/pgstatstatements.html)

---

## 📚 更多资源

- [GitHub题库仓库](https://github.com/qq286158530/interview-questions)
- [MySQL 8.0 Reference Manual](https://dev.mysql.com/doc/refman/8.0/en/)
- [PostgreSQL 16 Documentation](https://www.postgresql.org/docs/16/)

---

*本面试题由自动化脚本生成，每日更新 © 2026*
