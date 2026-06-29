# 性能优化

达梦数据库性能优化完整指南，涵盖架构选型、参数调优、SQL优化、执行计划分析和统计信息管理。

---

## 数据库架构选型

根据业务场景选择合适的数据库架构是性能优化的第一步。

| 架构 | 适用场景 | 说明 |
|------|---------|------|
| 单机 | 一般业务 | 单实例部署，适合中小规模业务系统 |
| DMDataWatch | 高可用，主备秒级切换 | 主备架构，通过重做日志实现数据同步，支持自动故障切换 |
| DMRWC | 读多写少，OA办公 | 读写分离集群，写操作在主节点，读操作可分发到备节点 |
| DMDSC | 金融级高可用，多实例共享存储 | 共享存储集群，多实例同时读写，故障透明切换 |
| DMDPC | 海量数据，HTAP混合负载 | 分布式计算集群，支持海量数据的OLTP和OLAP混合负载 |

### 选型建议

- **数据量 < 500G，并发 < 200**：单机或主备即可
- **读写分离需求明显**：DMRWC，将报表查询分发到备库
- **要求 RPO=0、RTO < 30秒**：DMDSC 或 DMDataWatch
- **数据量 TB 级且需要实时分析**：DMDPC

---

## INI参数优化

DM数据库的主要配置文件为 `dm.ini`，参数分为静态（需重启生效）和动态（可在线修改）两类。

### 内存相关参数

内存是数据库性能最关键的因素，合理配置可大幅提升性能。

| 参数 | 含义 | 默认值 | 建议值 |
|------|------|--------|--------|
| MEMORY_POOL | 共享内存池大小(MB) | 400 | 高并发时调大，建议 1000-4000 |
| MEMORY_N_POOLS | 内存池分区个数 | 1 | 高并发时增大（如设置为 4-8），减少临界区竞争 |
| BUFFER | 数据缓冲区大小(MB) | 250 | 若数据量 < 物理内存，设为数据量大小；否则设为物理内存的 2/3 |
| BUFFER_POOLS | BUFFER 分区数 | 19 | 高并发时调大，范围 1~10000，建议 CPU 核数 × 2 |
| RECYCLE | 回收站缓冲区(MB) | 60 | 大量使用 WITH 子句、临时表、排序操作时调大至 200-500 |
| DICT_BUF_SIZE | 字典缓冲区(MB) | 5 | 数据库对象多或分区表数量大时调大至 20-50 |
| MAX_BUFFER | 最大缓冲区(MB) | 250 | 与 BUFFER 保持一致 |
| KEEP | 保留缓冲区(MB) | 8 | 用于缓存高频小表（如配置表），增加到 100-500 |

### 查询相关参数

| 参数 | 含义 | 默认值 | 建议值 |
|------|------|--------|--------|
| HJ_BUF_GLOBAL_SIZE | HASH JOIN 全局缓存(MB) | 100 | 内存充足时调大至 1000-4000，减少磁盘溢出 |
| HJ_BUF_SIZE | 单个 HASH JOIN 缓存(MB) | 50 | OLAP 场景按被驱动表数据量调大，建议 200-1000 |
| HAGR_BUF_GLOBAL_SIZE | HASH 分组聚集全局缓存(MB) | 100 | 大量 GROUP BY/SUM 操作时调大至 500-2000 |
| HAGR_BUF_SIZE | 单个 HASH 分组缓存(MB) | 100 | 大表分组时调大，建议 200-1000 |
| SORT_BUF_SIZE | 排序缓存(MB) | 2 | 建索引时调大至 10-20，不超过 20M |
| WORKER_THREADS | 工作线程数 | 4 | 设置为 CPU 核数或 2 倍，最大不超 1000 |
| MAX_SESSIONS | 最大会话数 | 100 | 根据业务并发连接数设置，建议为连接池大小 × 2 |
| CACHE_POOL_SIZE | SQL 缓存池大小(MB) | 20 | 高并发时调大至 100-500 |
| MTAB_MEM_SIZE | 内存临时表大小(KB) | 1024 | 临时表数据量大时调大至 4096-10240 |

### 优化器参数

| 参数 | 含义 | 默认值 | 建议值 |
|------|------|--------|--------|
| OLAP_FLAG | OLAP 标志 | 2 | OLTP 系统保持默认 2；纯分析系统设置为 1 |
| OPTIMIZER_MODE | 优化器模式 | 1 | 1=基于成本，保持默认 |
| ENABLE_MONITOR | 监控开关 | 2 | 优化排查时设为 3（记录详细监控信息），生产运行为 2 |
| ENABLE_HASH_JOIN | 允许 HASH JOIN | 1 | OLTP 中若 NEST LOOP 更优则设为 0 强制嵌套循环 |
| ENABLE_INDEX_JOIN | 允许索引连接 | 1 | 1=启用 |
| ENABLE_MERGE_JOIN | 允许归并连接 | 1 | 1=启用 |
| SUBQ_CVT_SPL_FLAG | 子查询优化标志 | 2 | 按位组合：1=子查询转换为内连接，2=子查询拆分为独立查询 |
| PK_WITH_CLUSTER | 主键使用聚集索引 | 1 | 1=主键默认聚集索引 |
| COMPATIBLE_MODE | 兼容模式 | 0 | 0=不兼容/DM原生，1=SQL92，2=Oracle，3=SQL Server，4=MySQL，5=DM6，6=Teradata，7=PostgreSQL，8=DB2 |
| ENABLE_RQ_TO_NONREF_SPL | 非相关子查询拆分 | 0 | 设为 1 有助于子查询优化 |

### 参数修改方法

参数修改有多种方式，根据参数类型和作用范围选择：

```sql
-- ====================
-- 1. 动态参数：立即生效，scope=1
-- ====================
SP_SET_PARA_VALUE(1, 'BUFFER', 2000);
SP_SET_PARA_VALUE(1, 'HJ_BUF_SIZE', 500);

-- ====================
-- 2. 静态参数：需重启数据库，scope=2
-- ====================
SP_SET_PARA_VALUE(2, 'COMPATIBLE_MODE', 2);

-- ====================
-- 3. ALTER SYSTEM：修改全局参数
-- ====================
ALTER SYSTEM SET 'MTAB_MEM_SIZE' = 1200 SPFILE;  -- 静态参数，写入 spfile，下次启动生效
ALTER SYSTEM SET 'BUFFER' = 2048 BOTH;           -- 同时修改内存和 spfile（动态参数适用）

-- ====================
-- 4. ALTER SESSION：仅当前会话生效
-- ====================
ALTER SESSION SET 'HAGR_HASH_SIZE' = 2000000;

-- ====================
-- 5. 查询参数当前值
-- ====================
SELECT SF_GET_PARA_VALUE(1, 'BUFFER');              -- 查询动态值
SELECT SF_GET_PARA_VALUE(2, 'BUFFER');              -- 查询静态（文件）值
SELECT * FROM V$DM_INI WHERE PARA_NAME LIKE 'PK_WITH%';  -- 查询所有参数
SELECT * FROM V$PARAMETER WHERE NAME LIKE 'BUFFER%';     -- 查看参数信息

-- ====================
-- 6. 批量查询关键参数
-- ====================
SELECT PARA_NAME, PARA_VALUE 
FROM V$DM_INI 
WHERE PARA_NAME IN ('BUFFER','MEMORY_POOL','WORKER_THREADS','MAX_SESSIONS');
```

---

## SQL优化

### 定位慢SQL

生产环境中定位慢SQL是性能优化的起点。有以下几种方法：

#### 方法1：SQL日志分析（推荐）

**Step 1：开启SQL日志**

修改 `dm.ini`：
```ini
SVR_LOG = 1    -- 启用SQL日志
```

**Step 2：配置 `sqllog.ini`**

在数据库目录下创建 `sqllog.ini`：
```ini
BUF_TOTAL_SIZE = 10240      -- 日志缓冲区总大小(KB)
BUF_SIZE = 1024             -- 单块缓冲区大小(KB)
[SLOG_ALL]
FILE_PATH = ../log          -- 日志文件路径（相对 dm.ini 所在目录）
SWITCH_MODE = 2             -- 切换模式：1=按文件数，2=按文件大小
SWITCH_LIMIT = 1024         -- 每个日志文件大小限制(MB)
ASYNC_FLUSH = 1             -- 1=异步刷盘（推荐，避免影响SQL执行性能）
FILE_NUM = 200              -- 日志文件数量（循环覆盖）
SQL_TRACE_MASK = 1          -- 日志掩码（记录哪些SQL）
MIN_EXEC_TIME = 0           -- 最小执行时间(ms)，0=全部记录，建议1=超10ms
```

**SQL_TRACE_MASK 掩码说明：**

| 位号 | 含义 | 使用场景 |
|------|------|----------|
| 1 | 全部SQL | 全面诊断时使用 |
| 2 | DML语句 | 仅关注增删改 |
| 3 | DDL语句 | 关注结构变更 |
| 4 | UPDATE语句 | 仅UPDATE |
| 5 | DELETE语句 | 仅DELETE |
| 6 | INSERT语句 | 仅INSERT |
| 7 | SELECT语句 | 仅SELECT（生产推荐） |
| 23 | 错误语句 | 排查SQL错误 |
| 24 | 需记录执行信息的语句 | 需配合其他位使用 |
| 25 | 打印执行计划和执行时间 | 非常实用！生产分析首推 |
| 29 | 事务相关语句 | 排查事务问题 |

```sql
-- 使配置动态生效（无需重启）
SP_REFRESH_SVR_LOG_CONFIG();
```

**Step 3：分析SQL日志**

- 使用 `DMLOG` 工具：`./dmlog PATH_TO_LOG_FILE`
- 使用正则表达式快速筛选慢SQL：`[1-9][0-9][0-9][0-9][0-9](ms)` 匹配 10 秒以上的SQL
- 使用 grep：`grep -E '[0-9]{5,}ms' sqllog.log`

#### 方法2：V$LONG_EXEC_SQLS 视图

查询当前数据库中执行时间长且尚未结束的SQL：

```sql
SELECT 
    SESS_ID,
    SQL_TEXT,
    EXEC_TIME,         -- 执行时间(ms)
    REMAIN_TIME,       -- 剩余估算时间(ms)
    START_TIME,
    N_READS,           -- 物理读次数
    N_LOGIC_READS      -- 逻辑读次数
FROM V$LONG_EXEC_SQLS
ORDER BY EXEC_TIME DESC;
```

#### 方法3：V$SQL_HISTORY 视图

查询已执行完的SQL历史：

```sql
SELECT 
    SESS_ID,
    SQL_TXT,
    EXEC_TIME,          -- 执行时间(ms)
    DISK_READS,          -- 物理读次数
    BUF_READS,           -- 逻辑读次数
    LAST_RECV_TIME
FROM V$SQL_HISTORY
WHERE EXEC_TIME > 1000   -- 执行超过1秒的SQL
ORDER BY EXEC_TIME DESC;
```

#### 方法4：实时观察当前执行的SQL

```sql
SELECT 
    SESS_ID,
    STATE,
    CLNT_IP,
    SQL_TEXT,
    (SYSDATE - LAST_RECV_TIME) * 24 * 3600 AS ELAPSED_SECONDS
FROM V$SESSIONS 
WHERE STATE = 'ACTIVE'
  AND SQL_TEXT IS NOT NULL
ORDER BY LAST_RECV_TIME;
```

### 执行计划

理解执行计划是SQL优化的核心技能。

#### 执行计划的读取规则

**自右向左、自下而上**的顺序读取：
- 最右侧且最上方的操作最先执行
- 同一层级中，上方的操作先于下方的执行
- 缩进表示父子关系：子节点为父节点提供数据

#### 核心操作符详解

| 操作符 | 全称 | 含义 | 性能影响 | 优化方向 |
|--------|------|------|----------|----------|
| CSCN | Cluster Scan | 全表扫描（聚集索引扫描） | **高开销** | 尽量避免；添加过滤条件可用的索引 |
| SSCN | Secondary Scan | 全索引扫描 | **中等开销** | 优于CSCN；考虑是否需要全部扫描 |
| SSEK | Secondary Seek | 二级索引等值查找 | **低开销** | 如需BLKUP回表且开销高，考虑聚集索引 |
| CSEK | Cluster Seek | 聚集索引等值查找 | **最优** | 无需回表，直接访问数据行 |
| NEST LOOP | Nested Loop Join | 嵌套循环连接 | 取决于驱动表大小 | 被驱动表连接列**必须**有索引；小表作为驱动表 |
| HASH JOIN | Nested Loop Join(?) | 哈希连接 | 取决于内存 | HJ_BUF_SIZE 需足够大；适用于大数据量等值连接 |
| MERGE JOIN | Merge Join | 归并连接 | 取决于排序 | 两侧数据必须有序；适用于大数据量有序数据连接 |
| HAGR | Hash Aggregate | HASH分组聚集 | **较高** | 比SAGR慢；尽量走索引使分组有序 |
| SAGR | Stream Aggregate | 流分组聚集 | **低** | 要求分组列有序（有索引支持） |
| BLKUP | Block Lookup | 二级索引回表 | **额外开销** | 开销大时改用聚集索引或创建覆盖索引 |
| SLCT | Selection | 选择过滤器 | 取决于过滤条件 | 过滤性好的列建议单独建索引 |
| PRJT | Project | 投影 | 低 | 一般无需优化 |
| NSET | Result Set | 结果集收集 | — | 最外层操作符 |
| STAT | Statistics | 统计信息收集 | 低 | 显示操作各阶段的统计信息 |

#### 关键性能指标

执行计划中需要重点关注的指标：

| 指标 | 含义 | 关注点 |
|------|------|--------|
| logical reads | 逻辑读次数 | 越小越好，高则考虑加索引或优化SQL |
| physical reads | 物理读次数 | 需从磁盘读取，尽量避免。出现较多说明BUFFER不够 |
| sort(disk) | 磁盘排序 | 出现说明排序缓冲区不足，建议调大SORT_BUF_SIZE |
| rows | 行数估计 vs 实际 | 差异大说明统计信息不准，需要重新收集 |

#### 获取执行计划

```sql
-- ====================
-- 方法1：预估执行计划（不实际执行SQL）
-- ====================
EXPLAIN SELECT e.employee_name, d.department_name
FROM employees e
JOIN departments d ON e.department_id = d.department_id
WHERE e.salary > 10000;

-- ====================
-- 方法2：AUTOTRACE（实际执行SQL并输出统计信息）
-- ====================
SET AUTOTRACE ON;           -- 执行SQL并显示结果+执行计划+统计
SET AUTOTRACE TRACEONLY;    -- 执行SQL仅显示执行计划+统计（不显示结果）
SET AUTOTRACE OFF;          -- 关闭

SELECT * FROM orders WHERE order_date > '2025-01-01';

-- ====================
-- 方法3：DBMS_SQLTUNE 获取实际执行计划
-- ====================
SELECT * FROM TABLE(DBMS_SQLTUNE.REPORT_SQL_MONITOR(sql_id => 'xxxxx'));

-- ====================
-- 方法4：10053 trace 事件（类似Oracle）
-- ====================
ALTER SESSION SET EVENTS '10053 TRACE NAME CONTEXT FOREVER, LEVEL 1';
-- 执行目标SQL
ALTER SESSION SET EVENTS '10053 TRACE NAME CONTEXT OFF';
```

### SQL优化技巧

#### 1. 索引优化

```sql
-- ====================
-- 创建合适的索引
-- ====================
-- 单列索引：用于等值或范围过滤
CREATE INDEX idx_emp_dept ON employees(department_id);

-- 复合索引：注意列顺序！高选择性的列放前面
CREATE INDEX idx_emp_dept_sal ON employees(department_id, salary);

-- 覆盖索引：包含查询所需的所有列，避免回表
CREATE INDEX idx_emp_cover ON employees(department_id, employee_name, salary);

-- 函数索引：对列函数结果建索引
CREATE INDEX idx_emp_upper_name ON employees(UPPER(employee_name));

-- 查看索引使用情况
SELECT * FROM V$OBJECT_USAGE WHERE TABLE_NAME = 'EMPLOYEES';
```

#### 2. SQL写法优化

```sql
-- ❌ 避免：对索引列使用函数，导致索引失效
SELECT * FROM employees WHERE UPPER(employee_name) = 'SMITH';

-- ✅ 如果必须使用函数，创建函数索引
CREATE INDEX idx_upper_name ON employees(UPPER(employee_name));

-- ❌ 避免：隐式类型转换
SELECT * FROM orders WHERE order_id = '12345';  -- order_id是NUMBER类型

-- ✅ 正确
SELECT * FROM orders WHERE order_id = 12345;

-- ❌ 避免：SELECT *（增加了不必要的回表和传输开销）
SELECT * FROM employees;

-- ✅ 只查询需要的列
SELECT employee_id, employee_name, salary FROM employees;

-- ❌ 避免：不必要的 DISTINCT
SELECT DISTINCT department_id FROM (SELECT department_id FROM employees);

-- ✅ 直接查询
SELECT department_id FROM employees;

-- ❌ 避免：在 WHERE 中使用 != 或 <>（可能不走索引）
-- ✅ 尽量使用等值条件或 IN / BETWEEN
```

#### 3. 连接优化

```sql
-- ====================
-- 确保连接列有索引
-- ====================
-- orders.customer_id 和 customers.customer_id 都应有索引

-- NEST LOOP适用场景：小表驱动大表
SELECT /*+ USE_NL(o c) */ * 
FROM orders o JOIN customers c ON o.customer_id = c.customer_id
WHERE o.order_date = '2025-06-01';

-- HASH JOIN适用场景：两个大表等值连接
SELECT /*+ USE_HASH(o c) */ * 
FROM orders o JOIN customers c ON o.customer_id = c.customer_id;

-- MERGE JOIN适用场景：排序后的数据连接
SELECT /*+ USE_MERGE(o c) */ * 
FROM orders o JOIN customers c ON o.customer_id = c.customer_id;

-- HINT语法
SELECT /*+ INDEX(employees idx_emp_dept) */ * 
FROM employees WHERE department_id = 10;
```

#### 4. 子查询优化

```sql
-- ❌ 避免：关联子查询（每行执行一次）
SELECT e.*, 
    (SELECT d.department_name FROM departments d WHERE d.department_id = e.department_id) dept_name
FROM employees e;

-- ✅ 使用 JOIN 替代
SELECT e.*, d.department_name
FROM employees e
LEFT JOIN departments d ON e.department_id = d.department_id;

-- ❌ 避免：IN 大结果集
SELECT * FROM orders WHERE customer_id IN (SELECT customer_id FROM large_table);

-- ✅ 使用 EXISTS 或 JOIN
SELECT o.* FROM orders o WHERE EXISTS (SELECT 1 FROM large_table t WHERE t.customer_id = o.customer_id);
```

#### 5. 批量操作优化

```sql
-- ❌ 逐条插入（慢）
INSERT INTO target VALUES (1, 'A');
INSERT INTO target VALUES (2, 'B');
...
INSERT INTO target VALUES (1000, 'Z');

-- ✅ 批量插入
INSERT INTO target 
SELECT * FROM source WHERE condition;

-- ✅ 或使用 MERGE
MERGE INTO target t
USING source s ON (t.id = s.id)
WHEN MATCHED THEN UPDATE SET t.name = s.name
WHEN NOT MATCHED THEN INSERT (id, name) VALUES (s.id, s.name);
```

---

## 统计信息

统计信息是优化器生成正确执行计划的基础。**不准确或缺失的统计信息是慢SQL最常见的原因之一！**

### 为什么要更新统计信息

- 优化器根据统计信息估算行数和选择性
- **迁移后必须更新**：数据迁移后旧的统计信息完全失效
- **大批量数据变更后必须更新**：新增/删除大量数据后统计信息过时
- 统计信息不准 → 错误的执行计划 → 性能严重下降

### 收集统计信息

```sql
-- ====================
-- 1. 按模式收集（最常用，收集整个模式所有对象的统计信息）
-- ====================
-- 参数：模式名, 采样百分比, 是否强制刷新, 选项
DBMS_STATS.GATHER_SCHEMA_STATS(
    '模式名',                      -- schema_name
    100,                           -- estimate_percent: 采样百分比，100=全量
    FALSE,                         -- force: TRUE=强制刷新
    'FOR ALL COLUMNS SIZE AUTO'    -- options: 收集所有列的直方图(AUTO=自动决定)
);

-- ====================
-- 2. 按表收集
-- ====================
DBMS_STATS.GATHER_TABLE_STATS(
    '用户名',                      -- ownname
    '表名',                        -- tabname
    NULL,                          -- partname: 分区名(NULL=全部分区)
    100,                           -- estimate_percent
    TRUE,                          -- force: 是否强制
    'FOR ALL COLUMNS SIZE AUTO'    -- options
);

-- ====================
-- 3. 按索引收集
-- ====================
DBMS_STATS.GATHER_INDEX_STATS('用户名', '索引名');

-- ====================
-- 4. 收集数据库级别的统计信息（慎重，耗时长）
-- ====================
DBMS_STATS.GATHER_DATABASE_STATS(100, FALSE, 'FOR ALL COLUMNS SIZE AUTO');

-- ====================
-- 5. 收集系统统计信息
-- ====================
DBMS_STATS.GATHER_SYSTEM_STATS();

-- ====================
-- 6. 查看统计信息
-- ====================
-- 查看表的统计信息
SELECT 
    TABLE_NAME,
    NUM_ROWS,           -- 行数
    BLOCKS,             -- 块数
    AVG_ROW_LEN,        -- 平均行长
    LAST_ANALYZED       -- 最后分析时间
FROM USER_TABLES
WHERE TABLE_NAME = 'EMPLOYEES';

-- 查看列的统计信息
SELECT 
    COLUMN_NAME,
    NUM_DISTINCT,       -- 不同值数量
    NUM_NULLS,          -- NULL值数量
    HISTOGRAM,          -- 直方图类型
    LAST_ANALYZED
FROM USER_TAB_COLUMNS
WHERE TABLE_NAME = 'EMPLOYEES';

-- 查看索引的统计信息
SELECT 
    INDEX_NAME,
    BLEVEL,             -- 索引层级
    LEAF_BLOCKS,        -- 叶子块数
    DISTINCT_KEYS,      -- 不同键值数量
    LAST_ANALYZED
FROM USER_INDEXES
WHERE TABLE_NAME = 'EMPLOYEES';
```

### 统计信息管理

```sql
-- ====================
-- 锁定统计信息（防止自动收集覆盖）
-- ====================
DBMS_STATS.LOCK_TABLE_STATS('用户名', '表名');

-- ====================
-- 解锁统计信息
-- ====================
DBMS_STATS.UNLOCK_TABLE_STATS('用户名', '表名');

-- ====================
-- 删除统计信息
-- ====================
DBMS_STATS.DELETE_TABLE_STATS('用户名', '表名');

-- ====================
-- 导出/导入统计信息
-- ====================
-- 创建统计信息表
DBMS_STATS.CREATE_STAT_TABLE('用户名', 'STAT_BACKUP');

-- 导出到备份表
DBMS_STATS.EXPORT_TABLE_STATS('用户名', '表名', NULL, 'STAT_BACKUP', '备份标记', TRUE);

-- 从备份表导入
DBMS_STATS.IMPORT_TABLE_STATS('用户名', '表名', NULL, 'STAT_BACKUP', '备份标记', TRUE);
```

### 统计信息自动收集

建议配置定时作业自动收集统计信息：

```sql
-- ====================
-- 创建定时作业：每天凌晨2点自动收集所有模式的统计信息
-- ====================
CALL SP_INIT_JOB_SYS(1);

CALL SP_CREATE_JOB('AUTO_GATHER_STATS', 1, 0, '', 0, 0, '', 0, '自动收集统计信息');

CALL SP_JOB_CONFIG_START('AUTO_GATHER_STATS');

CALL SP_ADD_JOB_STEP('AUTO_GATHER_STATS', 'GATHER', 0,
    'BEGIN
        DBMS_STATS.GATHER_SCHEMA_STATS(''SCHEMA1'', 30, FALSE, ''FOR ALL COLUMNS SIZE AUTO'');
        DBMS_STATS.GATHER_SCHEMA_STATS(''SCHEMA2'', 30, FALSE, ''FOR ALL COLUMNS SIZE AUTO'');
    END;',
    1, 2, 0, 0, NULL, 0);

CALL SP_ADD_JOB_SCHEDULE('AUTO_GATHER_STATS', 'SCHEDULE', 
    1,                              -- 启用
    1,                              -- 按天
    1,                              -- 每1天执行一次
    0,                              -- 不分时
    0,                              -- 不分分
    '02:00:00',                     -- 凌晨2点
    NULL, 
    '2025-01-01 00:00:00',          -- 起始日期
    NULL, 
    '');

CALL SP_JOB_CONFIG_COMMIT('AUTO_GATHER_STATS');
```

### 重要注意事项

> ⚠️ **数据迁移后必须立即更新统计信息！**
>
> 大批量数据导入/删除后，旧的统计信息会导致优化器生成完全错误的执行计划，可能造成：全表扫描替代索引扫描、错误的连接顺序和连接方法、不合理的并行度等。
>
> **最佳实践：**
> - ETL 完成后统一执行 `DBMS_STATS.GATHER_SCHEMA_STATS`
> - 表数据变化超过 10% 时重新收集
> - 生产环境使用 30% 采样而非 100%，平衡准确度和收集时间
> - 开发/测试环境使用 100% 采样获得最准确的统计信息

---

## 性能诊断快速参考

```sql
-- ====================
-- 1. 查看当前正在执行的SQL
-- ====================
SELECT SESS_ID, STATE, SQL_TEXT, 
    (SYSDATE - LAST_RECV_TIME) * 86400 AS ELAPSED_SEC
FROM V$SESSIONS 
WHERE STATE = 'ACTIVE' AND SQL_TEXT IS NOT NULL;

-- ====================
-- 2. 查看锁等待（阻塞关系）
-- ====================
SELECT 
    L_TID,          -- 等待者事务ID
    BLK_TID,        -- 持有者事务ID
    TABLE_ID,       -- 锁定的表
    L_TYPE          -- 锁类型
FROM V$LOCK 
WHERE BLOCKED = 1;

-- ====================
-- 3. 查看内存使用
-- ====================
SELECT 
    NAME,
    TYPE,
    TOTAL_SIZE/1024/1024 AS TOTAL_MB,
    USED_SIZE/1024/1024 AS USED_MB
FROM V$MEM_POOL;

-- ====================
-- 4. 查看BUFFER命中率
-- ====================
SELECT 
    NAME,
    N_READ,          -- 物理读
    N_HIT,           -- 命中次数
    ROUND(N_HIT * 100.0 / DECODE(N_READ + N_HIT, 0, 1, N_READ + N_HIT), 2) AS HIT_RATIO
FROM V$BUFFERPOOL;

-- ====================
-- 5. 查看TOP SQL（按逻辑读排序）
-- ====================
SELECT 
    SQL_TXT,
    EXEC_TIME,
    DISK_READS,
    BUF_READS,
    LAST_RECV_TIME
FROM V$SQL_HISTORY
ORDER BY BUF_READS DESC
LIMIT 20;
```
