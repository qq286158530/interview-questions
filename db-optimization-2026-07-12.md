# 数据库性能优化面试题

> 📅 2026-07-12

---

## 题目一：MySQL 索引失效的场景有哪些？

### 参考答案

**常见导致索引失效的场景：**

1. **使用 SELECT \*** 或查询不包含索引列
   - 只select需要字段，避免回表

2. **索引列参与运算或函数**
   ```sql
   -- 索引失效
   SELECT * FROM orders WHERE YEAR(created_at) = 2026;
   -- 正确写法
   SELECT * FROM orders WHERE created_at >= '2026-01-01' AND created_at < '2027-01-01';
   ```

3. **隐式类型转换**
   - 字段为字符串类型，传入数字；字段为整数，传入字符串
   - 如 `phone` 是 varchar，查询 `WHERE phone = 13800138000` 会导致全表扫描

4. ** LIKE 开头使用通配符**
   ```sql
   -- 索引失效
   WHERE name LIKE '%张%'
   WHERE name LIKE '%张'
   -- 索引有效
   WHERE name LIKE '张%'
   ```

5. **使用 OR 连接非索引列**
   ```sql
   -- 索引失效（age无索引）
   WHERE name = '张三' OR age = 25
   -- 改用 UNION
   WHERE name = '张三' UNION WHERE age = 25
   ```

6. **联合索引违反最左前缀原则**
   - 联合索引 `(name, age, city)`，查询 `WHERE age = 25` 不走索引

7. **使用 NOT、!=、<>、NOT IN、NOT EXISTS**
   - 优化器可能选择全表扫描

8. **WHERE 子句中对索引列使用 IS NULL / IS NOT NULL**
   - 部分场景可走索引，但数据分布不均时可能失效

9. **数据量过小时优化器选择全表扫描**
   - 小表直接走全表扫描可能更快

10. **统计信息不准确**
    - 执行 `ANALYZE TABLE` 更新统计信息

---

## 题目二：如何定位并优化慢查询？

### 参考答案

**定位慢查询的步骤：**

1. **开启慢查询日志**
   ```sql
   -- 查看慢查询配置
   SHOW VARIABLES LIKE 'slow_query%';
   SHOW VARIABLES LIKE 'long_query_time';
   
   -- 开启慢查询日志
   SET GLOBAL slow_query_log = 'ON';
   SET GLOBAL long_query_time = 1; -- 超过1秒记录
   
   -- 查看日志位置
   SHOW VARIABLES LIKE 'slow_query_log_file';
   ```

2. **使用 EXPLAIN 分析执行计划**
   ```sql
   EXPLAIN SELECT * FROM orders WHERE status = 1;
   ```
   关注字段：
   - `type`: ALL（全表扫描）> range > ref > eq_ref > const
   - `key`: 实际使用的索引
   - `rows`: 扫描行数
   - `Extra`: Using filesort、Using temporary 等警告

3. **使用 SHOW PROCESSLIST 查看实时连接**
   ```sql
   SHOW PROCESSLIST;
   -- 找出长时间运行的SQL
   ```

4. **使用 Performance Schema / sys schema**
   ```sql
   -- MySQL 5.6+ 启用
   SELECT * FROM performance_schema.events_statements_summary_by_digest 
   ORDER BY sum_timer_wait DESC LIMIT 10;
   ```

**优化方法：**

| 优化方向 | 具体措施 |
|---------|---------|
| 索引优化 | 添加合适索引，覆盖索引，避免回表 |
| SQL 改写 | 分解大查询，避免 SELECT \*，减少嵌套 |
| 分库分表 | 大表拆分，读写分离 |
| 业务层优化 | 减少查询频率，引入缓存 |

---

## 题目三：MySQL InnoDB 与 MyISAM 的区别？如何选择？

### 参考答案

| 特性 | InnoDB | MyISAM |
|------|--------|--------|
| 事务支持 | ✅ 支持 ACID 事务 | ❌ 不支持 |
| 行锁 | ✅ 支持行级锁 | ❌ 仅表级锁 |
| 外键 | ✅ 支持 | ❌ 不支持 |
| 全文索引 | ✅ 5.6+ 支持 | ✅ 原生支持 |
| 崩溃恢复 | ✅ 自动恢复 | ❌ 需手动修复 |
| 存储结构 | 表空间 + 行存储 | 3个文件/表 |
| 并发能力 | 高（行锁+MVCC） | 低（表锁） |
| 唯一/主键 | 允许有主键 | 主键唯一索引 |

**选择建议：**

- **选择 InnoDB**：生产环境、事务场景、高并发、需要数据可靠性
- **选择 MyISAM**：只读/低并发表、全文检索、历史归档表

**MySQL 8.0+ 变化：**
- MySQL 8.0 已移除 MyISAM 存储引擎相关优化，仅保留向后兼容

---

## 题目四：如何设计分库分表方案？说说一致性哈希？

### 参考答案

**分库分表常见策略：**

1. **垂直拆分**
   - 按业务模块拆分（用户库、订单库、商品库）
   - 单表字段拆分（冷热数据分离）

2. **水平拆分**
   - 按数据范围拆分（如按 ID 取模、按时间）
   - 按一致性哈希环分配数据

**一致性哈希（Consistent Hashing）：**

```
                     ┌──────┐
              ┌──────▶│ 节A  │◀────┐
              │       └──────┘     │
              │                   │
    哈希环 ───┼───▶ 数据A          │
              │                   │
              │       ┌──────┐    │
              └──────▶│ 节点B│────┘
                      └──────┘
```

**解决的问题：**
- 普通取模：节点宕机导致大量数据迁移
- 一致性哈希：节点增减只影响局部数据

**实现要点：**
```python
# 一致性哈希核心逻辑
import hashlib

def get_node(key, nodes, virtual_nodes=150):
    """根据key找到对应节点"""
    hash_val = int(hashlib.md5(str(key).encode()).hexdigest(), 16)
    
    # 找到第一个 >= hash_val 的节点
    for node in sorted(nodes):
        node_hash = int(hashlib.md5(node.encode()).hexdigest(), 16)
        if node_hash >= hash_val:
            return node
    return nodes[0]  # 环尾部回环到第一个节点
```

**常见中间件：** ShardingSphere、MyCat、Sharding-JDBC、TiDB

---

## 题目五：Redis 缓存与数据库一致性如何保证？

### 参考答案

**常见策略：**

### 策略1：Cache Aside（旁路缓存）—— 最常用

```
读操作：
1. 先查缓存 → 命中则返回
2. 未命中则查数据库 → 回填缓存 → 返回

写操作：
1. 先更新数据库
2. 删除缓存（而非更新）
```

**为什么删除而不是更新？**
- 更新缓存时如果并发请求可能导致脏数据
- 删除缓存下次查询自动回填，逻辑更简单

### 策略2：Read Through / Write Through

- Read Through：缓存自动从数据库加载
- Write Through：写缓存时同步写数据库

### 策略3：延迟双删（解决并发问题）

```python
def update_user(user_id, data):
    # 1. 先更新数据库
    db.update(user_id, data)
    
    # 2. 删除缓存
    redis.del(f"user:{user_id}")
    
    # 3. 延迟一段时间后再次删除（处理并发：旧数据回填后被清理）
    time.sleep(0.5)
    redis.del(f"user:{user_id}")
```

### 策略4：基于 MQ 的最终一致性

```
更新数据库 → 发送消息 → 消费者更新/删除缓存
```

### 策略5：BigKey 应对

- 避免缓存雪崩：key 过期时间 + 随机偏移
- 避免缓存击穿：互斥锁 / 逻辑过期

**如何选型：**

| 场景 | 推荐策略 |
|------|---------|
| 读多写少 | Cache Aside |
| 强一致性 | Write Through + 事务 |
| 高并发写 | 延迟双删 / MQ |
| 允许短暂不一致 | TTL + 最终一致性 |

---

> 📚 **参考来源**
> - https://dev.mysql.com/doc/refman/8.0/en/optimization.html
> - https://www.postgresql.org/docs/current/performance-tips.html
> - https://redis.io/docs/manual/patterns/
> - 《高性能MySQL（第3版）》
> - https://www.cnblogs.com/clsn/p/8214047.html
