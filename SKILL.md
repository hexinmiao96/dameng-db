---
name: dameng-db
description: >-
  达梦数据库技能包（DM8）。覆盖达梦安装部署、表空间/用户管理、Oracle/MySQL/PostgreSQL/SQL Server/DB2→达梦迁移、SQL/DDL 改写、Java/Spring/MyBatis/MyBatis-Plus/Hibernate 适配、PL/SQL、系统包、性能优化、运维监控和集群架构。
  Use when: (1) 安装或配置达梦数据库, (2) 从其他数据库迁移到达梦, (3) 改造达梦兼容 SQL/MyBatis XML/Java 数据源, (4) 处理达梦迁移后的保留字、表别名、GROUP、IDENTITY、自增 ID、字符集、Excel 导入回归等问题, (5) 进行达梦 SQL 开发、性能调优、运维管理或集群方案核对。
---

# 达梦数据库技能包（DM8）

国产自研关系型数据库，兼容 Oracle/MySQL/SQL Server 语法，支持 Windows/Linux/信创平台。

> **v2 增强**（2026-06-10）：基于达梦官方文档 eco.dameng.com 梳理，并持续补充迁移复盘规则。新增 Java 生态、PL/SQL、SQL 高级技巧、系统包速查、编程接口、其他开发框架等深度参考章节，原应用开发、迁移、集群章节也已扩展。

## 技能包定位

`dameng-db` 是达梦数据库任务的通用技能包，迁移到达梦是其中的核心场景。迁移任务不要只做 SQL 文本替换，要把源库差异、达梦初始化参数、Java/MyBatis 适配、数据迁移校验和迁移后问题复盘串成闭环。

迁移工作按三类 reference 分工：`migration.md` 负责迁移流程、SQL 改写和校验清单；`app-dev-java.md` 负责 Java/Spring/MyBatis/MyBatis-Plus 适配；`post-migration-triage.md` 负责迁移后问题台账研判，区分达梦系统性风险和非达梦迁移问题。`migration-faq.md` 用于补充现场报错和源库差异。

发布或交付迁移结论时，必须说明哪些问题已经沉淀为达梦迁移规则，哪些只是非达梦迁移问题。`batchInsert` 主键列、Excel 公式/空行过滤等可复用迁移风险可以写入复盘；纯项目问题只保留排除结论，不纳入通用达梦技能。

## 使用入口

先明确任务类型，再只读取相关 reference，避免把资料当成结论：

1. 安装/部署/容器化：读 `installation.md`、`ops.md`；需要用户/表空间再读 `management.md`。
2. MySQL/Oracle/PG/SQL Server/DB2 迁移：先读 `migration.md`；Java/MyBatis 项目再读 `app-dev-java.md`。
3. Java/Spring/MyBatis/MyBatis-Plus/Hibernate 适配：先读 `app-dev-java.md`；跨语言或其他框架再读 `app-dev-frameworks.md`。
4. SQL、PL/SQL、系统包：分别读 `sql-dev.md`、`plsql.md`、`system-packages.md`。
5. 性能、监控、集群：分别读 `performance.md`、`monitoring.md`、`clusters.md`。
6. 迁移后问题清单、缺陷台账、上线回归复盘：先读 `post-migration-triage.md`，再按问题类型补读 `migration.md` 或 `app-dev-java.md`。

开始执行前先确认边界：目标环境（本地/测试/生产）、是否允许写库、源库是否只读、达梦版本、兼容模式、驱动版本、项目框架、可用验证方式。没有明确授权时，不要对源库或生产库执行写操作。

迁移类任务默认成功标准：静态扫描有记录，改动只覆盖迁移相关点，每个 mapper XML 都完成“改一个 → 比一个 → 记一个”，可连达梦时至少验证核心 SQL 可执行，最终给出残留风险和待业务确认项。

## 快速参考索引

根据任务类型选择对应的 reference 文件：

| 任务 | 文件 | 说明 |
|------|------|------|
| 了解版本区别、获取安装包 | [versions.md](references/versions.md) | 版本线索与下载入口 |
| 安装部署数据库（Windows/Linux/信创） | [installation.md](references/installation.md) | 初始化参数、服务注册、常见安装问题 |
| 表空间、用户、模式管理 | [management.md](references/management.md) | 表空间、用户、权限、模式 |
| 图形化工具（DM Manager/DISQL/DTS/SQLark） | [tools.md](references/tools.md) | 管理工具、迁移工具、命令行工具 |
| **数据迁移 Oracle/MySQL/PG/SQL Server/DB2 → DM** | [migration.md](references/migration.md) | 迁移流程、SQL 改写、校验清单 |
| 迁移常见问题 100+ FAQ | [migration-faq.md](references/migration-faq.md) | 源库差异与典型报错处理 |
| 迁移后问题清单研判、缺陷复盘、系统性风险扫描 | [post-migration-triage.md](references/post-migration-triage.md) | 问题分类、表别名、`GROUP BY`、MyBatis `<if>` 等复盘规则 |
| **应用开发总览（多语言）** | [app-dev.md](references/app-dev.md) | 多语言驱动与框架入口 |
| **Java 生态深度实战（JDBC/MyBatis/MyBatis-Plus/Spring/JPA/Sharding-JDBC）** | [app-dev-java.md](references/app-dev-java.md) | Java 栈配置、XML 迁移、上线检查 |
| **其他开发框架（Hibernate/iBatis/MyCat/Django/Flask/GORM/R2DBC/...）** | [app-dev-frameworks.md](references/app-dev-frameworks.md) | 非 Java 或跨框架适配 |
| **PL/SQL 程序设计（块结构/存储过程/触发器/包/游标/异常/动态 SQL）** | [plsql.md](references/plsql.md) | 过程化 SQL 与触发器 |
| **SQL 高级技巧（字符串/数字/日期/范围/闪回/物化视图/层次查询/读写分离）** | [sql-dev.md](references/sql-dev.md) | SQL 写法、函数、查询技巧 |
| **52+ 系统包速查（DBMS_*/UTL_*/DMSQL 编程）** | [system-packages.md](references/system-packages.md) | 系统包能力速查 |
| **编程接口与底层 API（DPI/ODBC/JDBC/Node.js/Go/XA/Logmnr/R2DBC/FLDR）** | [programmer-guide.md](references/programmer-guide.md) | 底层接口与编程 API |
| 运维管理（环境准备、备份、归档、日志、安全） | [ops.md](references/ops.md) | 运维基线、备份恢复、安全 |
| 性能优化（参数调优、SQL 优化、执行计划、统计信息） | [performance.md](references/performance.md) | 参数、执行计划、统计信息 |
| 监控工具（DEM/DMLOG/NMON/Prometheus） | [monitoring.md](references/monitoring.md) | 监控与告警 |
| **集群架构（DataWatch/DSC/RWC/DPC/K8s）** | [clusters.md](references/clusters.md) | 高可用、读写分离、容器化 |

## 核心原则

### 安装部署

1. **严禁 root 安装**，必须创建 dmdba 用户
2. **不可变参数**需慎重：PAGE_SIZE、EXTENT_SIZE、CASE_SENSITIVE、CHARSET、BLANK_PAD_MODE
3. 生产环境推荐：页大小 32K、簇大小 32、日志 2048M
4. 安装后必须用 root 执行 `root_installer.sh` 注册服务
5. **信创平台**（麒麟/统信）需额外做 OS 适配；生产环境按现场基线做最小端口放行和内核参数调整，不默认关闭防火墙

### 迁移

1. 迁移前必须确认 `COMPATIBLE_MODE`（0=不兼容，1=SQL92，2=Oracle，3=SQL Server，4=MySQL，5=DM6，6=Teradata，7=PostgreSQL，8=DB2；不同 DM 小版本以 `V$DM_INI`/官方手册为准）
2. MySQL→DM 注意字符 vs 字节差异，varchar 映射为 `varchar(N char)`
3. Java/MyBatis 项目先做静态扫描：分页、自增、函数、布尔字面量、时间函数、upsert、大小写、关键字
4. DTS 是静态迁移工具，迁移中源库不能有变更；需要连源库时默认使用只读账号或只读事务
5. **PG→DM**：数组类型用 CLOB+JSON 替代；DB2→DM：FETCH FIRST → LIMIT
6. MyBatis XML 必须逐文件迁移、逐文件比对验证：每完成一个 `.xml`，立即对比原 SQL/新 SQL、扫描残留 MySQL 方言，并记录验证结果后再进入下一个文件
7. 迁移流程：评估→准备→实施→逐 XML 比对→整体校验→统计信息→备份→应用回归
8. 通用原则：先 PoC 1-2 个非核心业务验证全链路，再批量迁移
9. 执行导入、转换、清理、DDL、DML 前必须说清楚将写入哪个数据库；用户只要求“本地导入/本地测试”时，不允许写源库
10. 迁移后问题复盘要先分类：保留字、表别名/同名字段、`GROUP BY`、`CHAR` 补空格、`IDENTITY` 插入 null、`<if>` 空串条件等写入系统性规则；非达梦迁移问题只记录排除结论，不写入通用技能

### 运维

1. 开启归档才能备份
2. 磁盘建议分 3 块：数据盘、备份盘、归档盘
3. 生产环境优先最小端口放行；测试环境可临时关闭防火墙定位问题；SELinux/swap/透明大页需按厂商建议和现场基线评估
4. limits.conf 配 nofile 65536, nproc 65536
5. XFS 文件系统优于 ext4，磁盘调度用 deadline
6. **三权分立**：SYSDBA / SYSSSO / SYSAUDITOR 分离职责

### 开发

1. 驱动版本必须与数据库服务器版本一致
2. **JDBC**：`jdbc:dm://host:5236`，驱动类 `dm.jdbc.driver.DmDriver`
3. **MyBatis-Plus** ≥ 3.0 支持达梦，IdType.AUTO 对应 IDENTITY 列
4. **Spring Boot**：Druid 数据源 + MyBatis-Plus 自动配置
5. **Hibernate**：方言用 `org.hibernate.dialect.DmDialect`
6. **Python**：dmPython（DPI 接口），`?` 占位符
7. **Go**：GORM（V1 dmgorm1 / V2 dmgorm2）+ ODBC + 原生 go_dm 三选一
8. 端口默认 5236

### SQL 编程

1. `||` 字符串拼接（默认 Oracle 风格）
2. 伪列 `ROWNUM` / `ROWID`（ROWNUM > N 是无效查询）
3. `MERGE INTO` upsert 标准（SQL:2003）
4. `CONNECT BY` 层次查询（兼容 Oracle）
5. `dm.ini` 的 `COMPATIBLE_MODE` 与 JDBC URL 的 `compatibleMode=mysql/oracle` 是两层设置：前者影响数据库实例解析/行为且多为静态参数，后者影响驱动会话兼容；不要混用取值

### 性能调优

1. 读懂 EXPLAIN 执行计划（CSCN/SSC/NL/HASH JOIN）
2. 索引选择度 ≥ 95% 才有意义
3. 深分页（OFFSET 大值）改 `ROW_NUMBER()` 锚点分页
4. 关键参数：BUFFER POOL、SORT_BUF、HASH_AREA_SIZE、WORK_THREADS
5. 定期 `DBMS_STATS.GATHER_SCHEMA_STATS` 收集统计信息

## v2 增强变更日志

### 相比 v1 增量

- **新增 6 个深度参考章节**：
  - `app-dev-java.md`（4 千字，Java 全栈实战）
  - `plsql.md`（3.4 千字，PL/SQL 独立章节）
  - `sql-dev.md`（2.9 千字，SQL 高级技巧）
  - `system-packages.md`（3.6 千字，52+ 系统包速查）
  - `app-dev-frameworks.md`（2.2 千字，13 框架适配）
  - `programmer-guide.md`（2.2 千字，12 编程接口）
- **扩展 3 个原参考章节**：
  - `app-dev.md` 1373→2246（+SQLark/Django/Flask 实战）
  - `migration.md` 2704→4287（+SQLark/PG/SQLServer/DB2 迁移）
  - `clusters.md` 1962→3007（+DMDPC 分布式 / MPP+DW 二级 HA / K8s）
- **内容规模**：由 v1 的基础指南扩展为覆盖迁移、开发、SQL、运维、集群的多章节知识库
- **代码示例**：补充大量 SQL、Java、配置和命令行示例，后续维护不再依赖精确字数统计

### 覆盖度

- 达梦官方文档 6 大模块（快速上手/运维/应用开发/SQL 开发/FAQ/产品手册）核心内容覆盖
- 引用 100+ 官方 URL（eco.dameng.com）
- 仍待补充：DM9 新版本特性、>1TB 大库并行 DTS、K8s Operator 具体仓库、Zabbix/Prometheus 集成模板

## 官方权威源

所有内容基于达梦官方文档（https://eco.dameng.com/document/dm/zh-cn/）撰写。
