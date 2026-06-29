# DM8 编程接口与底层 API

> 适用版本：DM8（达梦数据库管理系统）。本章为《程序员手册》精华，聚焦 13 种编程接口的定位、典型场景与最小调用示例。

## 1. 编程接口全景

DM8 提供完整的编程接口体系，覆盖 C、Java、.NET、PHP、Node.js、Go、Python 等主流语言。下表按"语言/场景 → 推荐接口"汇总。

| 接口 | 适用语言/场景 | 性能 | 关键特性 |
|------|--------------|------|---------|
| DPI（DM Programming Interface） | C/C++、高并发 | ★★★★★（比 ODBC 快 3-10 倍） | 达梦原生 C 接口、ODBC 3.0 兼容句柄 |
| ODBC 3.0 | C/C++/Python/通用 | ★★★ | 跨数据库、unixODBC 支持 |
| JDBC 3.0/4.0 | Java/Spring | ★★★★ | 连接池、批量、LOB、读写分离集群 |
| .NET Data Provider | C#/.NET Framework & Core | ★★★★ | EF6 + EFCore、连接池、BulkCopy |
| PHP 扩展 | PHP 5/7/8 | ★★★ | PDO、原生 dm\_\* API、PDO_DM |
| FLDR | C/Java（批量装载） | ★★★★★（百万行/秒级） | 高速数据载入、控制文件 |
| DEXP/DIMP JNI | Java（逻辑导入导出） | ★★★ | 表/模式/全库逻辑备份 |
| Logmnr JNI/C | C/Java（日志分析） | ★★★ | 归档日志挖掘、SQL 重放 |
| Node.js（dmdb） | Node.js 12+ | ★★★ | 连接池、Promise、流式查询 |
| Go（dm） | Go 1.13+ | ★★★★ | database/sql 兼容、Lob 扩展 |
| XA | C/分布式事务 | ★★★ | 两阶段提交、ORACLE/MySQL 兼容模式 |
| R2DBC | Java 响应式（WebFlux） | ★★★★★（非阻塞） | Reactor、Savepoint、Blob/Clob |

## 2. DPI（DM Programming Interface）

**定位**：达梦原生 C 接口，是访问 DM 数据库"最直接"的途径。参考 Microsoft ODBC 3.0 设计，函数命名 `dpi_xxx`（小写+下划线），与 ODBC 函数一一对应（例：`SQLAllocStmt` ↔ `dpi_alloc_stmt`）。

**典型场景**：C/C++ 高并发服务端、嵌入式应用、性能敏感的批量写入、高性能中间件。

**性能优势**：相比 ODBC 减少一层 Driver Manager 与部分协议解析开销，性能提升 3-10 倍，高并发首选。

**四类核心句柄**：`dhenv`（环境）、`dhcon`（连接）、`dhstmt`（语句）、`dhdesc`（描述符），外加 `dhloblctr`（LOB）、`dhobj`（复合类型）、`dhbfile`（BFILE）。

**返回值约定**：`DSQL_SUCCESS(0)`、`DSQL_SUCCESS_WITH_INFO(1)`、`DSQL_NO_DATA(100)`、`DSQL_ERROR(-1)`、`DSQL_INVALID_HANDLE(-2)`、`DSQL_NEED_DATA(99)`、`DSQL_STILL_EXECUTING(2)`。

**最小调用示例**（伪代码）：
```c
dpi_alloc_env(&henv);              // 1. 申请环境
dpi_alloc_con(henv, &hcon);        // 2. 申请连接
dpi_login(hcon, server, user, pwd);// 3. 登录
dpi_alloc_stmt(hcon, &hstmt);      // 4. 申请语句
dpi_exec_direct(hstmt, "SELECT ...");
dpi_fetch(hstmt, &col1, &col2);
dpi_free_stmt(hstmt); dpi_free_con(hcon); dpi_free_env(henv);
```

**支持协议**：`DSQL_INET_TCP/UDP/IPC/UNIXSOCKET/RDMA`，Linux 支持 UNIXSOCKET 直连（高性能本地通信）。

## 3. ODBC 编程

**定位**：跨数据库的工业标准接口，DM ODBC 3.0 兼容 Microsoft ODBC 3.0 规范。

**典型场景**：C/C++ 通用应用、PowerBuilder/C++Builder/Visual Studio 集成、跨数据库迁移、Python pyodbc 访问。

**Windows DSN 配置**：控制面板 → 管理工具 → ODBC 数据源管理器 → 系统 DSN → 添加 `DM8 ODBC DRIVER`（驱动文件 `dodbc.dll`）。

**Linux unixODBC 配置**（/etc/odbcinst.ini + /etc/odbc.ini）：
```ini
# odbcinst.ini
[DM8 ODBC DRIVER]
Driver = /lib/libdodbc.so

# odbc.ini
[dm]
Driver   = DM8 ODBC DRIVER
SERVER   = localhost
UID      = SYSDBA
PWD      = <DM_PASSWORD>
TCP_PORT = 5236
```

**连接方式**：`SQLConnect`（最基本）、`SQLDriverConnect`（字符串方式，支持 UNIXSOCKET）、`SQLBrowseConnect`（交互式分步）。

**最小调用示例**（C）：
```c
SQLAllocHandle(SQL_HANDLE_ENV, NULL, &henv);
SQLSetEnvAttr(henv, SQL_ATTR_ODBC_VERSION, SQL_OV_ODBC3, 0);
SQLAllocHandle(SQL_HANDLE_DBC, henv, &hdbc);
SQLConnect(hdbc, "DM", SQL_NTS, "SYSDBA", SQL_NTS, "<DM_PASSWORD>", SQL_NTS);
SQLAllocHandle(SQL_HANDLE_STMT, hdbc, &hstmt);
SQLExecDirect(hstmt, "SELECT * FROM t", SQL_NTS);
```

## 4. JDBC 编程

**定位**：Java 标准数据库访问接口，DM JDBC 包含 3.0 与 4.0 两个版本，对应 JDK 1.6/1.7/1.8/11，驱动 Jar 为 `DmJdbcDriver6/7/8/11.jar`。

**典型场景**：Java Web/Spring/JPA/MyBatis、Hibernate 集成、企业应用、Android 端。

**连接字符串**（三种格式）：
```java
// 格式一：host:port 不作属性
"jdbc:dm://localhost:5236/SYSDBA?compatibleMode=oracle"

// 格式二：host/port 作为属性
"jdbc:dm://?host=192.0.2.10&port=5236&user=APP_USER&password=<DM_PASSWORD>"

// 格式三：服务名（多节点）
"jdbc:dm://test?test=(192.0.2.10:5236,192.0.2.11:5237)"
```

**Statement 三件套**：
- `Statement`（静态 SQL）
- `PreparedStatement`（预编译 + 参数绑定，**推荐**）
- `CallableStatement`（存储过程调用，含 IN/OUT/INOUT）

**批处理与事务**：
```java
conn.setAutoCommit(false);
PreparedStatement ps = conn.prepareStatement("INSERT INTO t VALUES (?,?)");
for (int i = 0; i < 1000; i++) {
    ps.setInt(1, i); ps.setString(2, "v" + i);
    ps.addBatch();
}
ps.executeBatch();    // 批量执行
conn.commit();        // 手动提交
```

**BLOB/CLOB 流式读写**：
```java
// 写入 BLOB
ps.setBinaryStream(2, new FileInputStream("big.jpg"), fileLen);
// 读取 CLOB 流式
Reader r = rs.getCharacterStream("content");
```

**连接池**：通过 `DmdbDataSource` 注册到 JNDI 或应用服务器；底层 `javax.sql.ConnectionPoolDataSource` 由驱动实现。`Statement` 池默认 15，`PStmt` 池默认 0（按需开启）。

**连接串属性关键项**：`autoCommit`、`LobMode`（1=流式 2=全缓存）、`batchType`（1=批量绑定 2=逐行）、`continueBatchOnError`、`bufPrefetch`（32-65535 KB）、`compatibleMode`（oracle/oracle11/oracle19/mysql）、`schema`、`loginMode`、`RW_SEPARATE`。

## 5. .NET Data Provider

**定位**：达梦 .NET 生态入口，命名空间 `Dm`，主类 14 个：`DmConnection / DmCommand / DmDataAdapter / DmDataReader / DmParameter / DmParameterCollection / DmTransaction / DmCommandBuilder / DmConnectionStringBuilder / DmClob / DmBlob / DmBulkCopy / DmBulkCopy2 / DmTracingCommand`。

**典型场景**：C#/.NET Framework 4.5+、.NET Core、ASP.NET、WPF/WinForms、企业 ERP。

**连接串**（分号分隔）：
```csharp
"Server=localhost;UserId=SYSDBA;PWD=<DM_PASSWORD>;PORT=5236;charset=utf-8;connPooling=true;connPoolSize=100"
```

**关键属性**：`connPooling`（强烈建议开启）、`stmtPoolSize=15`、`poolSize=100`、`rw_separate`（读写分离）、`compatibleMode`、`isBdtaRS`（列模式结果集）、`lobMode`、`batchType/ContinueOnError`、`maxRows`、`socketTimeout`、`sslKeyPass`、`useSkyWalking`（链路追踪）。

**EF6 / EFCore 集成**：
- EF6：使用 `EFDmProvider.EF6` 包，`App.config` 注册 `<provider invariantName="Dm" type="EFDmProvider.DmProviderServices, EFDmProvider.EF6" />`
- EFCore：使用 `EFCore.Dm` 包，`OnConfiguring` 中 `UseDm("Server=...;PORT=5236;USER=SYSDBA;PASSWORD=<DM_PASSWORD>")`

**最小调用示例**：
```csharp
using Dm;
var conn = new DmConnection("Server=localhost;UserId=SYSDBA;PWD=<DM_PASSWORD>;PORT=5236");
conn.Open();
var cmd = new DmCommand("SELECT name FROM t WHERE id=:id", conn);
cmd.Parameters.Add(new DmParameter("id", DbType.Int32) { Value = 1 });
var rdr = cmd.ExecuteReader();
while (rdr.Read()) Console.WriteLine(rdr.GetString(0));
```

**批量装载 `DmBulkCopy2`**：使用 DM 私有协议，效率远高于 `DmBulkCopy`，推荐。`BatchSize=[100,10000]`、`IndexOption=1/2/3`（不刷新/重建/刷新二级索引）。

## 6. PHP、Node.js、Go

### 6.1 PHP

支持 PHP 5.2~8.4，提供两套 API：**原生 `dm_xxx` 函数**（`dm_connect / dm_pconnect / dm_close / dm_query / dm_fetch_array / dm_fetch_object / dm_execute / dm_bind_array_by_name`）和 **PDO_DM**（推荐）。php.ini 中 `extension=php53_dm.dll`（Windows）或 `extension=libphp53_dm.so`（Linux），设置 `dm.default_host/user/pw` 与 `LD_LIBRARY_PATH=$DM_HOME/bin`。

### 6.2 Node.js（dmdb）

Node.js ≥ 12，`npm install dmdb`。连接串 `dm://SYSDBA:<DM_PASSWORD>@localhost:5236`。支持 `createPool / getConnection / execute / executeMany / queryStream / Lob`，返回 Promise 或回调。`outFormat=OBJECT/ARRAY/AUTO` 控制结果格式。

```js
const db = require('dmdb');
db.createPool({ connectString: "dm://SYSDBA:<DM_PASSWORD>@localhost:5236" })
  .then(pool => pool.getConnection(conn => conn.execute(
      "INSERT INTO t VALUES (:1,:2)", [1, 'a'], () => conn.close())));
```

### 6.3 Go（dm）

兼容 `database/sql`，`sql.Open("dm", "dm://SYSDBA:<DM_PASSWORD>@localhost:5236?autoCommit=true")`。扩展对象 `DmBlob / DmClob / DmTimestamp / DmDecimal / DmIntervalYM / DmIntervalDT / DmArray`（均带 `Valid` 标识 NULL）。批量执行传 `[][]interface{}`。支持 `dialName` 自定义传输（`dm.RegisterDial`）。

## 7. FLDR 编程（快速装载）

**定位**：达梦超高速数据载入工具，C 版本比 INSERT 提升 1-2 个数量级（百万行/秒级）。同时提供 JNI 版本（`com.dameng.floader.Instance`）。

**典型场景**：初始化大批量数据导入、数据迁移 ETL 阶段、批量初始化测试库。

**核心 C API**：`fldr_alloc → fldr_set_attr → fldr_initialize → fldr_bind[_nth] → fldr_sendrows[_nth] → fldr_batch → fldr_finish → fldr_uninitialize → fldr_free`。`fldr_set_attr` 设置 `FLDR_ATTR_SERVER/UID/PWD/PORT/TABLE/INDEX_OPTION`（1=不刷新二级索引，装载后排序填入；2=装载后重建；3=实时刷新）、`BDTA_SIZE`（100-10000，默认 5000）、`DIRECT_MODE`（1=快速模式）、`READ_ROWS`、`SEND_NODE_NUM`、`TASK_THREAD_NUM`、`COMPRESS_FLAG`、`FLUSH_FLAG`、`LOAD_MODE`（1=载入 2=载出 3=Oracle 模式载出）。

**控制文件（CTL）**：使用 `fldr_exec_ctl_low` 通过 ctl 字符串控制，支持 `load / infile / into table / fields / TRAILING NULLCOLS` 等 SQL*Loader 兼容语法。

**导出模式**：`LOAD_MODE=2` 载出 + `EXPORT_MODE=2`（缓存模式 BUFFER），用 `fldr_fetch_data_len / fldr_fetch_data` 拉取数据，可与上层应用集成。

## 8. DEXP/DIMP JNI

**定位**：通过 JNI 在 Java 应用内调用 `dexp / dimp` 逻辑导入导出工具。

**典型场景**：跨模式数据迁移、定时备份任务、应用内嵌导出按钮、归档/回滚流程自动化。

**Java 类**：`com.dameng.impexp.ImpExpDLL`，包 `com.dameng.impexp.jar`。
- `dll_exp_dm(int mode, byte[] userid, byte[] schema, byte[] table, byte[] expFile, byte[] logFile)`：mode=1 表方式 / 2 模式方式 / 3 全库
- `dll_imp_dm(...)`：导入（多一个 `remapSchema` 模式映射）
- `dll_exec_sql_file(byte[] userid, byte[] sqlFile, byte[] logFile)`：执行 SQL 脚本
- `dll_exp_dm(List<String[]> params)` / `dll_imp_dm(List<String[]> params)`：用 List 键值对自由指定参数

## 9. Logmnr（日志挖掘）

**定位**：分析归档日志，重放 SQL 操作，用于审计、数据恢复、误操作追踪、增量同步。

**前置配置**：开启归档 `ARCH_INI=1`；开启逻辑日志 `RLOG_APPEND_LOGIC=1/2/3`。

**两套接口**：JNI（`com.dameng.logmnr.LogmnrDll`，Jar 在 `$DM_HOME/jar/logmnr.jar`）和 C（`dmlogmnr_client.dll`，头文件 `logmnr_client.h`）。

**调用流程**：`initLogmnr → createConnect(host, port, user, pwd) → addLogFile(connId, logPath, option=3) → startLogmnr(connId, trxid=-1, startTime=null, endTime=null) → getData(connId, rownum) → endLogmnr → closeConnect → deinitLogmnr`。

**关键属性**：`LOGMNR_ATTR_PARALLEL_NUM`（2-16，默认 2）、`BUFFER_NUM`（8-1024，默认 8）、`CONTENT_NUM`（256-2048，默认 256）、`TRX_END`（1=等事务结束）、`TRX_WAIT_TIME`（0-600s）、`OPTIONS`（与 `DBMS_LOGMNR.START_LOGMNR` 一致）。

**`LogmnrRecord` 字段**：`scn/startScn/commitScn`、`timestamp/startTimestamp/commitTimestamp`、`xid`、`operation`（INSERT/DELETE/UPDATE/BATCH_UPDATE/DDL/START/COMMIT/ROLLBACK/SEQ MODIFY/XA_COMMIT/UNSUPPORTED）、`sqlRedo`、`segOwner/tableName/rowId`、`rbasqn/rbablk/rbabyte`、`ssn/csf`（长 SQL 切片标记）。

## 10. XA 分布式事务

**定位**：符合 X/Open DTP 规范的 XA 接口，使外部事务管理器（TM，如 Oracle Tuxedo、Atomikos）能在 DM 上协调全局事务。

**库**：`dmxai.dll/libdmxai.so`（DM XA 库），配合 PRO*C 需 `dmdpc`，配合 DPI 需 `dmdpi`。

**核心接口**：`xa_open / xa_close / xa_start / xa_end / xa_rollback / xa_prepare / xa_commit / xa_recover / xa_forget`。流程：TM 先 `xa_start` 启动分支 → 业务 SQL → `xa_end` 分离 → `xa_prepare`（一阶段预提交）→ 所有 RM 成功 → `xa_commit`（二阶段提交），失败则 `xa_rollback`。

**打开字符串**：`HOST=127.0.0.1+USER=test+PWD=Test_12345`（`+` 分隔必填字段）。

**兼容模式 INI 参数**：`XA_COMPATIBLE_MODE=0`（默认，达梦错误码）/ 1（ORACLE 兼容）/ 2（MySQL 兼容，不支持 dbms_xa 包）。

## 11. R2DBC（响应式编程）

**定位**：基于 `r2dbc-spi-1.0.0.RELEASE` 的反应式数据库访问规范，**非阻塞、响应式**，适合 Spring WebFlux / Reactor 高并发场景。

**典型场景**：Spring WebFlux、Project Reactor、响应式微服务、高并发短连接。

**核心对象**：`io.r2dbc.spi.Connection / Statement / Result / Row / RowMetadata`；达梦实现为 `DmConnection / DmStatement / DmResult / DmRow / DmRowMetadata / DmBatch / DmConnectionFactory / DmConnectionFactoryProvider`。

**建立连接三种方式**：
```java
// URL（推荐，R2DBC URL 中用户名密码只支持字母数字）
ConnectionFactories.get("r2dbc:dm://SYSDBA:<DM_PASSWORD>@127.0.0.1:5236")

// Properties + Options
ConnectionFactoryOptions.builder()
    .option(HOST, "127.0.0.1").option(PORT, 5236)
    .option(USER, "SYSDBA").option(PASSWORD, "<DM_PASSWORD>").build();

// JDBC DataSource
dataSource.setURL("jdbc:dm://127.0.0.1");
options.option(DmConnectionFactoryProvider.DATASOURCE, dataSource);
```

**事务与 Savepoint**：`beginTransaction() / beginTransaction(IsolationLevel)`、`commitTransaction() / rollbackTransaction() / createSavepoint(name) / rollbackTransactionToSavepoint(name) / releaseSavepoint(name)`。

**批处理**：`connection.createBatch().add(sql).add(sql).execute()`。

## 12. 接口选型指南

| 业务场景 | 推荐接口 | 关键理由 |
|---------|---------|---------|
| C/C++ 高并发服务端 | **DPI** | 性能 3-10 倍优于 ODBC，协议最薄 |
| Java Spring 业务系统 | **JDBC + HikariCP/Druid** | 生态成熟，连接池、批量、读写分离完善 |
| Java 响应式微服务（WebFlux） | **R2DBC** | 非阻塞、Reactor 友好 |
| 跨数据库/多语言通用 | **ODBC** | 标准最广，pyodbc/Excel/PB 通用 |
| .NET / ASP.NET 应用 | **.NET Data Provider + EF6/EFCore** | ORM 集成度高 |
| Node.js 服务 | **dmdb** | 标准 `database/sql` 风格，Promise/流 |
| Go 服务 | **dm (database/sql)** | 标准库兼容，Lob 扩展 |
| PHP Web | **PDO_DM** | 通用性强，与 Laravel/Symfony 集成 |
| 百万/亿级批量导入 | **FLDR（C/JNI）** | 百万行/秒，原生高速通道 |
| 逻辑备份/模式迁移 | **dexp/dimp（CLI 或 JNI）** | 跨模式、跨平台、保留原始信息 |
| 归档日志审计/数据回溯 | **Logmnr（JNI/C）** | 重放 SQL、定位误操作 |
| 跨多库分布式事务 | **XA** | 2PC 全局一致性，ORACLE/MySQL 兼容模式 |
| Python 业务 | **dmPython / dmDjango / dmSQLAlchemy** | DB-API 2.0、ORM 方言包、连接池 |
| 应用内嵌 P/CRUD 而又需独立进程 | **DPI 多线程** | 稳定可控、无 JVM 启动开销 |

**选型三条铁律**：
1. **同语言同栈优先**：Java 用 JDBC/.NET 用 Provider/Go 用 dm，避免为了"性能"强行切到 DPI（C 项目除外）。
2. **高并发要看协议层**：DPI > JDBC > ODBC；R2DBC 适合响应式，但**业务复杂度不能换来就用 R2DBC**，同步阻塞模型下 JDBC 连接池仍是黄金标准。
3. **批量分两种**：业务级批量（10-1000 行）走 `executeBatch`/批量绑定；**装载级批量**（10 万+）必须走 FLDR 或 DEXP/DIMP。

**连接配置项优先级**（**注意** `dm_svc.conf` 与接口的覆盖关系）：
- **DPI/ODBC/PROC/dmPython/FLDR**：全局配置区 < 接口设置 < 服务配置区
- **JDBC/.NET/Go/PHP/DEXP-DIMP/Node.js/XA/R2DBC**：全局配置区 < 服务配置区 < 接口设置
- **Logmnr**：仅接口设置，不读 `dm_svc.conf`

## 13. 常见踩坑与最佳实践

1. **BLOB/CLOB 大对象**：JDBC 建议 `LobMode=1`（流式分批缓存），避免 1 GB 大对象把应用内存打爆；需要二次处理时再切到 `LobMode=2` 全缓存。
2. **批处理失败继续**：`continueBatchOnError=true` + `batchAllowMaxErrors=N` 可让部分行失败后继续执行，常用于 ETL 容错。
3. **预处理语句池**：`PStmtPoolSize=15`、`pstmtPoolValidTime=0`（永不过期），减少硬解析开销。
4. **编码与字符集**：连接串明确指定 `charset=utf-8` 或 `PG_UTF8`；服务器/客户端编码不一致会触发隐式转换，搜索/排序性能下降。
5. **读写分离路由**：通过 `loginMode` + `rw_separate` 控制事务一致性备库；金融场景必须保证"读己之写"，用 `rwSeparate=4`。
6. **SSL 加密**：客户端 `sslFilesPath / sslKeyPass`，服务端同步开启 `COMM_ENCRYPTION`。
7. **EF EnsureCreated/EnsureDeleted**：需要 `DBAPassword=SYSDBA密码`，否则无法创建/删除数据库（仅 EFCore 方言包）。
8. **ResultSet 流式 vs 全缓存**：`isBdtaRS=true` + `RS_BDTA_FLAG=2` 服务端开启列模式，超大结果集必备。
9. **Logmnr 长 SQL 切片**：`ssn` 相同的行表示同一 SQL，`csf=1` 表示中间片段，`csf=0` 表示最后/完整片段，应用层需按 `ssn` 拼装。
10. **XA 与 dbms_xa**：默认 `XA_COMPATIBLE_MODE=0`（达梦错误码），切换为 `2`（MySQL 兼容）时 `dbms_xa` 包不可用。

## 14. 参考资料

1. DM 程序员手册 - 概述：https://eco.dameng.com/document/dm/zh-cn/pm/programmer-overview.html
2. DPI 编程指南：https://eco.dameng.com/document/dm/zh-cn/pm/dpi-rogramming-guide.html
3. DM ODBC 编程指南：https://eco.dameng.com/document/dm/zh-cn/pm/odbc-rogramming-guide.html
4. DM JDBC 编程指南：https://eco.dameng.com/document/dm/zh-cn/pm/jdbc-rogramming-guide.html
5. .NET Data Provider 编程指南：https://eco.dameng.com/document/dm/zh-cn/pm/net-rogramming-guide.html
6. DM PHP 编程指南：https://eco.dameng.com/document/dm/zh-cn/pm/php-rogramming-guide.html
7. DM FLDR 编程指南：https://eco.dameng.com/document/dm/zh-cn/pm/fldr-rogramming-guide.html
8. DM DEXP/DIMP JNI 编程指南：https://eco.dameng.com/document/dm/zh-cn/pm/dexp-dimp-jni-rogramming-guide.html
9. Logmnr 接口使用说明：https://eco.dameng.com/document/dm/zh-cn/pm/logmnr-interface-instructions.html
10. DM Node.js 编程指南：https://eco.dameng.com/document/dm/zh-cn/pm/nodejs-rogramming-guide.html
11. DM Go 编程指南：https://eco.dameng.com/document/dm/zh-cn/pm/go-rogramming-guide.html
12. DM XA 编程指南：https://eco.dameng.com/document/dm/zh-cn/pm/xa-rogramming-guide.html
13. DM R2DBC 编程指南：https://eco.dameng.com/document/dm/zh-cn/pm/r2dbc-rogramming-guide.html

## 15. 已知不完整 / 待补充

- **性能基准数据**：DPI "比 ODBC 快 3-10 倍" 来自官方说明，但未提供具体测试环境（TPS、并发数、硬件）。如需准确选型数据，应在生产硬件上自跑 benchmark。
- **PROC 编程接口**：本手册未覆盖（属 PROC 使用手册）；嵌入式 SQL 通过 DM 预编译器生成 C 代码。
- **dmPython / dmDjango / dmSQLAlchemy**：本手册未列出（见 dmPython 使用手册）。
- **PHP 8.x 全部 53 个扩展函数完整签名**：本节只列了 5.x/7.x/8.x 的分类与变更情况，每个函数详细参数请查官方手册。
- **JDBC 4.0 Annotations 完整 API**：本节未展开 `@QueryResult` 等注解用法。
- **XA 完整 INI 调优矩阵**：XA_COMPATIBLE_MODE 三个值的边界场景（ORACLE 19c/20c 新行为）建议查 `DM XA 编程指南` 12.2.3 与 `DM8 系统包使用手册 - DBMS_XA`。
- **R2DBC 性能对比 R2DBC vs JDBC**：未提供官方基准，仅说"基于 Reactor 的非阻塞"。

> ⚠️ 以上"待补充"项标注"⚠️ 待官方手册确认"——内容如需精确落地，请以对应官方手册原文为准。
