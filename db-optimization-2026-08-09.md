# 数据库性能优化面试题

> 📅 更新时间：2026-08-09
> 🗄️ 涵盖：MySQL / PostgreSQL 索引优化、查询优化、存储引擎、缓存策略

---

## 题目一：MySQL 中 InnoDB 存储引擎的行锁是通过什么机制实现的？Gap Lock 和 Next-Key Lock 有什么区别？

### 参考答案

### InnoDB 行锁实现机制

InnoDB 的行锁基于**索引间锁**（Index-based locking）实现，而非直接对数据行加锁：

- 锁定的是索引记录，而非数据行本身
- 如果 UPDATE 语句走了主键索引 → 锁住主键索引
- 如果走了普通索引 → 锁住该普通索引 + 相关主键索引

**核心数据结构：**
```
锁信息存储在 ibdata1 系统表空间中（lock_sys）
每个锁是一个 (space_id, page_no, heap_no) → lock_t 的映射
```

### Gap Lock（间隙锁）

**定义**：锁住索引记录之间的"间隙"，防止其他事务在间隙中插入新记录。

**场景**：当使用 `WHERE` 条件进行范围查询时，InnoDB 会锁定扫描过的所有间隙。

```sql
-- 假设 orders 表有 index(status)
SELECT * FROM orders WHERE status = 'paid' FOR UPDATE;
-- 如果 status='paid' 的记录 id 范围是 100~200
-- 会锁定 (负无穷, 100), (100, 150), (150, 200), (200, 正无穷) 这些间隙
```

### Next-Key Lock（临键锁）

**定义**：Gap Lock + Record Lock 的组合，锁住索引记录本身 + 它前面的间隙。

**InnoDB 的默认行为**：在 `REPEATABLE READ` 隔离级别下，对于扫描到的每条记录，InnoDB 自动将行锁升级为 Next-Key Lock。

### 二者关键区别

| 特性 | Gap Lock | Next-Key Lock |
|------|---------|---------------|
| **锁定内容** | 仅间隙，不包含记录本身 | 间隙 + 记录本身 |
| **默认行为** | 仅在特定条件下使用 | RR 隔离级别的默认锁算法 |
| **防止幻读** | 可防止间隙中插入新记录（防插入） | 防止间隙中插入 + 防止记录被修改/删除 |
| **非唯一索引** | 范围查询时锁定多个间隙 | 每条记录都会形成 Next-Key Lock |

### 面试加分点

- **唯一索引**：UPDATE/DELETE 命中唯一记录时，仅加 Record Lock（不锁间隙），因为唯一性保证不会有"间隙"问题
- **幻读问题**：Next-Key Lock 是 InnoDB 在 RR 级别解决幻读的核心手段
- **死锁风险**：多个事务对相邻范围的 Next-Key Lock 容易引发死锁，建议按顺序访问索引

---

## 题目二：如何分表分库？ShardingSphere 和 MyCAT 有什么区别？如何选择？

### 参考答案

### 分表分库的常见策略

#### 垂直拆分（按业务）

```
用户库：users, user_profiles, user_settings
订单库：orders, order_items, payments
商品库：products, skus, inventory
```

#### 水平拆分（按数据量）

**Range 分片**：按 ID 区间划分（如 0~1000万在库1，1000万~2000万在库2）
- 优点：扩容简单，天然支持范围查询
- 缺点：数据热点不均（历史库冷、新库热）

**Hash 分片**：按 `shard_key % N` 划分
- 优点：数据分布均匀
- 缺点：跨分片查询困难，需要额外交叉路由

### ShardingSphere vs MyCAT 对比

| 维度 | ShardingSphere | MyCAT |
|------|---------------|-------|
| **架构类型** | 客户端分片（JDBC 代理）| 服务端分片（MySQL Proxy）|
| **部署模式** | 引入 jar 包，应用直连 DB | 独立部署为 MySQL 中间件 |
| **协议** | JDBC | MySQL Protocol |
| **配置方式** | YAML 配置文件 | schema.xml + rule.xml |
| **SQL 支持度** | 高（支持复杂 JOIN、聚合）| 中等（有限制，需避免跨分片 JOIN）|
| **事务支持** | XA 强事务 / 最大努力送达 | 弱事务 |
| **性能损耗** | 较低（无额外网络跳转）| 较高（多一跳 Proxy）|
| **运维复杂度** | 低（纯 Java，集成在应用中）| 高（独立服务，需要维护）|

### 如何选择

**选 ShardingSphere 的场景：**
- 微服务架构，团队有 Java 能力
- 对性能要求高，希望减少网络跳数
- 需要支持复杂事务（XA）
- 云原生环境，K8s 部署

**选 MyCAT 的场景：**
- 老旧项目，不方便改动应用代码
- 多语言环境（PHP/Python/Go），不想每个语言都适配 ShardingSphere
- 已有 MyCAT 经验，团队运维能力强

---

## 题目三：PostgreSQL 的 TOAST 机制是什么？它如何影响大字段存储和查询性能？

### 参考答案

### TOAST 是什么

TOAST = **The Oversized-Attribute Storage Technique**

PostgreSQL 使用固定页面大小（通常 8KB/16KB），当一行数据超过页面大小时，无法直接存储。TOAST 就是用来处理"超大型列值"的机制。

### TOAST 存储策略

PostgreSQL 对 `text`、`bytea`、`jsonb` 等可变长类型有 4 种存储策略：

| 策略 | 说明 | 适用场景 |
|------|------|---------|
| `PLAIN` | 不压缩，不存储外部 | 固定长度类型（int、timestamp）|
| `EXTENDED` | 可压缩，优先存储外部 | 需要频繁访问的大字段（默认）|
| `EXTERNAL` | 不压缩，仅存储外部 | 不需要搜索的大字段（如 HTML 内容）|
| `MAIN` | 可压缩，优先内部存储 | 最后备选，尽量不存储外部 |

### TOAST 工作原理

```
原始数据行（8KB limit）：
┌─────────────────────────────────┐
│ toast_table_oid │ toast_value_id │
├─────────────────────────────────┤
│ 指向 toast 表的指针（引用）        │
└─────────────────────────────────┘

Toast 表（独立存储超大列）：
┌──────────┬───────┬──────────┐
│ chunk_id │ seq   │ chunk_data│
│  1       │  0    │ 前 2KB   │
│  1       │  1    │ 中 2KB   │
│  1       │  2    │ 后 1KB   │
└──────────┴───────┴──────────┘
```

### TOAST 对性能的影响

**负面影响：**

1. **查询放大**：即使只查询 `SELECT id FROM orders WHERE status='paid'`，如果表有 TOAST 列，MySQL 只读主键索引，而 PG 仍需读取整行（包含 toast 指针），增加 I/O
2. **索引失效**：在 TOAST 列上创建 `LIKE '%xxx%'` 全文搜索，无法利用 BTree 索引
3. **表膨胀**：TOAST 数据存储在独立表，频繁更新会导致 toast 表膨胀

**优化方案：**

```sql
-- 1. 选择合适的存储策略
CREATE TABLE articles (
    id SERIAL PRIMARY KEY,
    title TEXT,
    content TEXT STORAGE EXTERNAL  -- 不压缩，减少 CPU 开销
);

-- 2. 将大字段分离到独立表（行式存储 + 列式补充）
CREATE TABLE article_content (
    article_id INT PRIMARY KEY REFERENCES articles(id),
    content TEXT
);
CREATE TABLE article_index (  -- 用于搜索的索引表
    article_id INT REFERENCES articles(id),
    content_tsvector TSVECTOR
);

-- 3. 使用 pg_toast 功能手动压缩
SELECT compress_data(my_large_column) FROM my_table;
```

---

## 题目四：MySQL 的慢查询日志如何分析？有哪些关键指标？如何通过 pt-query-digest 定位问题 SQL？

### 参考答案

### 慢查询日志配置

```ini
# my.cnf 配置
slow_query_log = 1
slow_query_log_file = /var/log/mysql/slow.log
long_query_time = 1        # 超过 1 秒的查询记入慢日志
log_queries_not_using_indexes = 1  # 记录未走索引的查询
min_examined_row_limit = 1000      # 至少扫描 1000 行才记录
```

### 关键分析指标

| 指标 | 含义 | 健康阈值 |
|------|------|---------|
| **Query_time** | 查询执行总时间 | 越低越好，>1s 需关注 |
| **Rows_examined** | 扫描行数 | 与返回行数比值越大越需优化 |
| **Rows_sent** | 返回行数 | 正常应远小于 Rows_examined |
| **Lock_time** | 等锁时间 | 高锁时间说明并发争抢严重 |
| **Filesort** | 是否产生文件排序 | 存在则需优化 |

### pt-query-digest 使用

```bash
# 安装
yum install percona-toolkit -y

# 分析慢查询日志（Top 10 最慢查询）
pt-query-digest /var/log/mysql/slow.log \
    --limit 10 \
    --filter '$event->{fingerprint} =~ m/^select/i' \
    > /tmp/slow_analysis.txt

# 分析结果解读
```

**pt-query-digest 输出结构：**

```
# 220ms user time, 40ms system time, 32.66M rss, 115.23M vsz
# Current date: Sat Aug  8 18:30:00 2026
# Runtime: 45.234s
# Schema: shop_db
# Query times: 0.005-2.345s, avg 0.234s
# Response time: 93.2% in 0.5s以内, 6.8% in 0.5-2.345s

# Profile
# Rank  Query ID          Response time  Calls  R/Call
#    1  0xD4A5B3C2...     12.345  45.2%   1234   0.010
#    2  0xE5F6A7B8...      8.234  30.1%    567   0.015
```

### 常见优化模式

**模式 1：全表扫描 + 排序**
```
SELECT * FROM orders ORDER BY created_at DESC LIMIT 10;
-- 问题：未加 LIMIT 时会 filesort，扫描全表
-- 解决：SELECT id FROM orders ORDER BY created_at DESC LIMIT 10 (利用索引)
```

**模式 2：隐式类型转换导致索引失效**
```
-- phone 列为 VARCHAR，传入数字
SELECT * FROM users WHERE phone = 13800138000;
-- 实际变成 WHERE CAST(phone AS SIGNED) = 13800138000
-- 解决：使用字符串 '13800138000'
```

---

## 题目五：如何设计一个支持每日千万级写入的 MySQL 分库分表方案？写入热点如何处理？

### 参考答案

### 整体架构设计

```
                    ┌─────────────────┐
                    │   DataX / Flink │  实时同步
                    │   CDC 同步      │
                    └────────┬────────┘
                             ▼
┌──────────┐   ┌──────────┐   ┌──────────┐   ┌──────────┐
│ Shard 1  │   │ Shard 2  │   │ Shard 3  │   │ Shard N  │
│ orders_0 │   │ orders_1 │   │ orders_2 │   │ orders_N │
└──────────┘   └──────────┘   └──────────┘   └──────────┘
      │              │              │              │
      └──────────────┴──────────────┴──────────────┘
                             │
                    ┌────────▼────────┐
                    │   Proxy/SDK     │  ShardingSphere-JDBC
                    │   (读写分离)    │
                    └────────┬────────┘
                             │
              ┌──────────────┴──────────────┐
              │                             │
       ┌──────▼──────┐               ┌──────▼──────┐
       │   写库集群   │               │   读库集群   │
       │ (3主从)     │               │ (3主从)     │
       └─────────────┘               └─────────────┘
```

### 分片策略选择

**按时间分片**（适合订单、流水等有明显时间属性的数据）：

```yaml
# ShardingSphere 分片配置示例
shardingRules:
  tables:
    orders:
      actualDataNodes: ds_${0..7}.orders_${202601..202612}
      tableStrategy:
        standard:
          shardingColumn: created_at
          shardingAlgorithmName: orders_by_month
      databaseStrategy:
        standard:
          shardingColumn: user_id
          shardingAlgorithmName: orders_db_by_user
```

### 写入热点解决方案

**问题**：按用户 ID 分片时，部分高活跃用户（如大主播、头部商家）集中在某个分片，形成热点。

**解决方案 1：二级分片**

```sql
-- 对热点用户再按时间戳二次路由
-- 路由公式：shard = hash(user_id) % N + (timestamp % M)
-- 其中 M 是时间片数量（如每小时一个时间片）

-- 应用层实现
def get_shard(user_id, timestamp):
    base_shard = hash(user_id) % 8
    time_shard = (timestamp // 3600) % 4  # 每4小时换一个虚拟分片
    return (base_shard * 4 + time_shard) % 32
```

**解决方案 2：写缓冲 + 批量写入**

```python
# 高频写入场景，用 Redis 缓冲，批量落库
import redis, pymysql, json
from datetime import datetime

r = redis.Redis(host='localhost', db=0)
buffer_key = 'orders:buffer:{shard_id}'

def write_order(order_data):
    shard_id = hash(order_data['user_id']) % 8
    r.rpush(buffer_key.format(shard_id=shard_id), json.dumps(order_data))
    
    # 每 100 条或每 1 秒，批量写入
    if r.llen(buffer_key.format(shard_id=shard_id)) >= 100:
        flush_shard(shard_id)

def flush_shard(shard_id):
    pipe = r.pipeline()
    for _ in range(100):
        pipe.lpop(buffer_key.format(shard_id=shard_id))
    orders = [json.loads(o) for o in pipe.execute() if o]
    
    if orders:
        sql = "INSERT INTO orders (id, user_id, amount, created_at) VALUES (%s, %s, %s, %s)"
        conn = get_conn(shard_id)
        cursor.executemany(sql, [(o['id'], o['user_id'], o['amount'], o['created_at']) for o in orders])
        conn.commit()
```

**解决方案 3：多主复制（Multi-Master）**

```
写入路径：App → LVS/Nginx → Master_1 / Master_2 / Master_3（轮询或随机）
          ↓              ↓              ↓
        Shard 0-2      Shard 3-5      Shard 6-8
```

### 扩容方案（平滑迁移）

1. **双写**：新旧分片同时写入
2. **后台补偿**：历史数据用 Canal/Debezium 同步
3. **灰度切读**：先读新，再读旧，逐步切流
4. **切完删旧**：确认无误后删除老数据

---

## 📚 参考来源

1. **MySQL InnoDB Locking**  
   https://dev.mysql.com/doc/refman/8.0/en/innodb-locking.html

2. **ShardingSphere 官方文档**  
   https://shardingsphere.apache.org/

3. **PostgreSQL TOAST 机制**  
   https://www.postgresql.org/docs/current/storage-toast.html

4. **Percona pt-query-digest 文档**  
   https://www.percona.com/doc/percona-toolkit/LATEST/pt-query-digest.html

5. **MySQL 分库分表最佳实践**  
   https://dev.mysql.com/doc/refman/8.0/en/partitioning.html

---

*本文件由 AI 自动生成并推送至 GitHub 仓库*
