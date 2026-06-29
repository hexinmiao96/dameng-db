# DM8 系统包速查（DBMS_* / UTL_*）

> **本章目标**：快速查阅达梦 DM8 提供的 50+ 内置系统包（DBMS_*、UTL_*），覆盖每个常用包的**用途、关键函数、1-2 个 SQL/PLSQL 示例**。所有示例基于 DM8 官方手册（eco.dameng.com）整理，按使用频率排序。

## 1. 系统包总览

达梦 DM8 在新建库首次启动时会自动创建 50+ 个内置系统包，与 Oracle PL/SQL 包体系保持高度兼容。系统包按功能可分为以下 10 大类：

| 类别 | 代表包 | 主要用途 |
| --- | --- | --- |
| 调试输出 | DBMS_OUTPUT, DBMS_SESSION | PL/SQL 调试、日志、客户端消息缓冲 |
| 大对象 LOB | DBMS_LOB, DBMS_XMLGEN | BLOB/CLOB/XML 大字段读写、转换 |
| 动态 SQL | DBMS_SQL, DBMS_XMLGEN | 运行时拼装 SQL、DDL、XML |
| 元数据 | DBMS_METADATA, DBMS_DDL, DBMS_SYSTEM | 提取/重放 DDL、批量 DDL |
| 作业调度 | DBMS_JOB, DBMS_SCHEDULER | 定时任务、调度日历 |
| 错误诊断 | DBMS_UTILITY, DBMS_ERRLOG | 错误堆栈、调用栈、错误日志 |
| 统计信息 | DBMS_STATS, DBMS_SQLTUNE | 收集/删除直方图、SQL 调优 |
| 闪回恢复 | DBMS_FLASHBACK, DBMS_REDEFINITION | 会话级闪回查询、在线重定义 |
| 加密与编码 | DBMS_CRYPTO, UTL_RAW, UTL_ENCODE, UTL_I18N | 散列、对称加密、字节转换 |
| 文件与网络 | UTL_FILE, UTL_HTTP, UTL_SMTP, UTL_TCP, UTL_URL | 文件读写、HTTP/邮件/TCP |
| 审计与安全 | DBMS_AUDIT, DBMS_RLS, DBMS_LDAP | 细粒度审计、行级安全 |
| 高级特性 | DBMS_XA, DBMS_AQ/AQADM, DBMS_MVIEW, DBMS_XPLAN | 分布式事务、高级队列、SQL plan |
| XML/JSON | DBMS_XMLDOM, DBMS_XMLPARSER, DBMS_JSON | XML/JSON 解析 |
| 内部诊断 | DBMS_SYSTEM, DBMS_DEBUG, DBMS_APPLICATION_INFO | 跟踪、DEBUG 钩子 |

### 1.1 系统包创建与启用

DM 在新库第一次启动时**自动创建**所有系统包（含 DBMS_JOB 之外的全部），但部分包需要 DBA 手动启用：

```sql
-- 创建/删除全部系统包（不含 UTL_MAIL）
CALL SP_CREATE_SYSTEM_PACKAGES(1);   -- 1=创建, 0=删除, 2=重建

-- 创建/删除单个系统包
CALL SP_CREATE_SYSTEM_PACKAGES(1, 'DBMS_OUTPUT');

-- 特殊 4 个包需要专用过程
CALL SP_INIT_JOB_SYS(1);              -- DBMS_JOB
CALL SP_INIT_DBMS_SCHEDULER_SYS(1);   -- DBMS_SCHEDULER
CALL SP_INIT_RLS_SYS(1);              -- DBMS_RLS
CALL SP_INIT_AWR_SYS(1);              -- DBMS_WORKLOAD_REPOSITORY

-- 检测系统包启用状态
SELECT SF_CHECK_SYSTEM_PACKAGES();              -- 0=未启用, 1=已启用
SELECT SF_CHECK_SYSTEM_PACKAGE('DBMS_JOB');     -- 单个包
```

### 1.2 包间依赖

部分包存在自动依赖：DBMS_METADATA 依赖 DBMS_LOB+UTL_RAW；DBMS_XMLGEN 依赖 DBMS_LOB；DBMS_SQL 依赖 DBMS_LOB；DBMS_DDL 依赖 DBMS_SQL；DBMS_AQ 依赖 DBMS_AQADM；UTL_I18N 依赖 UTL_RAW。**DBMS_PIPE 依赖 DBMS_XMLGEN**，需要用户手动建依赖包。

## 2. DBMS_OUTPUT（调试输出）

`DBMS_OUTPUT` 把 PL/SQL 程序中的文本写入内存缓冲区，由客户端读取显示，是 PL/SQL 调试必备包。**注意：DIsql 中需要 `SET SERVEROUTPUT ON;` 才能看到输出。**

### 2.1 关键方法

| 方法 | 说明 |
| --- | --- |
| `ENABLE(buffer_size INT DEFAULT 20000)` | 启用包，指定缓冲区（实际取 2000~1000000） |
| `DISABLE` | 禁用包，释放缓冲区 |
| `PUT(str VARCHAR)` | 写一段到当前行，**不换行** |
| `PUT_LINE(str VARCHAR)` | 写一行到缓冲区，**自动换行**（最常用） |
| `NEW_LINE` | 单独插入一个行结束符 |
| `GET_LINE(line OUT, status OUT)` | 从缓冲区读一行，status=0 成功，1 失败 |
| `GET_LINES(lines OUT, numlines IN OUT)` | 批量读取多行 |

### 2.2 示例

```sql
SP_CREATE_SYSTEM_PACKAGES(1, 'DBMS_OUTPUT');
SET SERVEROUTPUT ON;

DECLARE
  v_buf   VARCHAR(100);
  v_st    INT;
BEGIN
  DBMS_OUTPUT.ENABLE(10000);
  DBMS_OUTPUT.PUT('达梦');
  DBMS_OUTPUT.PUT_LINE('数据库有限公司');
  DBMS_OUTPUT.GET_LINE(v_buf, v_st);
  DBMS_OUTPUT.PUT_LINE('读出=' || v_buf || ' status=' || v_st);
END;
/
```

## 3. DBMS_LOB（大对象操作）

`DBMS_LOB` 用于操作 BLOB / CLOB / BFILE 大对象，是处理图片、文档、长文本的核心包。**依赖 UTL_RAW。**

### 3.1 关键方法（按使用频率）

| 方法 | 说明 |
| --- | --- |
| `APPEND(dest_lob, src_lob)` | 把 src 追加到 dest 末尾 |
| `COPY(dest, src, amount, dest_off, src_off)` | 拷贝指定长度数据 |
| `ERASE(lob, amount IN OUT, offset)` | 擦除数据（BLOB 补 0x00，CLOB 补空格） |
| `GETLENGTH(lob) RETURN BIGINT` | 返回字节/字符长度 |
| `READ(lob, amount IN OUT, offset, buffer OUT)` | 读取子串到缓冲区 |
| `WRITE(lob, amount, offset, buffer)` | 从 offset 写入（覆盖） |
| `WRITEAPPEND(lob, amount, buffer)` | 追加到末尾 |
| `TRIM(lob, new_len)` / `TRIM_LOB` | 截断到 new_len |
| `SUBSTR(lob, amount, offset) RETURN` | 取子串 |
| `INSTR(lob, pattern, offset, nth) RETURN` | 查找子串位置 |
| `COMPARE(lob1, lob2, amount, off1, off2) RETURN INT` | 比较（0=相等，-1/1=大小） |
| `CREATETEMPORARY(lob, cache, dur)` | 创建临时 LOB（dur=SESSION/CALL） |
| `FREETEMPORARY(lob)` | 释放临时 LOB |
| `OPEN/CLOSE/ISOPEN/ISTEMPORARY` | 显式打开/关闭 LOB |
| `LOADBLOBFROMFILE / LOADCLOBFROMFILE` | 从 BFILE 加载到 BLOB/CLOB |
| `CONVERTTOBLOB / CONVERTTOCLOB` | BLOB↔CLOB 编码转换 |
| `GETCHUNKSIZE` | 块大小 |

### 3.2 示例：BLOB 转 CLOB

```sql
SP_CREATE_SYSTEM_PACKAGES(1, 'DBMS_LOB');
SP_CREATE_SYSTEM_PACKAGES(1, 'UTL_RAW');

CREATE OR REPLACE FUNCTION BLOBTOCLOB(BLOB_IN IN BLOB) RETURN CLOB AS
  V_CLOB    CLOB;
  V_VARCHAR VARCHAR2(32767);
  V_START   PLS_INTEGER := 1;
  V_BUFFER  PLS_INTEGER := 32767;
BEGIN
  DBMS_LOB.CREATETEMPORARY(V_CLOB, TRUE);
  FOR I IN 1 .. FLOOR(DBMS_LOB.GETLENGTH(BLOB_IN) / V_BUFFER) + 1 LOOP
    V_VARCHAR := UTL_RAW.CAST_TO_VARCHAR2(
                   DBMS_LOB.SUBSTR(BLOB_IN, V_BUFFER, V_START));
    DBMS_LOB.WRITEAPPEND(V_CLOB, LENGTH(V_VARCHAR), V_VARCHAR);
    V_START := V_START + V_BUFFER;
  END LOOP;
  RETURN V_CLOB;
END;
/
SELECT BLOBTOCLOB('B4EFC3CECAFDBEDDBFE2D3D0CFDEB9ABCBBE');
-- 返回：达梦数据库有限公司
```

## 4. DBMS_RANDOM（随机数生成）

提供**正态分布随机数、随机字符串、范围随机数**。DM 兼容 Oracle。

### 4.1 关键方法

| 方法 | 说明 |
| --- | --- |
| `INITIALIZE(val INT)` | 用整数初始化随机种子（-2^31 ~ 2^31-1） |
| `SEED(val INT/VARCHAR2)` | 重置种子（支持字符串，内部转 INT） |
| `TERMINATE` | 仅语法支持，无实际作用 |
| `VALUE(low NUMBER, high NUMBER) RETURN NUMBER` | 生成 [low, high] 范围随机数 |
| `RANDOM RETURN INTEGER` | 生成随机整数 |
| `NORMAL RETURN NUMBER` | 标准正态分布随机数（μ=0, σ=1） |
| `RANDOM_STRING(opt CHAR, len NUMBER) RETURN VARCHAR2` | 按模式生成字符串 |
| `STRING(opt, len) RETURN VARCHAR2` | 同 RANDOM_STRING（Oracle 兼容别名） |

`opt` 模式：`u/U` 大写字母、`l/L` 小写、`a/A` 大小写混合、`x/X` 大写+数字、`p/P` 可打印字符。

### 4.2 示例

```sql
SP_CREATE_SYSTEM_PACKAGES(1, 'DBMS_RANDOM');
CALL DBMS_RANDOM.INITIALIZE(15);                   -- 初始化种子
SELECT DBMS_RANDOM.VALUE(10, 100) FROM DUAL;       -- 10~100 之间 NUMBER
SELECT DBMS_RANDOM.STRING('X', 16) FROM DUAL;      -- 16 位大写字母+数字
SELECT DBMS_RANDOM.NORMAL() FROM DUAL;             -- 正态分布
```

## 5. DBMS_METADATA（提取 DDL）

`DBMS_METADATA` 把数据库对象（表/视图/存储过程/索引等）的元数据提取为**DDL 语句或 XML**，比 `SHOW CREATE TABLE` 更强大——支持批量、按模式过滤、依赖对象、授权语句。**依赖 DBMS_LOB+UTL_RAW。**

### 5.1 关键方法

| 方法 | 说明 |
| --- | --- |
| `GET_DDL(object_type, name, schname) RETURN CLOB` | **取单个对象 DDL**（最常用） |
| `GET_DEPENDENT_DDL(object_type, base_name, base_schema)` | 取依赖对象（约束、索引、触发器）DDL |
| `GET_GRANTED_DDL(object_type, grantee)` | 取授权语句 DDL |
| `OPEN(object_type) RETURN INT` | 打开游标（程序化接口） |
| `SET_FILTER(handle, name, value)` | 过滤（SCHEMA/NAME 等） |
| `ADD_TRANSFORM(handle, 'DDL')` | 转换输出格式（DDL/XML） |
| `SET_TRANSFORM_PARAM(th, 'CONSTRAINTS', FALSE)` | 过滤约束/存储选项 |
| `FETCH_CLOB(handle) RETURN CLOB` | 取一条 CLOB 格式 |
| `FETCH_DDL(handle) RETURN KU$_DDLS` | 取一组带解析项的 DDL |
| `CLOSE(handle)` / `CLOSE_ALL` | 关闭游标 |
| `GET_QUERY(handle) RETURN VARCHAR` | 获取底层查询 SQL |
| `CHECK_HANDLE` | 打印游标使用情况（调试用） |

支持的 object_type 30+ 种：`TABLE、VIEW、INDEX、PROCEDURE、FUNCTION、PACKAGE、TRIGGER、SEQUENCE、SYNONYM、USER、ROLE、TABLESPACE、PROFILE、SCHEMA_EXPORT、TABLE_EXPORT、DATABASE_EXPORT、MATERIALIZED_VIEW、COMMENT、CONSTRAINT、OBJECT_GRANT` 等。

### 5.2 示例

```sql
SP_CREATE_SYSTEM_PACKAGES(1, 'DBMS_METADATA');

-- 例1：直接取单表 DDL
SELECT DBMS_METADATA.GET_DDL('TABLE', 'T1', 'SYSDBA') FROM DUAL;

-- 例2：程序化批量取表 DDL
DECLARE
  h   NUMBER;
  doc CLOB;
BEGIN
  h := DBMS_METADATA.OPEN('TABLE');
  DBMS_METADATA.SET_FILTER(h, 'SCHEMA', 'SYSDBA');
  DBMS_METADATA.ADD_TRANSFORM(h, 'DDL');
  LOOP
    doc := DBMS_METADATA.FETCH_CLOB(h);
    EXIT WHEN doc IS NULL;
    -- 业务处理：保存到文件/日志表
  END LOOP;
  DBMS_METADATA.CLOSE(h);
END;
/

-- 例3：去掉约束输出（SET_TRANSFORM_PARAM）
CALL DBMS_METADATA.SET_TRANSFORM_PARAM(
  DBMS_METADATA.SESSION_TRANSFORM, 'CONSTRAINTS', FALSE);
SELECT DBMS_METADATA.GET_DDL('TABLE', 'T1', 'SYSDBA') FROM DUAL;
```

> ⚠️ 系统内建聚集索引受 `INNER_INDEX_DDL_SHOW` 参数控制：1=显示为 "INNER CLUSTER INDEX"，0=报错。

## 6. DBMS_SQL（动态 SQL）

`DBMS_SQL` 在 PL/SQL 中执行**动态拼装的 SQL 语句**，支持参数绑定、批量绑定、批量 fetch，比 EXECUTE IMMEDIATE 更细粒度控制。**依赖 DBMS_LOB。**

### 6.1 关键方法

| 方法 | 说明 |
| --- | --- |
| `OPEN_CURSOR RETURN INT` | 打开游标，返回游标号 |
| `PARSE(cursor, statement, language_flag)` | 解析 SQL（language_flag 仅语法兼容） |
| `BIND_VARIABLE(cursor, name, value)` | 绑定单值参数 |
| `BIND_ARRAY(cursor, name, table_variable)` | 绑定数组（批量绑定） |
| `DEFINE_COLUMN(cursor, position, col)` | 定义单值接收列 |
| `DEFINE_ARRAY(cursor, position, tab, least, start)` | 定义数组接收列 |
| `EXECUTE(cursor) RETURN INT` | 执行，返回受影响行数（INSERT/UPDATE/DELETE） |
| `FETCH_ROWS(cursor) RETURN INT` | 取一行，0=游标结束 |
| `COLUMN_VALUE(cursor, position, value OUT)` | 取列值到变量 |
| `COLUMN_VALUE_RAW / COLUMN_VALUE_CHAR` | 二进制/字符列值 |
| `VARIABLE_VALUE(cursor, name, value OUT)` | 取绑定的变量值 |
| `EXECUTE_AND_FETCH(cursor, exact)` | 执行+取一行 |
| `CLOSE_CURSOR(cursor)` | 关闭游标 |
| `DESCRIBE_COLUMNS(cursor, col_cnt OUT, desc_t OUT)` | 取列描述（类型、长度、精度） |
| `TO_CURSOR_NUMBER / TO_REFCURSOR` | DBMS_SQL 游标 ↔ SYS_REFCURSOR 互转 |

支持 18 种索引表类型（`NUMBER_TABLE / VARCHAR2_TABLE / DATE_TABLE / BLOB_TABLE / CLOB_TABLE / BINARY_FLOAT_TABLE` 等）。

### 6.2 示例

```sql
SP_CREATE_SYSTEM_PACKAGES(1, 'DBMS_SQL');
SP_CREATE_SYSTEM_PACKAGES(1, 'DBMS_OUTPUT');
SET SERVEROUTPUT ON;

-- 例1：单行查询
DECLARE
  c  NUMBER;
  d1 NUMBER;
BEGIN
  c := DBMS_SQL.OPEN_CURSOR;
  DBMS_SQL.PARSE(c, 'SELECT N1 FROM SYSDBA.T1', 1);
  DBMS_SQL.DEFINE_COLUMN(c, 1, d1);
  DBMS_SQL.EXECUTE(c);
  DBMS_SQL.FETCH_ROWS(c);
  DBMS_SQL.COLUMN_VALUE(c, 1, d1);
  DBMS_OUTPUT.PUT_LINE(d1);
  DBMS_SQL.CLOSE_CURSOR(c);
END;
/

-- 例2：批量绑定（数组）
DECLARE
  c  NUMBER;
  d1 DBMS_SQL.NUMBER_TABLE;
BEGIN
  d1(1) := 12.3; d1(2) := 6.12; d1(3) := 8.12;
  c := DBMS_SQL.OPEN_CURSOR;
  DBMS_SQL.PARSE(c, 'INSERT INTO SYSDBA.T1 VALUES(:c1)', 1);
  DBMS_SQL.BIND_ARRAY(c, 'c1', d1);
  DBMS_SQL.EXECUTE(c);
  DBMS_SQL.CLOSE_CURSOR(c);
END;
/
```

## 7. DBMS_JOB / DBMS_SCHEDULER（作业调度）

达梦提供两套作业系统：**DBMS_JOB**（Oracle 兼容 JOB 风格）和 **DBMS_SCHEDULER**（Oracle 11g 风格调度器）。MPP 环境**不支持** DBMS_JOB；DSC 集群支持但需指定 INSTANCE（节点号 0~15）。

### 7.1 DBMS_JOB 关键方法

| 方法 | 说明 |
| --- | --- |
| `SUBMIT(job OUT, what, next_date, interval, ...)` | 提交作业，返回 job 号 |
| `ISUBMIT(job IN, what, next_date, interval, no_parse)` | 用指定 job 号提交 |
| `REMOVE(job)` | 删除作业（运行中不能删） |
| `RUN(job, force)` | 立即执行（force=TRUE 异步） |
| `BROKEN(job, broken BOOLEAN, next_date)` | 标记 broken/恢复 |
| `WHAT(job, what IN OUT)` | 修改作业内容（in 为空时返回原值） |
| `NEXT_DATE(job, next_date)` | 修改下次执行时间 |
| `INTERVAL(job, interval)` | 修改执行间隔（保留字，调用需加双引号） |
| `CHANGE(job, what, next_date, interval, instance, force)` | 一次性改全部参数 |

`interval` 是日期表达式字符串：`'SYSDATE + 1/1440'` = 1 分钟；`'SYSDATE + 1/24'` = 1 小时；`'SYSDATE + 1'` = 1 天。**不支持周/月/季。**

相关视图：`DBA_JOBS、USER_JOBS、DBA_JOBS_RUNNING`。

### 7.2 DBMS_SCHEDULER

功能更丰富的调度器，支持日历表达式、程序定义、作业链、事件触发等。需调用 `SP_INIT_DBMS_SCHEDULER_SYS(1)` 创建。

### 7.3 DBMS_JOB 示例

```sql
SP_INIT_JOB_SYS(1);   -- 必须先建包

-- 每分钟插入 SYSDATE 到 A 表
CREATE TABLE A(A DATETIME);
CREATE OR REPLACE PROCEDURE TEST AS
BEGIN
  INSERT INTO A VALUES(SYSDATE);
  COMMIT;
END;
/

BEGIN
  DBMS_JOB.ISUBMIT(1, 'TEST', SYSDATE, 'SYSDATE+1/1440');
  COMMIT;
END;
/

-- 查看作业
SELECT JOB, WHAT, INTERVAL, NEXT_DATE, BROKEN FROM DBA_JOBS;
-- 删除
BEGIN DBMS_JOB.REMOVE(1); END;
/
```

## 8. DBMS_UTILITY（错误诊断/工具）

`DBMS_UTILITY` 是 PL/SQL 程序的"瑞士军刀"：错误堆栈、调用栈、HASH、字符串/表互转、执行 DDL。

### 8.1 关键方法

| 方法 | 说明 |
| --- | --- |
| `FORMAT_ERROR_STACK RETURN VARCHAR2` | 当前异常的错误堆栈 |
| `FORMAT_ERROR_BACKTRACE RETURN VARCHAR2` | 异常发生时的反向调用堆栈（带方法名+行号） |
| `FORMAT_CALL_STACK RETURN VARCHAR2` | 当前调用栈（不带行号） |
| `GET_HASH_VALUE(name, base, hash_size) RETURN INT` | 字符串 hash（[base, base+hash_size-1]） |
| `GET_TIME RETURN NUMBER` | 1/100 秒精度计时器（用于计算耗时） |
| `COMMA_TO_TABLE(list, tablen OUT, tab OUT LNAME_ARRAY)` | 逗号分隔字符串 → 索引表 |
| `TABLE_TO_COMMA(tab, tablen OUT, list OUT)` | 索引表 → 逗号分隔字符串 |
| `EXEC_DDL_STATEMENT(parse_string)` | 执行 DDL（PL/SQL 中动态建表/索引） |

### 8.2 示例：错误诊断

```sql
SP_CREATE_SYSTEM_PACKAGES(1, 'DBMS_OUTPUT');
SP_CREATE_SYSTEM_PACKAGES(1, 'DBMS_UTILITY');
SET SERVEROUTPUT ON;

DECLARE
  v_name  VARCHAR2(30) := 'NOT_EXIST_TABLE';
BEGIN
  EXECUTE IMMEDIATE 'SELECT * FROM ' || v_name;
EXCEPTION
  WHEN OTHERS THEN
    DBMS_OUTPUT.PUT_LINE('错误代码: ' || SQLCODE);
    DBMS_OUTPUT.PUT_LINE('错误信息: ' || SQLERRM);
    DBMS_OUTPUT.PUT_LINE('错误堆栈: ' || DBMS_UTILITY.FORMAT_ERROR_STACK);
    DBMS_OUTPUT.PUT_LINE('调用栈: '   || DBMS_UTILITY.FORMAT_CALL_STACK);
    DBMS_OUTPUT.PUT_LINE('反向回溯: ' || DBMS_UTILITY.FORMAT_ERROR_BACKTRACE);
END;
/
```

## 9. DBMS_STATS（统计信息收集）

`DBMS_STATS` 收集/删除/导入/导出表/索引/列的**优化器统计信息**（行数、页数、不同值数 NDV、直方图）。**统计信息不准是慢 SQL 第一大原因**。执行后会提交当前事务。

### 9.1 关键方法

| 方法 | 说明 |
| --- | --- |
| `GATHER_TABLE_STATS(ownname, tabname, partname, estimate_percent, method_opt, ...)` | **收集单表+列+索引统计**（最常用） |
| `GATHER_INDEX_STATS(ownname, indname, ...)` | 收集索引统计 |
| `GATHER_SCHEMA_STATS(ownname, options='GATHER')` | **收集整个 schema 统计** |
| `DELETE_TABLE_STATS(ownname, tabname, cascade_parts, cascade_columns, cascade_indexes)` | 删除表统计 |
| `DELETE_SCHEMA_STATS(ownname)` | 删除 schema 统计 |
| `DELETE_COLUMN_STATS / DELETE_INDEX_STATS` | 删除列/索引统计 |
| `TABLE_STATS_SHOW(ownname, tabname)` | **查看表统计**（NUM_ROWS/LEAF_BLOCKS） |
| `COLUMN_STATS_SHOW(ownname, tabname, colname)` | 查看列统计+直方图 |
| `INDEX_STATS_SHOW(ownname, indexname)` | 查看索引统计+直方图 |
| `SET_TABLE_STATS(ownname, tabname, numrows, numblks, avgrlen, ...)` | 手动设置表统计（用于修复失真） |
| `SET_TABLE_PREFS(ownname, tabname, ppname, pvalue)` | 设置表级静态属性（STALE_PERCENT/METHOD_OPT/DEGREE） |
| `GET_PREFS(ppname, ownname, tabname)` | 读取表级属性 |
| `CREATE_STAT_TABLE / DROP_STAT_TABLE` | 创建/删除统计信息备份表 |
| `EXPORT_TABLE_STATS / IMPORT_TABLE_STATS` | 导出/导入表统计（跨实例迁移） |
| `EXPORT_SCHEMA_STATS / IMPORT_SCHEMA_STATS` | 导出/导入整个 schema |
| `EXPORT_DATABASE_STATS / IMPORT_DATABASE_STATS` | 导出/导入全库 |
| `UPDATE_ALL_STATS` | 更新已有统计 |
| `CONVERT_RAW_VALUE` | 转换内部存储格式 |
| `COPY_TABLE_STATS(ownname, tabname, srcpart, dstpart)` | 复制分区统计 |

`METHOD_OPT` 格式：`FOR ALL COLUMNS SIZE AUTO/REPEAT/SKEWONLY/<n>`；`OPTIONS` 选项：`GATHER | GATHER STALE | GATHER EMPTY | LIST AUTO` 等。

### 9.2 示例

```sql
SP_CREATE_SYSTEM_PACKAGES(1, 'DBMS_STATS');

-- 例1：收集单表（含直方图、采样 30%）
CALL DBMS_STATS.GATHER_TABLE_STATS(
  'SYSDBA', 'T1', NULL, 30,
  FALSE, 'FOR ALL COLUMNS SIZE AUTO', 1, 'AUTO', TRUE);

-- 例2：收集整个 schema
CALL DBMS_STATS.GATHER_SCHEMA_STATS('SYSDBA', 30);

-- 例3：查看统计
CALL DBMS_STATS.TABLE_STATS_SHOW('SYSDBA', 'T1');
CALL DBMS_STATS.COLUMN_STATS_SHOW('SYSDBA', 'T1', 'C1');

-- 例4：手动设置（修复失真统计）
CALL DBMS_STATS.SET_TABLE_STATS('SYSDBA', 'T1', NULL,
  NULL, NULL, 1000000, 50000, 100);

-- 例5：导出/导入（跨环境同步）
CALL DBMS_STATS.CREATE_STAT_TABLE('SYSDBA', 'STATTAB');
CALL DBMS_STATS.EXPORT_SCHEMA_STATS('SYSDBA', 'STATTAB');
-- ...导出 STATTAB 表到目标库...
CALL DBMS_STATS.IMPORT_SCHEMA_STATS('SYSDBA', 'STATTAB');
```

## 10. DBMS_FLASHBACK（会话级闪回查询）

`DBMS_FLASHBACK` 在**会话级别**把视图切换到历史时间点/LSN，查询"过去"的数据库状态。**需 INI 参数 ENABLE_FLASHBACK=1**。不支持 MPP/DPC/数据守护备库；闪回对象不支持临时表、列存储表、外部表、视图；闪回模式下不能 DML/DDL。

### 10.1 关键方法

| 方法 | 说明 |
| --- | --- |
| `ENABLE_AT_TIME(query_time TIMESTAMP)` | 按时间闪回 |
| `ENABLE_AT_SYSTEM_CHANGE_NUMBER(query_scn BIGINT)` | **按 LSN 闪回** |
| `DISABLE` | 关闭闪回模式 |
| `GET_SYSTEM_CHANGE_NUMBER RETURN BIGINT` | **取当前 LSN** |

### 10.2 示例

```sql
SP_CREATE_SYSTEM_PACKAGES(1, 'DBMS_FLASHBACK');

CREATE TABLE T1(C1 INT, C2 INT);
INSERT INTO T1 VALUES(1, 1);
COMMIT;
SELECT DBMS_FLASHBACK.GET_SYSTEM_CHANGE_NUMBER;  -- 假设 117846
INSERT INTO T1 VALUES(2, 2);
COMMIT;

-- 闪回到 117846 时刻
DBMS_FLASHBACK.ENABLE_AT_SYSTEM_CHANGE_NUMBER(117846);
SELECT * FROM T1;   -- 只能看到 (1,1)
DBMS_FLASHBACK.DISABLE;
```

## 11. UTL_FILE（操作系统文件读写）

`UTL_FILE` 读写服务器操作系统上的文本/二进制文件，**必须通过 DIRECTORY 对象授权访问**（不是直接路径）。不支持 ASM 文件。

### 11.1 关键方法

| 方法 | 说明 |
| --- | --- |
| `FOPEN(location, filename, open_mode, max_linesize) RETURN FILE_TYPE` | 打开文件（r/w/a/rb/wb/ab） |
| `FCLOSE(file)` / `FCLOSE_ALL` | 关闭一个/全部 |
| `GET_LINE(file, buffer OUT, len)` | **读一行**（最常用） |
| `GET_RAW(file, buffer OUT, len)` | 读二进制 |
| `PUT(file, buffer)` / `PUT_LINE(file, buffer, autoflush)` | 写一段/写一行 |
| `PUT_RAW(file, buffer, autoflush)` | 写二进制 |
| `PUTF(file, format, arg1..arg5)` | printf 风格格式化（最多 5 个 %s） |
| `NEW_LINE(file, lines)` | 写入 N 个换行符 |
| `FFLUSH(file)` | 强制刷盘 |
| `FSEEK(file, absolute_offset, relative_offset)` | 调整文件指针 |
| `FGETPOS(file) RETURN INT` | 取当前文件位置 |
| `FCOPY(src_loc, src_file, dst_loc, dst_file, start, end)` | 文件内行拷贝 |
| `FRENAME / FREMOVE` | 重命名/删除 |
| `FGETATTR(location, filename, fexists, file_len, blocksize)` | 取文件属性 |
| `IS_OPEN(file) RETURN BOOLEAN` | 是否打开 |
| `FOPEN_NCHAR / GET_LINE_NCHAR / PUT_NCHAR / PUT_LINE_NCHAR / PUTF_NCHAR` | **UTF8 文件专用** |

`open_mode`：r 读、w 写（覆盖）、a 追加、rb/wb/ab 二进制版本；`max_linesize` 默认 1024，最大 32767。

### 11.2 示例

```sql
SP_CREATE_SYSTEM_PACKAGES(1, 'UTL_FILE');

-- 准备目录对象和权限
CREATE DIRECTORY DIR01 AS 'E:\UTL_FILE_TEMP';
GRANT READ, WRITE ON DIRECTORY DIR01 TO USER01;

-- 读文件
DECLARE
  v1 VARCHAR2(8186);
  f1 UTL_FILE.FILE_TYPE;
BEGIN
  f1 := UTL_FILE.FOPEN('DIR01', 'u12345.tmp', 'R');
  UTL_FILE.GET_LINE(f1, v1, 32767);
  DBMS_OUTPUT.PUT_LINE('GET LINE: ' || v1);
  UTL_FILE.FCLOSE(f1);
END;
/

-- 追加内容（PUTF printf 风格）
DECLARE
  h UTL_FILE.FILE_TYPE;
BEGIN
  h := UTL_FILE.FOPEN('DIR01', 'u12345.tmp', 'A');
  UTL_FILE.PUTF(h, '\nHELLO, %s!\nI AM FROM %s.\n', 'WORLD', 'DM8');
  UTL_FILE.FFLUSH(h);
  UTL_FILE.FCLOSE(h);
END;
/
```

## 12. UTL_RAW（字节/十六进制转换）

`UTL_RAW` 在 VARBINARY / VARCHAR2 / INTEGER / NUMBER / FLOAT / DOUBLE 之间做转换，**DBMS_LOB/DBMS_METADATA 内部强依赖**。

### 12.1 关键方法（20 个）

| 方法 | 说明 |
| --- | --- |
| `CAST_TO_RAW(c VARCHAR2) RETURN VARBINARY` | ASCII/字符串 → 十六进制 |
| `CAST_TO_VARCHAR2(r VARBINARY) RETURN VARCHAR2` | 十六进制 → 字符串 |
| `CAST_TO_INT / CAST_FROM_INT` | 十六进制 ↔ INTEGER（支持大小端 1/2/3） |
| `CAST_TO_BINARY_INTEGER / CAST_FROM_BINARY_INTEGER` | 同上（Oracle 兼容别名） |
| `CAST_TO_NUMBER / CAST_FROM_NUMBER` | 十六进制 ↔ NUMBER |
| `CAST_TO_BINARY_FLOAT / CAST_FROM_BINARY_FLOAT` | 十六进制 ↔ FLOAT |
| `CAST_TO_BINARY_DOUBLE / CAST_FROM_BINARY_DOUBLE` | 十六进制 ↔ DOUBLE |
| `BIT_AND / BIT_OR / BIT_XOR` | 按位与/或/异或 |
| `BIT_COMPLEMENT(r)` | 按位反码 |
| `COMPARE(r1, r2, pad) RETURN NUMBER` | 比较（0=相等，非 0=首个不等位置） |
| `CONCAT(r1..r12)` | 最多连接 12 个十六进制串 |
| `COPIES(r, n)` | 复制 n 份 |
| `LENGTH(r) RETURN NUMBER` | 字节长度 |
| `SUBSTR(r, pos, len)` | 子串 |
| `OVERLAY_RAW(overlay, target, pos, len, pad)` | 覆盖子串 |
| `REVERSE_RAW(r)` | 反转字节顺序 |
| `TRANSLATE(r, from_set, to_set)` | 字节替换 |
| `TRANSLITERATE(r, to_set, from_set, pad)` | 同 TRANSLATE |
| `XRANGE(start_byte, end_byte)` | 字节区间（如 'A' 到 'Z'） |
| `CONVERT_RAW(r, to_charset, from_charset)` | 字符集转换（UTF8↔GBK↔BIG5 等） |

### 12.2 示例

```sql
SP_CREATE_SYSTEM_PACKAGES(1, 'UTL_RAW');

SELECT UTL_RAW.CAST_TO_RAW('ABCD')        FROM DUAL;  -- 0x41424344
SELECT UTL_RAW.CAST_TO_VARCHAR2('40')     FROM DUAL;  -- @
SELECT UTL_RAW.CAST_TO_INT('00000064',1)  FROM DUAL;  -- 100
SELECT UTL_RAW.XRANGE(UTL_RAW.CAST_TO_RAW('A'),
                      UTL_RAW.CAST_TO_RAW('Z')) FROM DUAL;  -- 0x41...5A
-- 字符集转换 UTF8 → GBK
SELECT UTL_RAW.CONVERT_RAW('E4B8ADE69687', 'GBK', 'UTF8') FROM DUAL;
```

## 13. DBMS_CRYPTO（加密解密）

`DBMS_CRYPTO` 提供工业标准加密/散列。**支持的算法：**

- **分组加密**：DES、3DES、3DES_2KEY、AES128、AES192、AES256
- **工作模式**：ECB、CBC、CFB、OFB
- **填充模式**：PKCS5、NONE
- **流加密**：RC4
- **散列**：MD5、SHA-1、SHA-256、SHA-384、SHA-512

通过 `V$CIPHERS` 视图查询算法 KEY 长度（`KH_SIZE`）和分组大小（`BLOCK_SIZE`）。

### 13.1 关键方法

| 方法 | 说明 |
| --- | --- |
| `HASH(src VARBINARY/BLOB/CLOB, type) RETURN VARBINARY` | **散列**（HASH_MD5/HASH_SH1/HASH_SH256/HASH_SH384/HASH_SH512） |
| `ENCRYPT(src, typ, key, iv) RETURN VARBINARY` | **加密 VARBINARY** |
| `ENCRYPT(dst IN OUT BLOB, src, typ, key, iv)` | 加密 BLOB/CLOB |
| `DECRYPT(src, typ, key, iv) RETURN VARBINARY` | **解密 VARBINARY** |
| `DECRYPT(dst IN OUT, src, typ, key, iv)` | 解密 BLOB/CLOB |

`typ` = `加密算法 + 工作模式 + 填充模式`（如 `ENCRYPT_DES + CHAIN_ECB + PAD_PKCS5`）；`RC4` 只需算法名。

### 13.2 示例：DES 加密字符串

```sql
SP_CREATE_SYSTEM_PACKAGES(1, 'DBMS_CRYPTO');
SP_CREATE_SYSTEM_PACKAGES(1, 'DBMS_OUTPUT');
SP_CREATE_SYSTEM_PACKAGES(1, 'UTL_I18N');
SET SERVEROUTPUT ON;

DECLARE
  v_in   VARBINARY(128) := '74696765727469676572746967657274';
  v_md5  VARBINARY(2048);
  v_sha1 VARBINARY(2048);
BEGIN
  v_md5  := DBMS_CRYPTO.HASH(v_in, DBMS_CRYPTO.HASH_MD5);
  v_sha1 := DBMS_CRYPTO.HASH(v_in, DBMS_CRYPTO.HASH_SH1);
  DBMS_OUTPUT.PUT_LINE('MD5: '  || v_md5);
  DBMS_OUTPUT.PUT_LINE('SHA1: ' || v_sha1);
END;
/

-- 字符串 DES 加密（典型封装函数）
CREATE OR REPLACE FUNCTION ENCRYPT_FUNCTION(V_STR VARCHAR2, V_KEY VARCHAR2)
  RETURN VARCHAR2 AS
  v_key_raw  RAW(24);
  v_str_raw  RAW(2000);
  v_type     PLS_INTEGER;
BEGIN
  v_key_raw := UTL_I18N.STRING_TO_RAW(V_KEY, 'GBK');
  v_str_raw := UTL_I18N.STRING_TO_RAW(V_STR, 'GBK');
  v_type    := DBMS_CRYPTO.ENCRYPT_DES + DBMS_CRYPTO.CHAIN_ECB
             + DBMS_CRYPTO.PAD_PKCS5;
  v_str_raw := DBMS_CRYPTO.ENCRYPT(SRC => v_str_raw, TYP => v_type,
                                    KEY => v_key_raw);
  RETURN RAWTOHEX(v_str_raw);
END;
/
```

## 14. DBMS_AUDIT / DBMS_XPLAN（审计与执行计划）

### 14.1 DBMS_AUDIT（审计管理）

DBA 细粒度审计配置（用户/对象/操作维度），管理 SYSAUDITOR 系统用户下的审计记录。常用方法：`SET_AUDIT_OPTION / REMOVE_AUDIT_OPTION / SHOW_AUDIT_OPTIONS / SET_AUDIT_TRAIL`。**⚠️ 待官方手册确认**：DM 8.1+ 已重构为 SF 过程，建议同时查看《DM8 安全管理》手册。

### 14.2 DBMS_XPLAN（执行计划显示）

显示 SQL 实际/估算执行计划，DM 对应 `EXPLAIN` 命令的 PL/SQL 接口。

| 方法 | 说明 |
| --- | --- |
| `DISPLAY(table_name, statement_id)` | 显示已解释计划的 CLOB |
| `DISPLAY_CURSOR(sql_id, format)` | 显示**实际执行计划**（含 A-Rows/Starts 等运行时统计） |
| `DISPLAY_SQL_PLAN_BASELINE` | 显示 SQL plan baseline |
| `SET_PREVIEW_MODE` | 预览模式 |

```sql
-- 例：先 EXPLAIN，再 DISPLAY
EXPLAIN PLAN FOR SELECT * FROM T1 WHERE C1 = 1;
SELECT DBMS_XPLAN.DISPLAY FROM DUAL;
```

> ⚠️ DM 的 DBMS_XPLAN 函数签名与 Oracle 略有差异，**部分方法（如 DISPLAY_CURSOR）需要 V$SQL 视图数据**，使用前确认版本支持。

## 15. 其他重要包速查

### 15.1 DBMS_XA（分布式事务）

管理 XA 分布式事务（两阶段提交）。常用方法：`XA_START / XA_END / XA_PREPARE / XA_COMMIT / XA_ROLLBACK / XA_FORGET / XA_RECOVER`。配合应用服务器（Tuxedo/WebLogic）实现跨库事务。

### 15.2 DBMS_AQ / DBMS_AQADM（高级队列）

Oracle Streams AQ 兼容的消息队列，支持点对点/发布订阅。`DBMS_AQADM` 管理队列（CREATE_QUEUE_TABLE / CREATE_QUEUE / START_QUEUE），`DBMS_AQ` 收发消息（ENQUEUE / DEQUEUE）。

### 15.3 DBMS_MVIEW（物化视图）

管理物化视图的刷新：`REFRESH / REFRESH_ALL / REFRESH_DEPENDENT / PURGE_LOG / EXPLAIN_MVIEW / EXECUTE_MVIEW_SCHEDULE`。

### 15.4 DBMS_XMLGEN / DBMS_XMLDOM / DBMS_XMLPARSER（XML 系列）

- `DBMS_XMLGEN`：查询结果 → XML 字符串（`NEWXMLFROMQUERY`）
- `DBMS_XMLDOM`：DOM 树解析/构造
- `DBMS_XMLPARSER`：SAX 解析器

### 15.5 DBMS_SESSION（会话信息）

`SET_IDENTIFIER / SET_ROLE / SET_NLS / UNIQUE_SESSION_ID / CURRENT_INSTANCE_ID` 等，**配合 V$SESSION 视图调试**。

### 15.6 DBMS_SPACE（空间使用）

返回段/表空间的占用信息：`SPACE_USAGE / OBJECT_SPACE_USAGE`。

### 15.7 DBMS_ERRLOG（错误日志）

把 DML 错误记录到错误日志表而不是中断：`CREATE_ERROR_LOG / EXECUTE`。

### 15.8 DBMS_APPLICATION_INFO

注册应用模块/动作到 V$SESSION，便于性能追踪：`SET_MODULE / SET_ACTION / SET_CLIENT_INFO`。

### 15.9 DBMS_SYSTEM（内部诊断）

设置 SQL 跟踪、修改会话参数：`SET_SQL_TRACE_IN_SESSION / SET_EV`。

### 15.10 DBMS_DDL（DDL 操作）

`ALTER_COMPILE / ALTER_TABLE_NOT_REFERENCEABLE / SET_TRIGGER_FIRING_PROPERTY` 等。

### 15.11 DBMS_REDEFINITION（在线重定义）

不停机修改表结构：`START_REDEF_TABLE / FINISH_REDEF_TABLE / SYNC_INTERIM_TABLE / CAN_REDEF_TABLE`。

### 15.12 DBMS_SQLTUNE（SQL 调优）

`CAPTURE_CURSOR_CACHE / REPORT_TUNING_TASK / EXECUTE_TUNING_TASK / SET_TUNING_TASK_PARAMETER`。⚠️ 部分高级功能需要 **DBA 授权**。

### 15.13 DBMS_LOCK / DBMS_ALERT / DBMS_PIPE（协同）

- `DBMS_LOCK`：`REQUEST / RELEASE / SLEEP`（用户级锁）
- `DBMS_ALERT`：事件告警（`REGISTER / WAIT_ONE / WAIT_ANY / SET_DEFAULTS / REMOVE / SIGNAL`）
- `DBMS_PIPE`：管道消息（**依赖 DBMS_XMLGEN**）

### 15.14 DBMS_RLS（行级安全）

`ADD_POLICY / DROP_POLICY / ENABLE_POLICY / REFRESH_POLICY` 实现 VPD（Virtual Private Database）。

### 15.15 DBMS_LDAP / DBMS_OBSFUSCATION_TOOLKIT

- `DBMS_LDAP`：LDAP 目录服务访问
- `DBMS_OBSFUSCATION_TOOLKIT`：旧版 DES/MD5 加密（**DM 已基本由 DBMS_CRYPTO 取代**）

### 15.16 DBMS_TRANSACTION / DBMS_LOGMNR

- `DBMS_TRANSACTION`：`READ_ONLY / READ_WRITE / ADVISE_COMMIT / COMMIT_FORCE / COMMIT_COMMENT`
- `DBMS_LOGMNR`：日志挖掘（redo log → SQL）

### 15.17 DBMS_WORKLOAD_REPOSITORY（AWR）

`CREATE_SNAPSHOT / DROP_SNAPSHOT / AWR_REPORT_HTML / AWR_REPORT_TEXT`。需 `SP_INIT_AWR_SYS(1)` 启用。

### 15.18 DBMS_JSON

DM 8.1+ JSON 增强：`JSON_OBJECT / JSON_ARRAY / JSON_VALUE / JSON_QUERY / JSON_EXISTS`。

### 15.19 DBMS_ADVANCED_REWRITE

物化视图查询重写：`DECLARE_REWRITE_EQUIVALENCE / DROP_REWRITE_EQUIVALENCE`。

## 16. 完整系统包索引（30+）

按字母排序整理 50+ 系统包，便于反向查找：

| 包名 | 类别 | 备注 |
| --- | --- | --- |
| DBMS_ADVANCED_REWRITE | 优化 | 查询重写 |
| DBMS_ALERT | 协同 | 事件告警 |
| DBMS_APPLICATION_INFO | 诊断 | V$SESSION 标签 |
| DBMS_AQ | 高级队列 | 消息发送 |
| DBMS_AQADM | 高级队列 | 队列管理 |
| DBMS_AUDIT | 审计 | ⚠️ 部分方法用 SF 替代 |
| DBMS_BINARY | 工具 | 二进制实用 |
| DBMS_COMPRESSION ⚠️ | 压缩 | 待官方手册确认 |
| DBMS_CAPTURE_ADM ⚠️ | 流 | 待官方手册确认 |
| DBMS_CRYPTO | 加密 | DES/AES/MD5/SHA |
| DBMS_DDL | DDL | ALTER 操作 |
| DBMS_DEBUG | DEBUG | 内部 |
| DBMS_EDITIONS ⚠️ | 版本 | 待官方手册确认 |
| DBMS_EPG ⚠️ | 网关 | 待官方手册确认 |
| DBMS_ERRLOG | 错误 | DML 错误日志 |
| DBMS_FLASHBACK | 闪回 | 会话级 |
| DBMS_JOB | 调度 | Oracle JOB 风格 |
| DBMS_LDAP | 目录 | LDAP 访问 |
| DBMS_LOB | 大对象 | BLOB/CLOB |
| DBMS_LOCK | 锁 | 用户级锁 |
| DBMS_LOGMNR | 日志 | 挖掘 |
| DBMS_METADATA | 元数据 | DDL 提取 |
| DBMS_MVIEW | 物化视图 | 刷新 |
| DBMS_OBFUSCATION_TOOLKIT | 加密 | 旧版 |
| DBMS_OUTPUT | 输出 | 调试 |
| DBMS_PIPE | 协同 | 管道消息 |
| DBMS_PREDICTIVE_ANALYTICS ⚠️ | 分析 | 待官方手册确认 |
| DBMS_RANDOM | 随机 | 随机数 |
| DBMS_REDEFINITION | 重定义 | 在线重定义 |
| DBMS_REFRESH ⚠️ | 刷新 | 待官方手册确认 |
| DBMS_RLS | 安全 | VPD 行级安全 |
| DBMS_SCHEDULER | 调度 | 高级调度器 |
| DBMS_SESSION | 会话 | 会话信息 |
| DBMS_SNAPSHOT ⚠️ | 快照 | 待官方手册确认 |
| DBMS_SPACE | 空间 | 段空间 |
| DBMS_SQL | 动态 SQL | 运行时拼装 |
| DBMS_SQLDIAG ⚠️ | 诊断 | 待官方手册确认 |
| DBMS_SQLTUNE | 调优 | SQL 调优 |
| DBMS_STATS | 统计 | 收集/导出 |
| DBMS_SYSTEM | 内部 | 跟踪 |
| DBMS_TRANSACTION | 事务 | 控制 |
| DBMS_TYPES ⚠️ | 类型 | 待官方手册确认 |
| DBMS_UTILITY | 工具 | 错误/HASH |
| DBMS_WARNING ⚠️ | 告警 | 待官方手册确认 |
| DBMS_WORKLOAD_REPOSITORY | 性能 | AWR |
| DBMS_XA | 分布式 | XA 事务 |
| DBMS_XMLDOM | XML | DOM 解析 |
| DBMS_XMLGEN | XML | 查询→XML |
| DBMS_XMLPARSER | XML | SAX 解析 |
| DBMS_XPLAN | 执行计划 | EXPLAIN |
| UTL_COMPRESS | 压缩 | gzip/zip |
| UTL_ENCODE | 编码 | Base64/MIME |
| UTL_FILE | 文件 | 操作系统文件 |
| UTL_HTTP | HTTP | HTTP 客户端 |
| UTL_I18N | 国际化 | 字符集 |
| UTL_INADDR | 网络 | IP/域名 |
| UTL_MAIL | 邮件 | SMTP 发送 |
| UTL_MATCH | 匹配 | 模糊匹配 |
| UTL_RAW | 字节 | 十六进制 |
| UTL_SMTP | 邮件 | SMTP 协议 |
| UTL_TCP | 网络 | TCP 客户端 |
| UTL_URL | URL | 编码 |

> ⚠️ 标注"待官方手册确认"的包在 DM8 文档中链接可能不完整或为兼容 Oracle 占位，使用前请到 eco.dameng.com 验证。

## 17. 速查决策树：什么时候该用哪个包？

```
PL/SQL 调试输出?
└─ 是 → DBMS_OUTPUT.PUT_LINE

动态拼 SQL?
├─ 简单拼接 → EXECUTE IMMEDIATE
└─ 复杂（多次绑定/批量）→ DBMS_SQL

处理 BLOB/CLOB 大字段?
├─ 临时 LOB → DBMS_LOB.CREATETEMPORARY
├─ 读/写子串 → DBMS_LOB.READ/WRITE/SUBSTR
├─ 编码转换 → DBMS_LOB.CONVERTTOBLOB/CONVERTTOCLOB
└─ 二进制 ↔ 字符 → UTL_RAW

要提取表的 DDL?
├─ 快速查一张 → DBMS_METADATA.GET_DDL
└─ 批量 schema → DBMS_METADATA.OPEN + FETCH_CLOB

要跑定时任务?
├─ 简单定时 → DBMS_JOB
└─ 复杂日历/依赖 → DBMS_SCHEDULER

要收集统计信息?
├─ 单表 → DBMS_STATS.GATHER_TABLE_STATS
├─ 整 schema → DBMS_STATS.GATHER_SCHEMA_STATS
└─ 跨实例同步 → EXPORT/IMPORT_*_STATS

要回看历史数据?
├─ 会话级闪回 → DBMS_FLASHBACK.ENABLE_AT_TIME
└─ 已删除/误改 → FLASHBACK TABLE（DDL）

要读写服务器文件?
├─ 文本 → UTL_FILE.GET_LINE/PUT_LINE
└─ 二进制 → UTL_FILE.GET_RAW/PUT_RAW

要加密/散列?
├─ 散列 (MD5/SHA) → DBMS_CRYPTO.HASH
├─ 对称加密 (DES/AES) → DBMS_CRYPTO.ENCRYPT/DECRYPT
└─ 字节转换 → UTL_RAW

要调试错误?
├─ 错误堆栈 → DBMS_UTILITY.FORMAT_ERROR_STACK
└─ 反向回溯 → DBMS_UTILITY.FORMAT_ERROR_BACKTRACE
```

## 18. 最佳实践与踩坑提醒

1. **每个包先 SP_CREATE_SYSTEM_PACKAGES(1, 'XXX')**：DM 虽然自动创建了大部分包，但部分函数在新环境可能状态未知，**调用前先建包是稳妥做法**。
2. **DBMS_OUTPUT 必须 SET SERVEROUTPUT ON**：DIsql 默认关闭客户端输出。
3. **SP_INIT_XXX_SYS vs SP_CREATE_SYSTEM_PACKAGES**：DBMS_JOB / DBMS_SCHEDULER / DBMS_RLS / DBMS_WORKLOAD_REPOSITORY 走专用过程，其余走通用过程。
4. **DBMS_METADATA 的 SESSION_TRANSFORM 永久生效**：使用后要 `SET_TRANSFORM_PARAM(th, 'CONSTRAINTS', TRUE)` 还原。
5. **DBMS_LOB 子串操作单次最大 32K**（SUBSTR/FRAGMENT_INSERT 限制）；更大数据用循环。
6. **UTL_FILE 路径必须建 DIRECTORY**：用 `CREATE DIRECTORY ... AS '...'` + `GRANT READ/WRITE TO user`。
7. **DBMS_JOB 的 INTERVAL 不支持周/月/季**：如需周调度，`INTERVAL = 'TRUNC(SYSDATE) + 7'`，月调度用 `LAST_DAY + 1`。
8. **DBMS_FLASHBACK 在 MPP/DPC 不可用**：集群环境需其他方案（如 logmnr 闪回表）。
9. **DBMS_CRYPTO 的 KEY 长度查 V$CIPHERS.KH_SIZE**：长度不足会报错，超出部分被忽略。
10. **UTL_RAW 的 BIT_AND/OR/XOR 长度不等时按位补 0xFF/0x00**：跨长度比较时注意 padding 规则。

## 19. 参考资料

- 达梦官方文档中心：<https://eco.dameng.com/document/dm/zh-cn/pm/package-overview.html>
- 《DM8 系统包使用手册》：<https://eco.dameng.com/document/dm/zh-cn/pm/>
- 《DM8 SQL 程序设计》：PL/SQL 语言基础
- 《DM8 系统管理员手册》：系统包创建/管理
- Oracle PL/SQL Packages Reference（对照参考）：`DBMS_*` 包在 Oracle 与 DM 的兼容性差异
- 达梦技术社区：<https://eco.dameng.com/community/>
- 墨天轮 DM 专栏：<https://www.modb.pro/db?type=dameng>

> **版本说明**：本文基于 DM8 当前主流版本（≥8.1.3.x）整理，DBMS_JSON / DBMS_SQLTUNE 部分高级特性在更早版本可能不可用。升级后建议重新核对各包函数签名。
