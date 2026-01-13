## 1、索引优化

### 1、索引优化目标
**最终目标**：用树形结构将“线性搜索”变为“对数搜索”，用最少的 I/O 次数，最快地定位到所需数据行

- **减少磁盘 I/O**（走索引B+树IO次数很少，全表扫描IO次数很多）
- **避免回表**（减少二次回聚簇索引查找）
- **利用索引完成排序/分组**（避免 filesort / temporary）：排序不走索引，需要扫描所有匹配数据放入内存，如果数据太大，还要写入磁盘临时文件，进行外部排序，特别慢。ORDER BY 的字段顺序与索引一致，则走索引。
- **提升并发写入性能**（合理控制索引数量）：索引过多导致 INSERT/UPDATE/DELETE 索引维护成本上升，写性能下降

### 2、索引优化的方法

#### （1）选择合适的主键（聚簇索引优化），使用自增整数作为主键
- 插入顺序 = 存储顺序，顺序写入性能高，可以避免页分裂，而且范围查询效率高
- 而且主键值小 使得二级索引体积小（因叶子存主键）

#### （2）覆盖索引
- 如果 SELECT 的字段 + WHERE 条件字段 全部包含在某个索引中，InnoDB 可直接从二级索引 B+ 树的叶子节点返回结果，无需回表
- 不要盲目把所有字段塞进索引，索引过大维护成本和存储成本过高

#### （3）联合索引（Composite Index） + 最左前缀原则
- 将区分度高的列放前面
- 等值查询列放前，范围查询列放后
- 避免创建多个单列索引，一个索引支持多种查询组合（节省空间，写性能提升）

#### （4）避免无效索引 & 冗余索引
- 重复索引：idx_a(a), idx_a_b(a,b) → idx_a 是冗余的
- 未使用索引：长期未被查询使用的索引（可通过 sys.schema_unused_indexes 查看）
- 低区分度索引：如 gender（只有 M/F），全表扫描可能更快

优化方式：定期分析慢查询日志 + EXPLAIN，使用 SHOW INDEX FROM table 检查索引使用情况，删除无用索引（DROP INDEX）
好处：减少索引的维护，节省空间，写性能提升

#### （5）前缀索引（Prefix Index）—— 大字段优化
- 对 VARCHAR(255) 等大字段，只索引前 N 个字符，节省索引空间，但可能降低区分度。
- 适用于邮箱、URL、长字符串等高区分度大字段

好处：索引体积大幅减小，提升缓存命中率，前缀索引不能用于 ORDER BY / GROUP BY（因不完整）

#### （6）函数索引（MySQL 8.0+）
- 对表达式或函数结果建索引（如 UPPER(name)、YEAR(create_time)），适用于需对字段进行函数处理后查询的场景，避免全表扫描，简化应用逻辑。

#### （7）索引下推（ICP）
后续结合 MySQL 的结构来分析

## 2、Explain 分析慢查询日志

如下例子：
~~~ sql
EXPLAIN SELECT * FROM orders 
WHERE user_id = 1001 
  AND create_time > '2025-01-01'
ORDER BY amount DESC 
LIMIT 10;
~~~
查询结果如下：

| id | select_type | table | partitions | type | possible_keys | key | key_len | ref | rows | filtered | Extra |
|----|-------------|-------|------------|------|----------------|-----|---------|-----|------|----------|-------|
| 1  | SIMPLE      | orders| NULL       | ALL  | NULL           | NULL| NULL    | NULL| 987654| 10.00    | Using where; Using filesort |

包含12个字段：

### （1）id

含义：查询中每个 SELECT 的唯一标识。
作用：
id 相同 → 从上到下执行；
id 不同 → id 大的先执行（子查询）；
id 为 NULL → 最后执行（如 UNION 结果）。

### （2）select_type

含义：查询的类型。
常见值：
- SIMPLE	简单查询（无子查询/UNION）
- PRIMARY	最外层查询
- SUBQUERY	子查询（首次出现）
- DERIVED	派生表（如 FROM (SELECT ...)）
- UNION	UNION 中的第二个或之后的 SELECT

需要避免过多 DERIVED（会建临时表）。

### （3）table

含义：当前操作的表名。
特殊值：
<derivedN>：派生表（N 是子查询 id）
<unionM,N>：UNION 结果

### （4）partitions

含义：匹配的分区（如果表是分区表）。
非分区表：显示 NULL。

### （5）type ⭐⭐⭐（最重要！）
含义：**访问类型，反映 MySQL 如何查找行**。
性能从高到低排序：

![alt text](/复习资料/数据库/img/image.png)

### （6）possible_keys

含义：可能使用的索引列表。
注意：
显示 NULL ≠ 不能用索引，可能是优化器认为不用更快；有值但 key 为 NULL → 优化器选择了不用。

### （7）key ⭐⭐

含义：**实际使用的索引**。
关键判断：
NULL → 未走索引（危险！）；
与预期不符 → 索引设计不合理。

✅ 优化目标：确保高频查询命中合理索引。

### （8）key_len ⭐

含义：**使用的索引长度（字节）**。
用途：判断联合索引用到第几列。

🔍 计算规则（UTF8MB4）：
- INT → 4 字节
- BIGINT → 8 字节
- VARCHAR(N) → N×4 + 2（长度前缀）
- DATETIME → 5 字节
- 可为空字段 +1 字节

~~~ sql
-- 联合索引：idx(a INT, b VARCHAR(20), c DATETIME)
-- 查询：WHERE a=1 AND b='test'

-- key_len = 4 (a) + (20×4+2) (b) = 4 + 82 = 86
-- 如果 key_len=4 → 只用到 a 列！
~~~

### （9）ref

含义：与索引比较的列或常量。
常见值：
const：与常量比较（如 user_id = 1001）
db.table.column：与某列比较（JOIN 时）
func：与函数结果比较

### （10）rows ⭐⭐

含义：预估需要扫描的行数。

重要性：应接近慢查询日志中的 Rows_examined，越大越可能性能差，与 filtered 相乘 ≈ 实际返回行数。

### （11）filtered

含义：存储引擎返回的行中，预计有多少比例满足 WHERE 条件（百分比）。
计算：最终返回行数 ≈ rows × filtered / 100

### （12）Extra ⭐⭐⭐（最关键！）
含义：额外信息，揭示是否发生性能杀手操作。

必须关注的值：
![alt text](/复习资料/数据库/img/image-2.png)

需要关注的列：type、key、key_len、row、Extra.

## 3、查询性能优化

除了 **EXPLAIN 和索引优化**，还有 **优化数据访问** 和 **重构查询方式** 两种方式。

### （1）优化数据访问

核心思想：让 MySQL 少读、少传、少算。

- **只查询需要的列（避免 SELECT *）**
- **限制返回行数（LIMIT + 分页优化）** 

深分页性能杀手，如下SQL，MySQL 需扫描并排序前 1,000,020 行，仅返回最后 20 行，随偏移量增大，性能急剧下降。
~~~ sql
-- 跳过 100 万行，取 20 行
SELECT id, name FROM orders ORDER BY create_time DESC LIMIT 1000000, 20;
~~~

优化方案：基于游标分页,利用 (create_time, id) 索引直接定位起点，无论翻到第几页，始终 O(log N) 定位。
~~~ sql
-- 记住上一页最后一条的 create_time 和 id
SELECT id, name 
FROM orders 
WHERE create_time < '2023-10-01 12:00:00' OR (create_time = '2023-10-01 12:00:00' AND id < 9527)
ORDER BY create_time DESC, id DESC 
LIMIT 20;
~~~

- **提前过滤数据（将条件尽可能下推，查询过程中逐层过滤）**

低效写法：在应用层或外层过滤
~~~ sql
-- 先查出所有用户，再关联订单（可能数百万行）
SELECT u.name, o.amount
FROM users u
JOIN orders o ON u.id = o.user_id
WHERE u.country = 'CN'; -- 过滤条件在外层
~~~

 优化：在 JOIN 前过滤
~~~ sql
-- 方案1：子查询预过滤
SELECT u.name, o.amount
FROM (SELECT id, name FROM users WHERE country = 'CN') u
JOIN orders o ON u.id = o.user_id;

-- 方案2：将条件放入 JOIN ON（等价但更清晰）
SELECT u.name, o.amount
FROM users u
JOIN orders o ON u.id = o.user_id AND u.country = 'CN';
~~~

### （2）重构查询方式

核心思想：用多个简单查询替代一个复杂查询。

- **分解大连接查询**
  - 每个查询简单，易优化（可单独加索引）；
  - 减少单次查询持有锁的时间；
  - 可并行执行后续查询（如 CompletableFuture）；
  - 避免中间结果集过大导致磁盘临时表。

- **切分大查询**
  - 一次性更新/删除大量数据 -> 分批次操作

- **业务上：预计算 + 缓存替代实时计算**

## 4、复制

### （1）Mysql里面的各种日志

#### （1）bin log 日志

**所属层**：Server 层（与存储引擎无关）

**作用：**
- 记录所有对数据库的修改操作（DDL + DML，不含 SELECT）；
- 用于主从复制（从库拉取 Binlog 重放）；
- 用于基于时间点的数据恢复（PITR）

**开启条件**：需显式配置 log-bin = xxx

**典型场景：**
- 搭建主从集群；
- 误删数据后，用全量备份 + Binlog 恢复到故障前一刻。

#### （2）Redo Log（重做日志）

**所属层**：InnoDB 存储引擎层

**作用：**
- 保证事务的持久性（Durability）；
- 记录“物理页”的修改（如“将 page 100 的 offset 200 处写入 'abc'”）；
- 崩溃恢复时，**重做已提交但未刷盘的事务。**，因为SQL执行过程是先写 Redo Log（WAL 机制），再异步刷脏页到磁盘；

**典型场景：**
- MySQL 异常宕机后重启，InnoDB 自动通过 Redo Log 恢复未落盘的已提交事务。

#### （3）Undo Log（回滚日志）

**所属层**：InnoDB 存储引擎层

**作用**：
- 实现事务回滚（Rollback），保证原子性；
- 支持MVCC（多版本并发控制），提供一致性读（如 RR 隔离级别下读历史版本）；
- 记录数据修改前的“旧值”（逻辑日志）

**典型场景：**
- 执行 ROLLBACK 时恢复原数据；
- 一个长查询执行期间，其他事务更新了某行，该查询仍能读到事务开始时的快照。（MVCC）

#### （4）中继日志（Relay Log）

**所属层**：Server 层（仅从库存在）

**作用：**
- 从库接收主库的 Binlog 后，先写入本地 Relay Log；
- SQL 线程再从中继日志读取并重放，实现数据同步。

**结构**：格式与 Binlog 相同；
**自动管理**：SQL 线程执行完后自动清理（可通过 relay_log_purge 控制）。

✅ 典型场景：
主从复制架构中，从库断网重连后，可从中继日志断点续传，避免重新拉取全部 Binlog。

### （2）主从复制

MySQL 的复制（Replication）机制是实现**高可用、读写分离、数据备份、灾备容灾**的核心技术。

MySQL 复制通过 二进制日志（Binary Log, Binlog） 实现主库（Master）向从库（Slave/Replica）的数据同步，**复制是 异步 的（默认），主库不等待从库确认**。

整体的流程：
- **主库记录变更**：Binlog Dump Thread（主库）将所有 DDL/DML 操作写入 Binlog（需开启 log-bin）
- **从库拉取日志**：I/O Thread（从库）连接主库，请求 Binlog 事件，并写入本地 中继日志（Relay Log）
- **从库重放日志**：SQL Thread（从库）读取 Relay Log，重放 SQL 语句，使数据与主库一致

### （3）读写分离

**目标**：将读请求分流到从库，减轻主库压力，提升整体吞吐。

**主库只写，从库只读，减小主库压力。
**
![alt text](/复习资料/数据库/img/image-4.png)

#### （1）读写分离实现方式：
- 应用层手动路由（不推荐）：侵入业务代码，难以维护，仅适用于简单场景。
- 中间件代理（推荐）：ShardingSphere-Proxy / ShardingSphere-JDBC，**对应用透明，支持负载均衡、故障剔除**。

#### （2）读写分离的关键问题与解决方案

**主从延迟**：写入主库后，立即读从库可能读不到；
解决方案：
- 强制读主：对重要信息读取，“写后立即读”操作（如支付成功页），强制走主库；
- 延迟监控：通过 Seconds_Behind_Master 或心跳表监控延迟；
- 业务容忍：非核心读（如商品列表）可接受短暂延迟。
- 读写分离不解决写瓶颈：主库写压力大时，需分库分表

**从库不可用处理**：中间件应具备 故障自动剔除 能力，如 ShardingSphere 支持 health-check 自动下线异常从库。











