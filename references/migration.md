# 数据迁移

将其他数据库（Oracle、MySQL、SQL Server 等）迁移到达梦数据库的完整指南。

---

## 通用迁移流程

```
需求确认 → 数据库调研 → 迁移评估 → 选迁移工具 → 制定计划 → 迁移准备 → 迁移实施 → 数据校验 → 统计信息 → 备份 → 应用适配
```

### 执行安全边界

迁移任务先分清三类连接：源库、目标达梦库、本地验证库。源库默认只读，目标库和本地验证库也要在执行写操作前确认库名、主机、端口、用户和用途。

| 场景 | 默认动作 | 禁止动作 |
|------|----------|----------|
| 调研源库结构/行数 | 使用只读账号或只读事务执行 `SELECT` | 未授权执行 `INSERT`、`UPDATE`、`DELETE`、`DDL`、导入任务 |
| 本地导入验证 | 只写本地达梦库或临时 schema | 把本地验证脚本连接到源库后执行写入 |
| DTS/SQLark 迁移 | 明确源端和目标端，保存连接截图/配置摘要 | 源端和目标端命名含糊、凭证复用后直接执行 |
| 应用联调 | 使用测试配置和可回滚数据 | 直接切生产写流量或扩大权限定位问题 |

执行任何可能写库的命令前，先复述目标连接，例如：`将写入 DM 10.0.0.8:5236/APP_TEST，不写 MySQL 源库 10.0.0.3:3306`。如果用户只说“本地导入”“帮我测”，按本地/测试库处理；无法确认时停下来问。

### 各阶段详细说明

| 阶段 | 任务 | 输出物 |
|------|------|--------|
| **1. 需求确认** | 明确迁移范围、停机窗口、性能要求 | 需求文档 |
| **2. 数据库调研** | 统计源库对象数量、数据量、特殊类型 | 调研报告 |
| **3. 迁移评估** | 使用 DTS/DEM/SQLark 评估兼容性 | 评估报告 |
| **4. 选迁移工具** | 根据场景选择 DTS/SQLark/DMDRS/DMDIS | 工具方案 |
| **5. 制定计划** | 排期、回退方案、人员分工 | 迁移计划 |
| **6. 迁移准备** | 环境搭建、网络调通、参数预配置 | 就绪检查单 |
| **7. 迁移实施** | 分步骤执行迁移 | 实施记录 |
| **8. 数据校验** | 比对源端/目标端对象数、行数、抽样数据 | 校验报告 |
| **9. 统计信息** | 更新统计信息，优化执行计划 | 执行确认 |
| **10. 全库备份** | 迁移成功后立即备份 | 备份文件 |
| **11. 应用适配** | 修改应用连接配置，切换至达梦 | 上线确认 |

---

## 迁移工具选择

| 工具 | 适用场景 | 关键特点 | 许可 |
|------|----------|----------|------|
| **DTS** | 静态全量迁移 | 图形化操作，支持多种异构数据库 | 免费，需停机窗口 |
| **SQLark** | 评估 + 迁移 + 校验 | 断点续迁，专为 Oracle→DM 优化 | 随数据库附带 |
| **DMDRS** | 实时增量同步 | 秒级延迟，支持双向同步 | 单独授权 |
| **DMDIS** | 数据集成 / ETL | 支持数据清洗、转换、定时调度 | 单独授权 |

### 工具选择决策树

```
需要零停机？ ──是──→ DMDRS（实时同步）
   │
   否
   │
   源库是Oracle？ ──是──→ SQLark（评估+断点续迁）
   │
   否
   │
   需要数据清洗？ ──是──→ DMDIS（ETL）
   │
   否
   │
   → DTS（标准全量迁移）
```

---

## Oracle → DM

### 兼容性参数配置

```sql
-- ==========================================
-- COMPATIBLE_MODE 兼容模式（静态参数，需重启）
--   0 = 不兼容（默认）
--   2 = Oracle 兼容
-- ==========================================
SP_SET_PARA_VALUE(2, 'COMPATIBLE_MODE', 2);

-- 字符串末尾空格处理
--   0 = 不填充  1 = 填充（兼容 Oracle）
SP_SET_PARA_VALUE(2, 'BLANK_PAD_MODE', 1);

-- 大小写敏感
--   需要与 Oracle 行为一致时设为 Y
ALTER SYSTEM SET CASE_SENSITIVE = 'Y';

-- FLOAT 类型兼容（需与 COMPATIBLE_MODE=2 同时开启）
--   将 FLOAT 当作 Oracle 的 NUMBER 处理
SP_SET_PARA_VALUE(2, 'FLOAT_MODE', 1);
```

### 关键差异对照表

| Oracle 类型/特性 | DM 对应 | 注意事项 |
|-----------------|---------|----------|
| `DATE` | `TIMESTAMP` | Oracle DATE 含时分秒，DM DATE 仅日期，迁移时自动转换 |
| `VARCHAR2(4000)` | `VARCHAR(3900)` | 8KB 页大小下，DM VARCHAR 最大 3900 字节（扣除行头开销） |
| `NUMBER(m,n)` | `NUMBER(m,n)` | **DM 不允许 n > m**，Oracle 允许且自动扩展 |
| `RAW` | `VARBINARY` | 二进制数据 |
| `LONG` | `CLOB` / `TEXT` | Oracle LONG 已废弃，迁移时转为 CLOB |
| `LONG RAW` | `BLOB` | 二进制大对象 |
| `ROWID` | `ROWID` | 格式不同，不建议应用直接依赖 ROWID 值 |
| `SYSDATE` | `SYSDATE` | Oracle 返回 DATE，DM 返回 TIMESTAMP |
| `DUAL` | `DUAL` | DM 也支持 DUAL 表 |
| `ROWNUM` | `ROWNUM` | DM 支持，但行为略有差异（子查询中） |
| `.dmp` 导出文件 | ❌ 不兼容 | Oracle dmp 不可直接导入 DM，必须使用 DTS 工具 |
| `BM$` 开头表 | 位图索引辅助表 | 迁移时自动处理，勿手动操作 |
| `CTI$` 开头表 | 全文索引辅助表 | 迁移时自动处理，勿手动操作 |

### 迁移步骤（推荐顺序）

```
第1步：迁移序列（SEQUENCE）
   ↓
第2步：迁移表结构（DDL，不含约束和索引）
   ↓
第3步：迁移数据（分表进行）
   ↓
第4步：创建约束（主键、外键、唯一、检查）
   ↓
第5步：创建索引
   ↓
第6步：迁移视图
   ↓
第7步：迁移存储过程 / 函数 / 包
   ↓
第8步：迁移触发器
   ↓
第9步：编译校验所有对象
```

**为什么按这个顺序：**

- 先迁表结构再迁数据，才能写入数据
- 先迁数据再建约束/索引：避免数据导入过程中约束检查和索引维护的开销，大幅提升迁移速度
- 先迁序列再迁表结构：若表结构依赖序列（自增 ID），序列必须已存在
- 触发器最后：触发器依赖表、过程、函数，放在最后避免依赖缺失

### 迁移实战技巧

#### 1. 大表单独迁移

```sql
-- 识别大表（超过 100 万行）
SELECT owner, table_name, num_rows
FROM dba_tables
WHERE owner = 'YOUR_SCHEMA' AND num_rows > 1000000
ORDER BY num_rows DESC;
```

大表迁移时在 DTS 中可：
- 使用并行通道（多线程）
- 分段导出（按 ROWID 范围或主键分段）

#### 2. 大字段多的表调小批量行数

在 DTS 设置中，含 CLOB/BLOB 的表的 `批量提交行数` 建议调小至 500-1000：

```
含大字段表：批量行数 = 500
普通表：批量行数 = 5000-10000
```

#### 3. 分三步迁移（避免引用约束违反）

```
第一步：仅迁移表定义（CREATE TABLE，不含约束和索引）
第二步：迁移数据（INSERT）
第三步：迁移约束和索引（ALTER TABLE ADD CONSTRAINT / CREATE INDEX）
```

理由：如果同时迁移表定义+约束+数据，可能因外键引用的表尚未导入数据而报错。

### 对象统计 SQL（源库 Oracle）

迁移前统计源库对象，用于迁移后校验：

```sql
-- Oracle 源库对象统计
SELECT
    a.username 用户,
    (SELECT COUNT(1) FROM dba_tables b WHERE b.owner = a.username)       表数量,
    (SELECT COUNT(1) FROM dba_views g WHERE g.OWNER = a.username)        视图数量,
    (SELECT COUNT(1) FROM dba_triggers h WHERE h.owner = a.username)     触发器数量,
    (SELECT COUNT(DISTINCT i.NAME) FROM dba_source i
     WHERE i.OWNER = a.username AND i.TYPE = 'FUNCTION')                  函数数量,
    (SELECT COUNT(DISTINCT i.NAME) FROM dba_source i
     WHERE i.OWNER = a.username AND i.TYPE = 'PROCEDURE')                存储过程数量,
    (SELECT COUNT(DISTINCT i.NAME) FROM dba_source i
     WHERE i.OWNER = a.username AND i.TYPE = 'PACKAGE')                  包数量,
    (SELECT COUNT(1) FROM dba_sequences j
     WHERE j.sequence_owner = a.username)                                 序列数量,
    (SELECT COUNT(1) FROM dba_mviews k WHERE k.owner = a.username)       物化视图数量,
    (SELECT COUNT(1) FROM dba_indexes idx
     WHERE idx.owner = a.username AND idx.index_type <> 'LOB')           索引数量
FROM dba_users a
WHERE username IN ('YOUR_SCHEMA');
```

### DTS 连接 Oracle 12c+ 注意事项

Oracle 12c 及以上版本使用 PDB（可插拔数据库）架构时，普通 JDBC 连接串可能失败。需使用自定义 URL：

```
jdbc:oracle:thin:@(DESCRIPTION=(ADDRESS_LIST=(ADDRESS=(PROTOCOL=TCP)(HOST=192.0.2.100)(PORT=1521)))(CONNECT_DATA=(SERVICE_NAME=orclpdb)))
```

在 DTS 中，数据源类型选择 Oracle，手动填入上述 URL。

---

## MySQL → DM

### 兼容性参数配置

> `COMPATIBLE_MODE` 是数据库实例参数，和 JDBC URL 中的 `compatibleMode=mysql` 不是同一个配置层。实例参数通常需要重启后生效，变更前先确认现网业务是否依赖原有 Oracle/标准 SQL 行为。

```sql
-- ==========================================
-- COMPATIBLE_MODE=4：兼容 MySQL 模式
-- ==========================================
SP_SET_PARA_VALUE(2, 'COMPATIBLE_MODE', 4);

-- NULL 值排序行为
--   0 = NULL 最大（默认）
--   2 = NULL 最小（兼容 MySQL）
SP_SET_PARA_VALUE(2, 'ORDER_BY_NULLS_FLAG', 2);

-- 大小写不敏感（兼容 MySQL）
ALTER SYSTEM SET CASE_SENSITIVE = '0';

-- 字符串不填充尾部空格（兼容 MySQL）
SP_SET_PARA_VALUE(2, 'BLANK_PAD_MODE', 0);

-- 数据超长时报错（兼容 MySQL 严格模式）
SP_SET_PARA_VALUE(2, 'MY_STRICT_TABLES', 1);

-- MD5 返回 VARCHAR 而非 VARBINARY（静态参数，需重启）
SP_SET_PARA_VALUE(2, 'MD5_TYPE', 1);

-- 验证当前参数。IS_MODIFIED/IS_DYNAMIC 可辅助判断是否需要重启。
SELECT PARA_NAME, PARA_VALUE, IS_DYNAMIC, IS_MODIFIED
FROM V$DM_INI
WHERE PARA_NAME IN (
    'COMPATIBLE_MODE',
    'ORDER_BY_NULLS_FLAG',
    'CASE_SENSITIVE',
    'BLANK_PAD_MODE',
    'MY_STRICT_TABLES',
    'MD5_TYPE'
);
```

### 架构映射

```
MySQL 单实例多数据库         →    DM 单实例多模式
┌─────────────────────┐         ┌─────────────────────┐
│ MySQL Instance       │         │ DM Instance         │
│  ├── database_1      │   →    │  ├── 用户1 + 表空间1  │
│  ├── database_2      │   →    │  ├── 用户2 + 表空间2  │
│  └── database_3      │   →    │  └── 用户3 + 表空间3  │
└─────────────────────┘         └─────────────────────┘
```

**映射规则：** MySQL 每个数据库 → DM 一个用户（模式）+ 专属表空间

```sql
-- 为每个 MySQL 库创建 DM 用户和表空间
CREATE TABLESPACE tbs_db1 DATAFILE '/dm8/data/tbs_db1.dbf' SIZE 128;
CREATE USER db1_user IDENTIFIED BY "password" DEFAULT TABLESPACE tbs_db1;

-- 生产环境避免直接 GRANT DBA；按应用需要授予最小权限。
GRANT CREATE TABLE, CREATE VIEW, CREATE SEQUENCE, CREATE PROCEDURE TO db1_user;
GRANT SELECT, INSERT, UPDATE, DELETE ON schema_name.table_name TO db1_user;
```

### 核心注意事项

#### 1. 字符 vs 字节（最重要！）

MySQL 的 `VARCHAR(N)` 以**字符**为单位，DM 以**字节**为单位。

| 场景 | MySQL | DM |
|------|-------|-----|
| `VARCHAR(100)` 存中文 | 最多 100 个汉字 | UTF-8 下最多 33 个汉字（100÷3≈33） |
| 需要存 100 个汉字 | `VARCHAR(100)` | `VARCHAR(300)` 或 `VARCHAR(100 CHAR)` |

**解决方案一：建表时使用 CHAR 单位**

```sql
-- 在 DM 中建表时声明字符长度
CREATE TABLE t (
    name VARCHAR(100 CHAR)   -- 以字符计，等同于 MySQL VARCHAR(100)
);
```

**解决方案二：DTS 迁移时设置数据类型映射**

在 DTS 工具中配置数据类型映射规则：

```
源类型：VARCHAR     →     目标类型：VARCHAR
                           选项：强制为字符存储 = 是
```

#### 2. CHAR 类型尾部空格

| | 行为 |
|---|------|
| MySQL CHAR | 自动去除尾部空格 |
| DM CHAR | 保留尾部空格 |

**解决方案：**
- 将 MySQL CHAR 类型改为 DM VARCHAR
- 或在应用查询中使用 `RTRIM(column)` 处理

```sql
-- 迁移时将 CHAR 改为 VARCHAR
-- 或在 DTS 中配置类型映射：CHAR → VARCHAR
```

#### 3. TIMESTAMP 默认值问题

MySQL 的 `TIMESTAMP DEFAULT '0000-00-00 00:00:00'` 在 DM 中不合法。

```sql
-- MySQL（非法值在 DM 中）
CREATE TABLE t (
    created_at TIMESTAMP DEFAULT '0000-00-00 00:00:00'
);

-- 改为 DM 兼容写法
CREATE TABLE t (
    created_at TIMESTAMP DEFAULT SYSDATE
    -- 或
    created_at TIMESTAMP DEFAULT '2000-01-01 00:00:00'
);
```

#### 4. YEAR 类型

DM 没有 `YEAR` 类型，需转换：

```sql
-- MySQL
CREATE TABLE t (birth_year YEAR);

-- DM 等价
CREATE TABLE t (birth_year CHAR(4));
-- 或
CREATE TABLE t (birth_year INT);
```

#### 5. AUTO_INCREMENT → IDENTITY

```sql
-- MySQL
CREATE TABLE t (
    id INT AUTO_INCREMENT PRIMARY KEY
) AUTO_INCREMENT = 100;

-- DM 等价
CREATE TABLE t (
    id INT IDENTITY(100, 1) PRIMARY KEY  -- (初始值, 步长)
);
```

### 对象统计 SQL（源库 MySQL）

```sql
-- MySQL 源库 - 统计各库表数量
SELECT COUNT(*) AS TABLES, TABLE_SCHEMA
FROM INFORMATION_SCHEMA.TABLES
WHERE TABLE_SCHEMA = 'your_db'
  AND TABLE_TYPE = 'BASE TABLE'
GROUP BY TABLE_SCHEMA;

-- MySQL 源库 - 统计各库视图数量
SELECT TABLE_SCHEMA, COUNT(*) AS VIEWS
FROM INFORMATION_SCHEMA.VIEWS
WHERE TABLE_SCHEMA = 'your_db'
GROUP BY TABLE_SCHEMA;

-- MySQL 源库 - 统计各表行数
SELECT TABLE_NAME, TABLE_ROWS
FROM INFORMATION_SCHEMA.TABLES
WHERE TABLE_SCHEMA = 'your_db' AND TABLE_TYPE = 'BASE TABLE'
ORDER BY TABLE_ROWS DESC;

-- MySQL 源库 - 统计存储过程和函数
SELECT ROUTINE_TYPE, COUNT(*) FROM INFORMATION_SCHEMA.ROUTINES
WHERE ROUTINE_SCHEMA = 'your_db'
GROUP BY ROUTINE_TYPE;
```

### 常用函数改写对照表

| MySQL | DM 等价写法 | 说明 |
|-------|------------|------|
| `NOW()` / `CURRENT_TIMESTAMP()` | `SYSDATE` / `CURRENT_TIMESTAMP` | 统一项目内时间来源，注意时区 |
| `IFNULL(a,b)` | `NVL(a,b)` / `COALESCE(a,b)` | `COALESCE` 更标准 |
| `DATE_FORMAT(d,'%Y-%m-%d')` | `TO_CHAR(d,'YYYY-MM-DD')` | 格式符需逐项转换 |
| `STR_TO_DATE(s,'%Y-%m-%d')` | `TO_DATE(s,'YYYY-MM-DD')` | 格式符需逐项转换 |
| `GROUP_CONCAT(x)` | `LISTAGG(x, ',') WITHIN GROUP (ORDER BY x)` | 生产建议显式排序 |
| `CONCAT(a,b,...)` | `CONCAT(a,b)` 或 `a \|\| b` | 多参数 CONCAT 需确认当前版本；不稳时改 `\|\|` |
| `FIND_IN_SET(str, list)` | `INSTR(list, str)` 或自定义函数 | 查找字符串在列表中的位置 |
| `MD5('abc')` | `TO_CHAR(MD5('abc'))` | MD5 返回 VARBINARY，需 TO_CHAR 转换 |
| `yearweek(d, 1)` | `YEAR(d) \|\| WEEK(d, 1)` | 返回年份+周数 |
| `PERIOD_DIFF(a, b)` | `MONTHS_BETWEEN(TO_DATE(a,'yyyymm'), TO_DATE(b,'yyyymm'))` | 两个时期之间的月数差 |
| `(@age:=@age+age)` | `SUM(age) OVER (ORDER BY id)` | 变量累加 → 窗口函数 |
| `ON UPDATE CURRENT_TIMESTAMP` | `CREATE OR REPLACE TRIGGER ... BEFORE UPDATE` | 自动更新修改时间 |
| `INSERT ... ON DUPLICATE KEY UPDATE` | `MERGE INTO ... WHEN MATCHED ... WHEN NOT MATCHED ...` | upsert 需确认唯一键/并发语义 |
| `LIMIT offset, size` | `LIMIT size OFFSET offset` 或 `ROW_NUMBER()` | 旧版本/复杂排序优先 `ROW_NUMBER()` |
| `true` / `false` | `1` / `0`、`BIT`、`TINYINT` 或显式条件 | MyBatis XML 中常被遗漏 |
| 反引号 `` `col` `` | 双引号 `"COL"` 或统一大写 | 推荐迁移时对象名自动转大写 |
| `AES_ENCRYPT(str, key)` | `DBMS_CRYPTO.ENCRYPT(...)` | 加密函数 |
| `AES_DECRYPT(str, key)` | `DBMS_CRYPTO.DECRYPT(...)` | 解密函数 |
| `COLLATE utf8mb4_bin` | `NLSSORT(col, 'NLS_SORT=BINARY')` + 函数索引 | 二进制排序规则 |

#### 函数改写示例

```sql
-- MySQL FIND_IN_SET → DM INSTR 改写
-- MySQL:
SELECT * FROM t WHERE FIND_IN_SET('a', tags);

-- DM:
SELECT * FROM t WHERE INSTR(',' || tags || ',', ',a,') > 0;
-- 或创建自定义函数实现

-- MySQL ON UPDATE CURRENT_TIMESTAMP → DM 触发器
-- DM:
CREATE OR REPLACE TRIGGER trg_t_update_time
BEFORE UPDATE ON t
FOR EACH ROW
BEGIN
    :NEW.updated_at := SYSDATE;
END;

-- MySQL 变量累加 → DM 窗口函数
-- MySQL:
SELECT @age := @age + age AS cumulative_age FROM t, (SELECT @age := 0) init;

-- DM:
SELECT SUM(age) OVER (ORDER BY id) AS cumulative_age FROM t;
```

### Spring Boot / MyBatis 静态扫描清单

迁移应用代码时，先在 mapper XML、注解 SQL、初始化脚本中做全量扫描，再和迁移方案逐项对齐。以下命令在项目根目录执行，路径按实际项目调整。

```bash
# 1. MySQL 分页、单行取数和无序 LIMIT 1
rg -n --glob '*.xml' --glob '*.sql' '\bLIMIT\b|limit\s+1|limit\s+\#\{|\blimit\s+\?,\s*\?' src/main/resources

# 2. MySQL 专有函数、日期格式、字符串聚合
rg -n --glob '*.xml' --glob '*.sql' 'NOW\(|CURRENT_TIMESTAMP\(\)|DATE_FORMAT\(|STR_TO_DATE\(|IFNULL\(|GROUP_CONCAT\(|FIND_IN_SET\(' src/main/resources

# 3. upsert、自增、取自增 ID
rg -n --glob '*.xml' --glob '*.java' 'ON DUPLICATE KEY UPDATE|useGeneratedKeys|LAST_INSERT_ID|AUTO_INCREMENT|IDENTITY_VAL_LOCAL' src

# 4. 布尔字面量、反引号、MySQL 系统表/变量
rg -n --glob '*.xml' --glob '*.sql' '\btrue\b|\bfalse\b|`[^`]+`|information_schema|@@|SHOW TABLE|SHOW COLUMNS' src/main/resources

# 5. 字符串拼接、dual、排序/分组隐式行为
rg -n --glob '*.xml' --glob '*.sql' 'CONCAT\(|dual|GROUP BY|ORDER BY|FOR UPDATE' src/main/resources

# 6. 多表别名、同名字段、GROUP 相关问题
rg -n --glob '*.xml' --glob '*.sql' '\b(FROM|JOIN|LEFT JOIN|RIGHT JOIN|INNER JOIN)\b|GROUP BY|HAVING|ORDER BY|GROUP_CONCAT\(|LISTAGG\(' src/main/resources

# 7. MyBatis <if> 只判 null、未判空串的候选项
rg -n --glob '*.xml' '<if\s+test="[^"]*!=\s*null' src/main/resources
```

重点判定规则：

- `LIMIT 1` 不能机械改写；若没有 `ORDER BY`，必须和业务确认“第一条”的排序依据。
- `GROUP_CONCAT` 改 `LISTAGG` 时必须补 `WITHIN GROUP (ORDER BY ...)`，否则结果顺序可能漂移。
- `GROUP BY` 中的非聚合查询列必须逐项核对；MySQL 宽松模式下可执行的隐式分组，迁移到达梦要改为标准写法。
- 多表 SQL 中不同表有相同字段时必须使用表别名限定；表一旦取别名，后续引用必须使用该别名，不能再写原表名或裸字段。
- `now()`、`SYSDATE`、应用服务器时间不要混用；审计时间、编号生成、逻辑删除时间要统一来源。
- `true` / `false` 在 XML 动态 SQL、逻辑删除、权限过滤中很容易漏扫，建议统一为数值或显式字段类型。
- `<if>` 字符串条件必须同时判断 `!= null` 和 `!= ''`；非字符串字段不要机械补空串判断。
- `useGeneratedKeys="true"` 只有在目标表确认为 `IDENTITY` 且驱动版本支持时才保留；否则改序列或雪花 ID。
- `ON DUPLICATE KEY UPDATE` 改 `MERGE INTO` 前，先确认唯一键、并发冲突和默认值触发逻辑。
- 对象名策略二选一：迁移时统一转大写，或保留大小写并在 SQL 中全量双引号；不要混用。

### MyBatis XML 逐文件比对验证

迁移 mapper XML 时不要批量改完再总体验收。必须采用“改一个、比一个、记一个”的节奏：每完成一个 `.xml` 的迁移，立即完成以下验证，再进入下一个文件。

| 验证项 | 操作 | 通过标准 |
|--------|------|----------|
| 变更范围 | 对比当前 XML 的 git diff 或备份 diff | 只改达梦迁移相关 SQL，无无关格式化/重排 |
| 残留方言扫描 | 对当前 XML 单文件运行下方推荐扫描命令 | 所有命中项已确认：已改写、可兼容或明确保留原因 |
| SQL 语义对照 | 逐条核对原 SQL 与新 SQL 的筛选条件、排序、分页、聚合、默认值 | 查询范围、排序依据、返回列、更新字段一致 |
| 表别名/同名字段 | 检查多表 SQL 的 `SELECT`、`WHERE`、`ON`、`GROUP BY`、`HAVING`、`ORDER BY` | 同名字段全部带表别名；定义别名后全程使用别名，无裸字段或原表名混用 |
| MyBatis `<if>` | 检查字符串条件是否同时判断 `!= null` 和 `!= ''` | 字符串空值不会生成错误过滤条件；非字符串字段按类型判断 |
| `GROUP` | 检查 `GROUP BY`、`GROUP_CONCAT`/`LISTAGG`、名为 `group` 的字段 | 分组字段标准、聚合拼接排序稳定、保留字已处理 |
| `LIMIT 1` | 对每个单行查询检查 `ORDER BY` | 排序依据明确；无序取第一条必须业务确认 |
| 主键/自增 | 检查 `useGeneratedKeys`、序列、雪花 ID、IDENTITY | 新增后 ID 返回方式和目标表设计一致 |
| 时间/布尔 | 检查 `now()`、`CURRENT_TIMESTAMP`、`true/false`、逻辑删除 | 时间来源统一，布尔值与字段类型一致 |
| 可执行性 | 能连达梦环境时，用关键参数执行该 XML 涉及的核心 SQL | 无语法错误，核心查询/新增/更新路径通过 |
| 验证记录 | 在迁移清单中记录文件名、检查人、时间、残留项和处理结论 | 当前 XML 有明确“通过/待确认”状态 |

推荐单文件命令：

```bash
mapper=src/main/resources/mapper/path/ExampleMapper.xml

# 查看本文件改动范围
git diff -- "$mapper"

# 扫描本文件残留 MySQL 方言和高风险点
rg -n 'LIMIT|NOW\(|IFNULL\(|DATE_FORMAT\(|STR_TO_DATE\(|GROUP_CONCAT\(|LISTAGG\(|GROUP BY|ON DUPLICATE KEY UPDATE|useGeneratedKeys|LAST_INSERT_ID|\btrue\b|\bfalse\b|`[^`]+`|<if\s+test=' "$mapper"
```

单文件验证记录模板：

| 字段 | 填写要求 |
|------|----------|
| XML 文件名 | 例如 `src/main/resources/mapper/BizMapper.xml` |
| 改写点数量 | 本文件实际修改的 SQL 点数 |
| 残留命中数量 | 单文件扫描仍命中的高风险项数量 |
| 残留项结论 | 每个命中项写明：已兼容、业务保留、待确认或需继续改 |
| 语义核对结论 | 筛选条件、排序、分页、聚合、默认值是否与原 SQL 一致 |
| 达梦执行验证 | 已执行的核心 SQL、参数样例、结果或错误信息 |
| 需业务确认项 | 无序 `LIMIT 1`、默认排序、并发 upsert、时间来源等 |
| 最终状态 | 通过 / 待业务确认 / 阻塞 |

整体迁移报告中至少保留：`XML 文件名`、`改写点数量`、`残留命中数量`、`需业务确认项`、`是否已在达梦执行验证`、`结论`。没有达梦环境时，也要完成 diff、残留扫描和语义核对，并明确“未做可执行性验证”的风险。

---

## SQL Server → DM

### 兼容性参数配置

```sql
-- COMPATIBLE_MODE=3：兼容 SQL Server
SP_SET_PARA_VALUE(2, 'COMPATIBLE_MODE', 3);

-- MS_PARSE_PERMIT：支持 MSSQL 语法和 @变量赋值
--   0 = 不解析
--   1 = 仅解析 MSSQL 语法
--   2 = 解析 MSSQL 语法 + 支持 @变量赋值
SP_SET_PARA_VALUE(2, 'MS_PARSE_PERMIT', 2);

-- 大小写不敏感
ALTER SYSTEM SET CASE_SENSITIVE = '0';
```

### 关键差异对照表

| SQL Server | DM | 说明 |
|------------|-----|------|
| `VARCHAR(MAX)` | `VARCHAR(8188)` 或 `CLOB` | 超大文本用 CLOB |
| `NVARCHAR` | `VARCHAR` + `CHARSET=UTF8` | Unicode 文本 |
| `DATETIME` | `TIMESTAMP` | 日期时间类型 |
| `SMALLDATETIME` | `TIMESTAMP` | 精度降低 |
| `MONEY` | `NUMBER(19,4)` | 货币类型 |
| `BIT` | `TINYINT` 或 `CHAR(1)` | 布尔值 |
| `UNIQUEIDENTIFIER` | `VARCHAR(36)` 或 `CHAR(36)` | UUID |
| `IMAGE` | `BLOB` | 二进制大对象 |
| `TEXT` | `CLOB` | 文本大对象 |
| `IDENTITY(1,1)` | `IDENTITY(1,1)` | 自增列，语法相同 |

### SQL 语法改写

| SQL Server 写法 | DM 等价写法 | 说明 |
|-----------------|------------|------|
| `CONVERT(type, expr, style)` | `TO_CHAR(CONVERT(type, expr), 'yyyy-mm-dd hh24:MI:ss')` | 日期格式化 |
| `WITH (NOLOCK)` | `WITH UR` | 读未提交隔离级别 |
| `FOR XML PATH('')` | `LISTAGG(col, ',')` 或 `XMLAGG(...)` | 行转列拼接 |
| `@@ROWCOUNT` | `SQL%ROWCOUNT`（存储过程中） | 影响行数 |
| `@@IDENTITY` | `IDENTITY_VAL_LOCAL()` | 最后插入的自增值 |
| `INSERTED` / `DELETED` | `NEW` / `OLD`（触发器中） | 触发器虚拟表 |
| `@variable` | 去掉 `@`，声明用 `DECLARE` | 变量语法 |
| `SET @var = value` | `var := value` | 变量赋值用 `:=` |
| `NEXT VALUE FOR seq` | `sp_set_session_parse_type('TSQL')` 后用原语法 | 序列取值 |
| `TOP n` | `LIMIT n` 或 `WHERE ROWNUM <= n` | 限制行数 |
| `ISNULL(a, b)` | `NVL(a, b)` | 空值替换 |
| `GETDATE()` | `SYSDATE` | 当前时间 |

#### 语法改写示例

```sql
-- SQL Server CONVERT → DM
-- SQL Server:
SELECT CONVERT(VARCHAR, GETDATE(), 120);
-- DM:
SELECT TO_CHAR(SYSDATE, 'yyyy-mm-dd HH24:MI:SS');

-- SQL Server FOR XML PATH → DM
-- SQL Server:
SELECT STUFF((SELECT ',' + name FROM t FOR XML PATH('')), 1, 1, '');
-- DM:
SELECT LISTAGG(name, ',') WITHIN GROUP (ORDER BY name) FROM t;

-- SQL Server 触发器 → DM
-- SQL Server:
CREATE TRIGGER trg_audit ON t AFTER INSERT AS
BEGIN
    INSERT INTO audit_log SELECT * FROM INSERTED;
END;

-- DM:
CREATE OR REPLACE TRIGGER trg_audit
AFTER INSERT ON t
FOR EACH ROW
BEGIN
    INSERT INTO audit_log VALUES (:NEW.id, :NEW.name, SYSDATE);
END;

-- SQL Server 变量 → DM
-- SQL Server:
DECLARE @myvar INT = 100;
SET @myvar = @myvar + 1;
-- DM:
DECLARE
    myvar INT := 100;
BEGIN
    myvar := myvar + 1;
END;
```

### SQL Server 特有注意事项

1. **同一列多个单列索引：** SQL Server 允许同一列上创建多个单列索引，DM 不允许。迁移时报错可忽略，手动保留其中一个即可。

2. **自增列已有数据：** 如果目标表已有数据且包含自增列的值，需设置：

```sql
SET IDENTITY_INSERT ON;  -- 允许向自增列显式插入值
-- ... 插入数据 ...
SET IDENTITY_INSERT OFF;
```

3. **计算列（Computed Column）：** DM 支持计算列，语法类似：

```sql
-- SQL Server:
CREATE TABLE t (a INT, b AS a * 2);
-- DM:
CREATE TABLE t (a INT, b GENERATED ALWAYS AS (a * 2));
```

---

## 迁移后必做

无论从哪种数据库迁移，完成数据导入后必须执行以下步骤：

如果用户提供迁移后问题清单、缺陷台账或上线回归记录，先按 `post-migration-triage.md` 做分类研判：区分达梦兼容系统性风险、数据规则问题、应用 I/O 问题、前端状态问题和普通业务 bug，再决定哪些规则需要回写到迁移扫描清单。

### 1. 更新统计信息

```sql
-- 收集整个模式的统计信息（推荐）
-- 参数：模式名, 采样百分比, 是否级联, 选项
DBMS_STATS.GATHER_SCHEMA_STATS(
    'YOUR_SCHEMA',                    -- 模式名
    100,                               -- 采样 100%（全量）
    FALSE,                             -- 不级联（或 TRUE 级联索引）
    'FOR ALL COLUMNS SIZE AUTO'        -- 自动直方图
);

-- 收集单表统计信息
DBMS_STATS.GATHER_TABLE_STATS(
    'YOUR_SCHEMA',
    'YOUR_TABLE',
    100,
    FALSE,
    'FOR ALL COLUMNS SIZE AUTO'
);

-- 收集数据库级统计信息（DBA 权限）
DBMS_STATS.GATHER_DATABASE_STATS(100, FALSE, 'FOR ALL COLUMNS SIZE AUTO');
```

**为什么要更新统计信息：**
- 迁移后表的行数、数据分布已变化
- 旧的统计信息（如有）会导致执行计划不优
- 必须更新后才能获得最优的 SQL 执行效率

### 2. 全库备份

```sql
-- 在线全库备份
BACKUP DATABASE FULL BACKUPSET '/dm8/backup/full_after_migration';

-- 查看备份信息
SELECT * FROM V$BACKUPSET;
```

```bash
# 使用 DMRMAN 工具备份
dmrman CTLSTMT="BACKUP DATABASE '/dm8/data/DAMENG/dm.ini' FULL BACKUPSET '/dm8/backup/full_after_migration'"
```

### 3. 数据校验

```sql
-- ==========================================
-- 对比对象数量（在 DM 中执行）
-- ==========================================

-- 统计各类型对象数量
SELECT 'TABLE' AS 对象类型, COUNT(*) AS 数量 FROM DBA_TABLES WHERE OWNER = 'YOUR_SCHEMA'
UNION ALL
SELECT 'VIEW', COUNT(*) FROM DBA_VIEWS WHERE OWNER = 'YOUR_SCHEMA'
UNION ALL
SELECT 'INDEX', COUNT(*) FROM DBA_INDEXES WHERE OWNER = 'YOUR_SCHEMA'
UNION ALL
SELECT 'TRIGGER', COUNT(*) FROM DBA_TRIGGERS WHERE OWNER = 'YOUR_SCHEMA'
UNION ALL
SELECT 'FUNCTION', COUNT(DISTINCT NAME) FROM DBA_SOURCE WHERE OWNER = 'YOUR_SCHEMA' AND TYPE = 'FUNCTION'
UNION ALL
SELECT 'PROCEDURE', COUNT(DISTINCT NAME) FROM DBA_SOURCE WHERE OWNER = 'YOUR_SCHEMA' AND TYPE = 'PROCEDURE'
UNION ALL
SELECT 'SEQUENCE', COUNT(*) FROM DBA_SEQUENCES WHERE SEQUENCE_OWNER = 'YOUR_SCHEMA'
UNION ALL
SELECT 'MVIEW', COUNT(*) FROM DBA_MVIEWS WHERE OWNER = 'YOUR_SCHEMA';

-- ==========================================
-- 对比数据行数（抽样）
-- ==========================================
SELECT COUNT(*) AS ROW_COUNT FROM YOUR_SCHEMA.YOUR_TABLE;

-- ==========================================
-- 检查无效对象
-- ==========================================
SELECT OWNER, OBJECT_NAME, OBJECT_TYPE, STATUS
FROM DBA_OBJECTS
WHERE OWNER = 'YOUR_SCHEMA' AND STATUS = 'INVALID'
ORDER BY OBJECT_TYPE, OBJECT_NAME;

-- 重新编译无效对象
CALL DBMS_UTILITY.COMPILE_SCHEMA('YOUR_SCHEMA');
```

### 4. 开启 SQL 日志排查慢 SQL

```sql
-- 开启 SQL 日志（记录执行时间超过 200ms 的 SQL）
SP_SET_PARA_VALUE(1, 'SVR_LOG', 1);                        -- 开启日志
SP_SET_PARA_VALUE(1, 'SQL_TRACE_MASK', '1:3:23:25');       -- 跟踪类型
SP_SET_PARA_VALUE(1, 'SVR_LOG_MIN_EXEC_TIME', 200);        -- 最小执行时间(ms)
SP_SET_PARA_VALUE(1, 'SVR_LOG_FILE_NUM', 10);              -- 日志文件个数

-- 日志文件位置：$DM_HOME/log/
-- 文件命名：dmsql_实例名_日期_时间.log

-- 分析完记得关闭（生产环境长期开启影响性能）
-- SP_SET_PARA_VALUE(1, 'SVR_LOG', 0);
```

### 5. 应用切换检查清单

| 检查项 | 说明 | 状态 |
|--------|------|------|
| 数据库连接串 | 修改 IP/端口/用户名/密码 | ☐ |
| 驱动兼容性 | 确认驱动版本与服务器一致 | ☐ |
| 实例兼容参数 | `V$DM_INI` 确认 `COMPATIBLE_MODE`、大小写、空值排序等 | ☐ |
| SQL 语法适配 | 排查存储过程、函数中的特殊语法 | ☐ |
| 函数差异 | 检查 DATE_FORMAT、MD5、AES 等函数 | ☐ |
| MyBatis XML | 分页、upsert、布尔字面量、`useGeneratedKeys`、关键字、反引号已扫描 | ☐ |
| 逐 XML 比对 | 每完成一个 mapper XML 都已完成单文件 diff、残留扫描、语义核对和验证记录 | ☐ |
| 核心业务回归 | 编号生成、逻辑删除、审核时间、列表分页、导入导出、权限过滤 | ☐ |
| 事务隔离级别 | 确认隔离级别设置正确 | ☐ |
| 字符集 | 确认应用与数据库字符集一致 | ☐ |
| 连接池配置 | 调整连接池参数适配 DM | ☐ |
| 监控告警 | 配置 DM 监控和告警 | ☐ |
| 灰度切流 | 先切只读流量，再切写入流量 | ☐ |
| 回退方案 | 准备好切回源库的连接配置 | ☐ |

### 6. MySQL 迁移专项验收用例

| 场景 | 验收方式 | 通过标准 |
|------|----------|----------|
| 字符长度 | 插入中文、emoji、长文本样本 | 无截断、无乱码、字段长度符合预期 |
| 分页与排序 | 对每个列表接口查第一页/中间页/末页 | 页数、顺序、总数与源库一致 |
| `LIMIT 1` | 覆盖所有单行取数 SQL | 排序依据明确，结果稳定 |
| 自增/主键 | 新增记录并读取返回 ID | ID 不冲突，MyBatis 返回值正确 |
| upsert | 分别验证插入路径和更新路径 | `MERGE` 两条路径都命中预期 |
| 时间函数 | 新增、修改、审核、逻辑删除 | 时间来源和时区一致 |
| 布尔/逻辑删除 | 查询已删/未删、启用/禁用数据 | 条件过滤正确 |
| 聚合拼接 | 验证 `GROUP_CONCAT` 改写结果 | 内容和顺序都一致 |
| 权限/租户过滤 | 多租户、角色权限、数据范围 | 无越权、无漏数据 |
| 慢 SQL | 打开慢 SQL 日志或 APM | 核心接口耗时不劣化或有优化方案 |

---

## 常见迁移问题速查

| 问题 | 原因 | 解决方案 |
|------|------|----------|
| 迁移后中文乱码 | 字符集不匹配 | 确认 DM 初始化字符集为 UTF-8 |
| Oracle NUMBER 迁移报错 | NUMBER(n,m) 中 n < m | 改为 NUMBER(m+5, m) 或适当扩大精度 |
| MySQL TIMESTAMP 报错 | 默认值 '0000-00-00' 非法 | 改为 DEFAULT SYSDATE |
| 外键约束创建失败 | 被引用的表数据尚未导入 | 先导数据，后建约束 |
| 索引创建慢 | 数据量大 + 并行度不够 | 增加并行度参数 |
| VARCHAR 插入数据超长 | MySQL VARCHAR 按字符，DM 按字节 | 使用 VARCHAR(N CHAR) |
| 存储过程编译失败 | 语法不兼容 | 逐个排查，使用兼容模式 |
| DTS 连接 Oracle 12c 失败 | PDB 架构 | 使用自定义 JDBC URL（含 SERVICE_NAME） |
| 自增列值冲突 | 源库自增值与目标库现有数据冲突 | 调整 IDENTITY 初始值 |
| 迁移速度慢 | 批量行数过大或过小 | 普通表 5000-10000，含大字段表 500-1000 |

---

## 补充：基于 SQLark 的数据迁移

SQLark 提供了更友好的跨库迁移界面，适合非 DBA 角色。

### SQLark 迁移步骤

1. 配置源库、目标库连接
2. 选对象（可按表/视图/存储过程过滤）
3. 类型映射预览与调整
4. 启动迁移（支持断点续传）
5. 数据校验（行数 + 抽样比对）

### SQLark 适用场景

| 场景 | 推荐度 | 说明 |
|------|--------|------|
| Oracle → DM 一次性全量迁移 | ⭐⭐⭐⭐⭐ | SQLark 对 Oracle 兼容最完整，自带语法改写建议 |
| 中小数据量（< 500GB） | ⭐⭐⭐⭐⭐ | 断点续传友好，失败后可从断点继续 |
| 紧急窗口期（数小时） | ⭐⭐⭐⭐ | 评估 + 迁移 + 校验一条龙 |
| 实时增量同步 | ⭐⭐ | 需配合 DMDRS 使用 |

### SQLark vs DTS 对比

| 维度 | SQLark | DTS |
|------|--------|-----|
| 上手难度 | 低（向导式） | 中（需配置） |
| 评估功能 | 强（自动生成报告） | 弱 |
| 断点续传 | ✅ | ❌ |
| 异构类型映射 | 自动 + 可视化调整 | 自动 + 规则配置 |
| 数据校验 | 内置（行数 + 抽样 + 字段对比） | 需手动写 SQL |
| 适用数据库 | Oracle/MySQL/SQL Server/PG/DB2 | 主流数据库均支持 |
| 许可 | 随 DM 数据库附带 | 免费 |

### 注意事项

- SQLark 迁移时建议先运行「评估」模式，得到兼容报告后再实施
- 评估报告会列出所有不兼容对象（存储过程、函数、触发器）的具体位置
- 断点续传仅对数据迁移有效，DDL 阶段失败需重新执行
- 评估报告中的「建议改写」部分可导出为 SQL 脚本，人工审核后批量应用
- 迁移时建议勾选「生成回滚脚本」选项，便于失败时回退

### SQLark 评估报告关键指标

迁移前必看的几个数字：

- 兼容率：> 95% 算优秀，< 80% 需重点关注
- 不兼容对象数：按类型分组（存储过程、函数、触发器、视图）
- 大对象（CLOB/BLOB）表数量：决定批量行数设置
- 预计迁移时长：基于历史数据采样估算

如果兼容率 < 80%，强烈建议先在 PoC 阶段解决大部分改写问题，再进入正式迁移。

## 补充：PostgreSQL → DM 迁移

### 关键差异

- 序列：PG 用 `SERIAL`/独立 SEQUENCE，DM 用 `IDENTITY` 或 SEQUENCE
- 数组类型：PG 特有 `ARRAY<T>`，DM 用 `CLOB` 存 JSON
- JSONB → DM `JSON` 类型
- 函数风格：PG `function_name($1, $2)` → DM `:1, :2` 绑定变量

### 类型映射对照

| PostgreSQL | DM | 说明 |
|-----------|-----|------|
| `SERIAL` | `IDENTITY(1,1)` | 自增列 |
| `BIGSERIAL` | `IDENTITY(1,1)` + `BIGINT` | 大整数自增 |
| `TEXT` | `TEXT` 或 `CLOB` | 文本大对象 |
| `BYTEA` | `BLOB` | 二进制大对象 |
| `BOOLEAN` | `TINYINT` 或 `BIT` | 布尔值 |
| `UUID` | `CHAR(36)` 或 `VARCHAR(36)` | UUID |
| `TIMESTAMPTZ` | `TIMESTAMP WITH TIME ZONE` | 带时区时间 |
| `INTERVAL` | `INTERVAL` | 兼容但精度不同 |
| `JSON` / `JSONB` | `JSON` | JSON 数据 |
| `ARRAY<T>` | `CLOB` (JSON 序列化) | 数组用 JSON 字符串存 |

### 工具

- DTS 支持 PG → DM
- SQLark 同样支持
- 异构 SQL 改写：CTE、窗口函数、JSON 操作需要逐条 review

### 函数改写示例

```sql
-- PG 数组查询 → DM JSON 查询
-- PG:
SELECT * FROM t WHERE tags @> ARRAY['java', 'mysql'];
-- DM（tags 用 JSON 存）:
SELECT * FROM t WHERE JSON_EXISTS(tags, '$.[*] ? (@ == "java")')
  AND JSON_EXISTS(tags, '$.[*] ? (@ == "mysql")');

-- PG JSONB → DM JSON
-- PG:
SELECT data->>'name' FROM t WHERE data @> '{"age": 30}';
-- DM:
SELECT JSON_VALUE(data, '$.name') FROM t WHERE JSON_VALUE(data, '$.age') = '30';
```

### 常见踩坑

1. **PG 的 `ILIKE`（大小写不敏感模糊匹配）**：DM 默认大小写敏感，迁移后查询结果会不同。解决方案：用 `LOWER(col) LIKE LOWER(pattern)` 替代
2. **PG 的 `ILIKE '%xxx%'` 走索引**：DM 不会。需要建函数索引：`CREATE INDEX idx_t_name ON t(LOWER(name))`
3. **PG 的 `SERIAL` 列已经手工插入值**：DM 迁移后自增列需调整 IDENTITY 起始值，否则会冲突
4. **PG 的 `WITH RECURSIVE` 递归 CTE**：DM 支持但语法略有差异，`SEARCH`/`CYCLE` 子句需改写
5. **PG 的 `LISTEN`/`NOTIFY` 异步通知**：DM 无此功能，需改用轮询或消息队列
6. **PG 的 `hstore` / `ltree` 扩展类型**：迁移到 DM 后需用 JSON 替代，并重写所有相关 SQL

### 性能调优要点

迁移到 DM 后常遇见的性能问题及调优方法：

- PG 的 `EXPLAIN ANALYZE` → DM 的 `EXPLAIN`（不带 ANALYZE 关键字）
- PG 的统计信息更新 `ANALYZE table_name` → DM 的 `DBMS_STATS.GATHER_TABLE_STATS`
- PG 的 `VACUUM` 自动清理 → DM 的 `SP_TABLE_CHECK` + 定期重组表
- PG 的 `LISTEN/NOTIFY` 性能优势 → DM 需要靠应用层轮询补偿
- PG 的 `COPY FROM` 高速导入 → DM 用 `dmfldr` 工具或 `INSERT /*+ APPEND */` 提示

## 补充：SQL Server → DM 迁移

### 关键差异

- 自增列：SQL Server `IDENTITY(1,1)` → DM 同样 `IDENTITY(1,1)`，**完全兼容**
- T-SQL 语法：`TOP N`/`GETDATE()`/`ISNULL()` 部分兼容，部分需改写
- 临时表：`#temp` → DM `GLOBAL TEMPORARY TABLE` 或会话临时表
- 存储过程：T-SQL → DM PL/SQL 块需重写
- 触发器：`SET NOCOUNT ON` 等 T-SQL 特性需去除

### 临时表迁移方案

```sql
-- SQL Server 会话级临时表
CREATE TABLE #temp_orders (id INT, amount DECIMAL(18,2));

-- DM 替代方案 1：会话级 ON COMMIT ROWS（推荐）
CREATE GLOBAL TEMPORARY TABLE temp_orders (
    id INT,
    amount DECIMAL(18,2)
) ON COMMIT PRESERVE ROWS;

-- DM 替代方案 2：会话变量表
DECLARE
    TYPE order_rec IS RECORD (id INT, amount DECIMAL(18,2));
    TYPE order_tab IS TABLE OF order_rec INDEX BY PLS_INTEGER;
    v_orders order_tab;
BEGIN
    v_orders(1).id := 100;
    v_orders(1).amount := 99.5;
END;
```

### 存储过程改写要点

- 去掉 `SET NOCOUNT ON`、`SET XACT_ABORT ON`
- `@variable` → 去掉 `@`，用 `DECLARE` 声明
- `SELECT @var = col` → `SELECT col INTO var FROM ...`
- `IF EXISTS (SELECT ...)` 写法保持兼容
- `TRY...CATCH` → DM 用 `EXCEPTION` 块
- 游标 `DECLARE c CURSOR FOR` 语法兼容

### 函数差异对照

| SQL Server | DM | 说明 |
|-----------|-----|------|
| `LEN(str)` | `LENGTH(str)` 或 `CHAR_LENGTH(str)` | 字符串长度 |
| `CHARINDEX(sub, str)` | `INSTR(str, sub)` | 子串位置 |
| `SUBSTRING(str, start, len)` | `SUBSTR(str, start, len)` | 子串 |
| `GETDATE()` | `SYSDATE` | 当前时间 |
| `DATEDIFF(day, a, b)` | `DATEDIFF(day, a, b)` | 日期差，语法兼容 |
| `DATEADD(day, 1, d)` | `DATEADD(day, 1, d)` | 日期加，语法兼容 |
| `CAST(x AS INT)` | `CAST(x AS INT)` | 类型转换，兼容 |
| `STUFF(str, start, len, new)` | `REPLACE_PART(str, start, len, new)` | 字符串替换 |
| `ISNULL(a, b)` | `NVL(a, b)` | 空值替换 |
| `NEWID()` | `SYS_GUID()` | 生成 UUID |

### 触发器改写示例

```sql
-- SQL Server
CREATE TRIGGER trg_audit ON orders AFTER INSERT AS
BEGIN
    SET NOCOUNT ON;
    INSERT INTO audit_log(order_id, action, created_at)
    SELECT order_id, 'INSERT', GETDATE() FROM INSERTED;
END;

-- DM（去掉 SET NOCOUNT ON，GETDATE() → SYSDATE，INSERTED → :NEW）
CREATE OR REPLACE TRIGGER trg_audit
AFTER INSERT ON orders
FOR EACH ROW
BEGIN
    INSERT INTO audit_log(order_id, action, created_at)
    VALUES (:NEW.order_id, 'INSERT', SYSDATE);
END;
```

## 补充：DB2 → DM 迁移

### 关键差异

- DB2 的 `FETCH FIRST n ROWS ONLY` → DM `LIMIT n`
- DB2 的 `SELECT ... FROM TABLE(VALUES (...))` 表函数 → DM 用 CTE 替代
- DB2 `GENERATED ALWAYS AS IDENTITY` → DM `IDENTITY(1,1)`
- 字符编码：DB2 EBCDIC 需要先转 ASCII/UTF-8

### 语法改写对照

| DB2 | DM | 说明 |
|-----|-----|------|
| `FETCH FIRST 10 ROWS ONLY` | `LIMIT 10` | 取前 N 行 |
| `SELECT * FROM TABLE(VALUES (1,'a'),(2,'b')) AS t(c1,c2)` | `WITH t(c1,c2) AS (SELECT 1,'a' UNION ALL SELECT 2,'b') SELECT * FROM t` | 表函数 → CTE |
| `GENERATED ALWAYS AS IDENTITY` | `IDENTITY(1,1)` | 自增列 |
| `GENERATED ALWAYS AS (col1 * 2)` | `GENERATED ALWAYS AS (col1 * 2)` | 计算列，语法兼容 |
| `VARCHAR(n) FOR BIT DATA` | `VARBINARY(n)` | 二进制字符串 |
| `CLOB` / `BLOB` | `CLOB` / `BLOB` | 大对象类型，兼容 |
| `DECFLOAT` | `NUMBER` / `DECIMAL` | 高精度小数 |
| `CURRENT TIMESTAMP` | `SYSDATE` / `CURRENT_TIMESTAMP` | 当前时间 |
| `COALESCE(a, b, c)` | `COALESCE(a, b, c)` | 空值替换，兼容 |
| `XMLPARSE(DOCUMENT ...)` | `XMLPARSE(DOCUMENT ...)` | XML 解析，兼容 |

### 字符编码迁移

DB2 历史系统常用 EBCDIC 编码（如 CCSID 1140 中文），迁移前必须先转码：

```bash
# 导出为 ASCII/UTF-8
db2 "EXPORT TO orders.del OF DEL MODIFIED BY CODEPAGE=1208 SELECT * FROM orders"

# 或使用 iconv 转码
iconv -f EBCDIC-CN -t UTF-8 orders.ebc -o orders.utf8

# 再用 DTS 导入 DM
```

### DB2 → DM 特殊场景

1. **DB2 的 `SELECT FROM SYSIBM.SYSDUMMY1`**：DM 用 `DUAL` 替代
2. **DB2 的 `CURRENT DATE` / `CURRENT TIME`**：DM 用 `CURRENT DATE` / `CURRENT TIME`（兼容）
3. **DB2 的 `VALUES` 独立语句**：DM 也支持 `VALUES (...), (...)` 但需加 `UNION ALL` 多行
4. **DB2 的 `XMLCAST` / `XMLQUERY`**：DM XML 函数命名空间略有不同，需逐个映射
5. **DB2 的 `MQSEND` / `MQRECEIVE`（MQ 集成）**：DM 通过 DBMS_MQUEUE 包实现，需重写调用方式
6. **DB2 的 `LOCK TABLE ... IN EXCLUSIVE MODE`**：DM 用 `LOCK TABLE ... IN EXCLUSIVE MODE` 兼容
7. **DB2 的 `CREATE ALIAS`**：DM 用 `CREATE SYNONYM` 替代，语法不同

### DB2 兼容性参数

DM 提供部分 DB2 兼容支持（COMPATIBLE_MODE=8 模式，需确认当前版本是否支持）：

```sql
-- COMPATIBLE_MODE=8：兼容 DB2
SP_SET_PARA_VALUE(2, 'COMPATIBLE_MODE', 8);

-- 大小写不敏感
ALTER SYSTEM SET CASE_SENSITIVE = '0';

-- 字符串不填充尾部空格
SP_SET_PARA_VALUE(2, 'BLANK_PAD_MODE', 0);
```

### 序列迁移

```sql
-- DB2
CREATE SEQUENCE order_seq START WITH 1000 INCREMENT BY 1;

-- DM
CREATE SEQUENCE order_seq START WITH 1000 INCREMENT BY 1;
-- 兼容，语法一致
-- 取值：
-- DB2: VALUES NEXTVAL FOR order_seq
-- DM: SELECT order_seq.NEXTVAL FROM DUAL
```

## 通用迁移建议

- **先迁移 1-2 个非核心业务**做 PoC，验证完整链路
- 制定回滚方案：DMHS 反向同步、原库保留 1-2 周
- 应用层：JDBC 驱动替换 `dm.jdbc.driver.DmDriver`
- 性能压测：TPS 至少达到原库 90% 才算合格
- 数据校验：行数 + 关键字段 MD5 + 业务规则校验

### 迁移窗口期操作清单

| 阶段 | 操作 | 时长估计 |
|------|------|----------|
| T-7 天 | 通知业务方、确认停机窗口 | - |
| T-1 天 | 源库数据快照、DDL 备份 | 1 小时 |
| T-0（停机开始） | 停止源库写入、应用切只读 | 5 分钟 |
| T+0 | DTS/SQLark 全量迁移 | 1-4 小时（视数据量） |
| T+迁移完成 | 数据校验（行数 + MD5 + 抽样） | 30 分钟 - 1 小时 |
| T+校验通过 | 应用切流量到 DM | 10 分钟 |
| T+24 小时 | 持续观察慢 SQL、锁等待 | - |
| T+1 周 | 源库下线归档 | - |

### 常见回滚方案

1. **DMHS 反向同步**：DMHS 配置反向通道，DM 写回源库
2. **保留源库只读**：迁移成功后源库保留 1-2 周，应用可快速切回
3. **蓝绿部署**：新流量切 DM，旧流量仍走源库
4. **影子表对比**：DM 与源库双写，按天对比差异

### 应用层适配要点

- JDBC 驱动包名：`com.mysql.cj.jdbc.Driver` / `oracle.jdbc.OracleDriver` → `dm.jdbc.driver.DmDriver`
- JDBC URL：`jdbc:dm://host:5236`
- 连接池配置：DM 默认最大连接 100，需根据业务峰值调整 `MAX_SESSIONS`
- 字符集：客户端 NLS_LANG 与 DM 初始化字符集保持一致
- 事务隔离级别：DM 默认读已提交，应用需明确指定
- 批处理：DM 单批 INSERT 建议 1000-5000 行，避免大事务
