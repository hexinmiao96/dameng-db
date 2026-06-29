# 达梦数据库运维监控工具

> 涵盖 DEM、DMLOG、NMON、Prometheus 集成及关键监控指标。

---

## 目录

- [DEM（达梦企业管理器）](#dem达梦企业管理器)
  - [架构概览](#架构概览)
  - [部署步骤](#部署步骤)
  - [监控能力](#监控能力)
  - [注意事项](#注意事项)
- [DMLOG（SQL 日志分析工具）](#dmlogsql-日志分析工具)
- [NMON（Linux 系统监控）](#nmonlinux-系统监控)
- [Prometheus 集成](#prometheus-集成)
- [监控关键指标](#监控关键指标)
- [日常巡检清单](#日常巡检清单)

---

## DEM（达梦企业管理器）

DEM（Dameng Enterprise Manager）是达梦提供的 Web 化集中监控管理平台，可用于同时管理多套 DM 数据库实例。

### 架构概览

```
┌──────────────────────────────────────────────────────┐
│                    DEM Web 界面                       │
│              http://dem_host:8080/dem                 │
└──────────────────────┬───────────────────────────────┘
                       │
              ┌────────┴────────┐
              │  Tomcat 容器    │
              │  (DEM Server)   │
              └────────┬────────┘
                       │
         ┌─────────────┼─────────────┐
         │             │             │
    ┌────┴────┐   ┌────┴────┐   ┌────┴────┐
    │ dmagent │   │ dmagent │   │ dmagent │
    │ (主机A)  │   │ (主机B)  │   │ (主机C)  │
    └────┬────┘   └────┬────┘   └────┬────┘
         │             │             │
    ┌────┴────┐   ┌────┴────┐   ┌────┴────┐
    │ DM实例1  │   │ DM实例2 │   │ DM实例3 │
    │ DM实例2  │   └─────────┘   └─────────┘
    └─────────┘
```

**核心组件：**

| 组件 | 说明 | 部署位置 |
|------|------|---------|
| **DEM Server** | Web 管理应用，部署在 Tomcat 中 | 管理服务器 |
| **DEM 存储库** | 存储监控数据、配置的 DM 数据库 | 管理服务器或独立服务器 |
| **dmagent** | 部署在每台目标主机上的采集代理 | 每台数据库主机 |
| **目标数据库** | 被监控的 DM 实例 | 各数据库服务器 |

### 部署步骤

#### 第一步：环境准备

```bash
# 安装 Java 1.8（JDK 8）
yum install -y java-1.8.0-openjdk java-1.8.0-openjdk-devel

# 验证 Java 版本
java -version
# 期望输出：java version "1.8.0_xxx"

# 配置 JAVA_HOME 环境变量
echo 'export JAVA_HOME=/usr/lib/jvm/java-1.8.0-openjdk' >> ~/.bash_profile
echo 'export PATH=$JAVA_HOME/bin:$PATH' >> ~/.bash_profile
source ~/.bash_profile

# 安装 Tomcat（以 8.5.x 为例）
cd /opt
wget https://archive.apache.org/dist/tomcat/tomcat-8/v8.5.100/bin/apache-tomcat-8.5.100.tar.gz
tar -xzf apache-tomcat-8.5.100.tar.gz
mv apache-tomcat-8.5.100 tomcat-dem

# 确保 JAVA_HOME 正确
/opt/tomcat-dem/bin/version.sh
```

#### 第二步：创建 DEM 后台数据库

```sql
-- 在已经运行的 DM 实例上创建 DEM 存储库
-- 创建表空间
CREATE TABLESPACE "DEM_DATA"
  DATAFILE '/data/dmdata/DAMENG/DEM_DATA.DBF'
  SIZE 256
  AUTOEXTEND ON NEXT 100 MAXSIZE 10240;

-- 创建 DEM 管理用户
CREATE USER "DEM" IDENTIFIED BY "DemAdmin@2024"
  DEFAULT TABLESPACE "DEM_DATA";

-- 给 DEM 用户授权
GRANT "DBA" TO "DEM";   -- DEM 需要较高权限来采集监控数据
```

**执行初始化脚本：**

```bash
# dem_init.sql 位于 DM 安装目录的 web 子目录
# 在 DEM 用户下执行初始化脚本
/opt/dmdbms/bin/disql DEM/DemAdmin@2024@localhost:5236 \` /opt/dmdbms/web/dem_init.sql
```

> 该脚本会自动创建 DEM 所需的元数据表（约 200+ 张表）和存储过程。

#### 第三步：配置 DEM Server

**配置 db.xml（数据库连接）：**

编辑 `$TOMCAT_HOME/webapps/dem/WEB-INF/db.xml`：

```xml
<?xml version="1.0" encoding="UTF-8"?>
<ConnectPool>
    <Server>192.0.2.100</Server>
    <Port>5236</Port>
    <User>DEM</User>
    <Password>DemAdmin@2024</Password>
    <InitPoolSize>5</InitPoolSize>
    <MaxPoolSize>50</MaxPoolSize>
    <Database>DAMENG</Database>
</ConnectPool>
```

**配置 log4j.xml（日志）：**

编辑 `$TOMCAT_HOME/webapps/dem/WEB-INF/log4j.xml`，按需调整日志级别和输出路径。

#### 第四步：部署 dmagent

在**每台**被监控的主机上部署 dmagent：

```bash
# 1. 复制 dmagent 目录到目标主机
scp -r /opt/dmdbms/dmagent dm_host_2:/opt/dmdbms/

# 2. 在目标主机配置 dmagent
cd /opt/dmdbms/dmagent

# 编辑 agent.ini
vi agent.ini
```

**agent.ini 关键配置：**

```ini
[Agent]
# DEM Server 的 URL
center_url = http://192.0.2.100:8080/dem

# 本机 IP（多个 IP 逗号分隔）
ip_list = 192.168.1.101

# dmagent 监听端口
agent_listen_port = 8636

# 日志级别
log_level = info
```

```bash
# 3. 启动 dmagent
cd /opt/dmdbms/dmagent
./start.sh

# 4. 验证 dmagent 状态
./status.sh
# 期望：dmagent is running.
```

> **安全建议：** 为 DEM 创建独立的监控用户，不要直接使用 SYSDBA：
> ```sql
> CREATE USER "DEM_MONITOR" IDENTIFIED BY "Monitor@2024";
> GRANT "VTI", "SOI", "SVI", "RESOURCE" TO "DEM_MONITOR";
> ```

#### 第五步：启动并访问 DEM

```bash
# 启动 Tomcat
/opt/tomcat-dem/bin/startup.sh

# 查看启动日志
tail -f /opt/tomcat-dem/logs/catalina.out

# 确认 Tomcat 已启动
ps -ef | grep tomcat
```

**访问 DEM Web 界面：**

```
http://192.0.2.100:8080/dem
默认用户：admin；默认密码以安装配置为准
```

首次登录后需修改密码。

#### 在 DEM 中添加被监控主机

1. 登录 DEM Web 界面
2. 进入"主机管理" → "添加主机"
3. 输入目标主机 IP 和 dmagent 端口（默认 8636）
4. DEM 自动发现该主机上的 DM 实例
5. 将发现的实例添加到监控列表

### 监控能力

#### 系统资源监控

| 监控项 | 指标 | 用途 |
|--------|------|------|
| **CPU** | 使用率（用户态/内核态/IO等待） | 判断 CPU 是否成为瓶颈 |
| **内存** | 总内存/已用/可用/Swap | 判断内存是否充足 |
| **磁盘 IO** | FIO（文件IO）/ NIO（网络IO） | 判断磁盘读写性能 |
| **IO AWAIT** | 平均等待时间 | 判断磁盘繁忙程度 |
| **网络** | 发送/接收 流量和包数 | 判断网络是否拥塞 |

#### 负载统计

DEM 提供多时间周期的历史负载折线图：

| 时间范围 | 用途 |
|---------|------|
| **6 小时** | 近期异常排查 |
| **1 天** | 日间峰谷分析 |
| **1 周** | 业务周期趋势 |
| **1 月** | 容量规划基线 |
| **1 年** | 长期增长趋势 |

#### 数据库监控

| 类别 | 监控指标 |
|------|---------|
| **性能指标** | TPS（事务/秒）、QPS（查询/秒） |
| **会话** | 活动会话数、等待会话、长事务会话 |
| **死锁** | 死锁检测与告警 |
| **事务** | 长事务告警（超过阈值的未提交事务） |
| **SQL 分析** | TOP SQL（按执行时间/执行次数/逻辑读排序） |
| **表空间** | 使用率、剩余空间、自动扩展状态 |
| **归档日志** | 归档状态、归档空间使用率 |
| **事件** | 数据库启动/停止、检查点、备份完成等 |

#### 告警功能

- **邮件告警：** 配置 SMTP 服务器，自定义告警接收人
- **短信告警：** 通过短信网关发送（需额外配置）
- **自定义告警规则：** 可设置阈值和触发条件
  - CPU > 90% 持续 5 分钟
  - 表空间使用率 > 85%
  - 活动会话数 > 500
  - 死锁发生
  - 归档空间不足

### 注意事项

1. **版本一致性：** DEM Server 和所有 dmagent 必须使用完全相同的版本号，否则可能出现数据采集异常或界面显示错误。

2. **时间同步：** 所有被监控节点必须与 DEM Server 时间同步（建议配置 NTP）：
   ```bash
   # 配置 NTP 时间同步
   yum install -y ntp
   systemctl enable ntpd
   systemctl start ntpd
   ntpq -p  # 查看同步状态
   ```

3. **独立监控用户：** 不建议直接使用 SYSDBA 连接被监控数据库，应创建专门的监控用户，遵循最小权限原则：
   ```sql
   CREATE USER "DEM_MONITOR" IDENTIFIED BY "Monitor@2024";
   GRANT "VTI", "SOI", "SVI" TO "DEM_MONITOR";  -- 只给查询权限，不给写入
   ```

4. **定期清理历史数据：** DEM 存储库会持续积累监控数据，建议配置定时清理任务：
   ```sql
   -- 示例：删除 6 个月前的监控数据
   DELETE FROM DEM_METRIC_DATA WHERE COLLECT_TIME < ADD_MONTHS(SYSDATE, -6);
   COMMIT;
   ```
   建议将此操作配置为数据库定时作业。

5. **DEM 自身高可用：** 如果 DEM 是生产环境的关键组件，建议：
   - 使用独立数据库作为 DEM 存储库
   - 定期备份 DEM 存储库
   - 考虑 DEM Server 的高可用部署

---

## DMLOG（SQL 日志分析工具）

DMLOG 是达梦自带的 SQL 日志分析工具，可解析数据库 SQL 日志文件，生成统计报表。

### 功能特性

- 统计 SQL 执行频次和耗时排名
- 按最长执行时间和最高频次双维度排序
- 生成 Excel 数据报表（便于进一步分析）
- 生成 ECharts 散点图（可视化 SQL 分布）
- 生成 QPS 折线图（时间维度查询量变化）
- 支持单个或多个日志文件分析

### 使用要求

| 要求 | 说明 |
|------|------|
| Java 版本 | JDK 1.8 或以上 |
| 使用环境 | **仅限测试/开发环境**（生产环境直接运行可能影响性能） |
| 输入文件 | 达梦 SQL 日志文件（需预先开启 SQL 日志功能） |
| 输出格式 | Excel（.xls）、HTML 图表 |

### 开启 SQL 日志

在分析之前，需要先开启数据库的 SQL 日志功能：

```sql
-- 在 dm.ini 中开启 SQL 日志
ALTER SYSTEM SET 'SQL_TRACE_MASK' = 1 BOTH;       -- 开启 SQL 日志
ALTER SYSTEM SET 'SVR_LOG' = 1 BOTH;               -- 开启服务器日志
ALTER SYSTEM SET 'SVR_LOG_FILE_NUM' = 10 BOTH;     -- 日志文件个数
ALTER SYSTEM SET 'SVR_LOG_FILE_SIZE' = 100 BOTH;   -- 每个日志文件大小（MB）

-- 或者修改 sqllog.ini 文件，指定需要记录的 SQL 类型
-- SQL 日志文件默认位置：$DM_HOME/log/
-- 文件名格式：dmsql_实例名_日期_时间.log
```

也可以通过 `sqllog.ini` 文件进行更精细的过滤配置。

### 使用步骤

#### 第一步：配置

编辑 `dmlog.properties` 配置文件：

```properties
# SQL 日志文件目录（支持多个，逗号分隔）
LOG_PATH=/opt/dmdbms/log
# 或指定具体文件
# LOG_PATH=/opt/dmdbms/log/dmsql_DMSERVER_2024_01_01_10_00_00.log

# 输出目录
RESULT_PATH=/opt/dmlog/result

# Excel 名称
EXCEL_NAME=DMLOG_分析报告

# 散点图采样模式：0=全部，1=采样前50条，2=采样前100条
CHART_MODE=1

# SQL 摘要提取的字节数
SQL_BYTE_NUM=200
```

#### 第二步：执行分析

```bash
# 确保 Java 可用
java -version

# 执行分析
java -jar DMLOG.jar
# 或指定配置文件路径
java -jar DMLOG.jar -c /opt/dmlog/dmlog.properties
```

#### 第三步：查看结果

分析完成后，会在 `RESULT_PATH` 指定的目录下生成 `RESULT` 文件夹，包含：

```
RESULT/
├── EXCEL_NAME.xls         # Excel 统计报表
├── scatter_chart.html     # ECharts 散点图
├── qps_chart.html         # QPS 折线图
└── summary.txt            # 汇总统计文本
```

**Excel 报表包含的统计维度：**

| 统计项 | 说明 |
|--------|------|
| SQL 文本（截取） | 前 N 字节的 SQL 语句摘要 |
| 执行次数 | 该 SQL 的总执行次数 |
| 执行次数占比 | 该 SQL 占所有 SQL 的比例 |
| 最长执行时间 | 该 SQL 单次最长耗时 |
| 平均执行时间 | 该 SQL 平均耗时 |
| 总执行时间 | 累计总耗时 |
| 首次执行时间 | 日志中首次出现的时间 |
| 末次执行时间 | 日志中最后出现的时间 |

### 注意事项

1. **单页 Excel 最大行数限制：** 65536 行。如果分析结果超过此限制，超出的数据不会写入 Excel（但 summary.txt 仍会包含完整统计）。

2. **仅限测试环境：** 在生产环境分析 SQL 日志可能占用大量 CPU 和内存，建议将日志文件复制到测试/分析环境后执行。

3. **日志文件大小：** 如果 SQL 日志文件非常大（数十 GB），分析可能需要较长时间并有较高的内存需求，建议通过 `sqllog.ini` 过滤不需要记录的 SQL 类型。

4. **字符编码：** 确保日志文件和分析工具使用相同的字符编码（GB18030 或 UTF-8），否则可能出现中文乱码。

---

## NMON（Linux 系统监控）

NMON（Nigel's Performance Monitor）是一个强大的 Linux 系统性能监控工具，广泛用于数据库服务器的系统层性能数据采集。

### 部署

```bash
# 下载 NMON（根据系统架构选择对应版本）
# RHEL/CentOS 7 x86_64:
wget http://nmon.sourceforge.net/pmwiki.php?n=Site.Download -O /tmp/nmon.tar.gz

# 或直接使用 DM 安装包中自带的版本
cp /opt/setup/nmon_x86_64_rhel7 /usr/local/bin/nmon
# 或者
tar xvfz /opt/setup/nmon.tar.gz -C /usr/local/bin/

# 赋予执行权限
chmod 755 /usr/local/bin/nmon
# 或 chmod 777 /usr/local/bin/nmon
```

### 实时交互监控

```bash
# 启动实时交互模式
/usr/local/bin/nmon

# 交互快捷键：
# c = 显示 CPU 详情
# m = 显示内存详情
# d = 显示磁盘 IO
# n = 显示网络流量
# t = 显示 TOP 进程
# q = 退出
```

### 后台批量采集

#### 推荐采集命令

```bash
# -f: 输出到文件（自动命名包含时间戳）
# -s 30: 每 30 秒采集一次
# -c 60: 共采集 60 次（总计 30 分钟）
# -m /data/nmon/: 输出到指定目录
/usr/local/bin/nmon -f -s 30 -c 60 -m /data/nmon/

# 根据业务峰谷灵活调整采集参数
# 长时间监控（24小时，每分钟一次）
/usr/local/bin/nmon -f -s 60 -c 1440 -m /data/nmon/

# 短时高频（每秒一次，共 120 次=2分钟）
/usr/local/bin/nmon -f -s 1 -c 120 -m /data/nmon/
```

**输出文件命名格式：** `hostname_YYYYMMDD_HHMM.nmon`

### 数据分析

#### 使用 nmon_analyzer 生成 Excel 报表

```bash
# 下载 nmon_analyzer（Excel 宏工具）
# 需要 WPS Office 或 Microsoft Excel 打开

# 或使用命令行工具 nmonchart
# 将 .nmon 文件拖入 nmon_analyzer.xlsm，自动生成图表
```

#### 重点关注的性能指标

| 标签页 | 指标 | 正常范围 | 告警阈值 |
|--------|------|---------|---------|
| **SYS_SUMM** | 系统总览 | — | CPU > 90%, IO Wait > 10% |
| **CPU_ALL** | 总 CPU 使用率 | < 60% | > 80%（持续） |
| **CPU_ALL** | IO Wait | < 5% | > 10%（磁盘瓶颈） |
| **MEM** | 内存使用率 | < 80% | > 90% |
| **DISKBUSY** | 磁盘繁忙度 | < 50% | > 80%（严重 IO 瓶颈） |
| **DISKREAD/DISKWRITE** | 磁盘读写速率 | 视存储而定 | 接近磁盘标称上限 |

**各指标的诊断意义：**

- **CPU_ALL 高但 IO Wait 低** → CPU 瓶颈，需优化 SQL 或扩容 CPU
- **CPU_ALL 不高但 IO Wait 高** → 磁盘 IO 瓶颈，考虑 SSD/SAS 或优化 SQL 减少 IO
- **DISKBUSY 持续 > 80%** → 磁盘子系统严重过载，急需扩容或分离读写
- **MEM 中 Free 持续减少** → 内存压力，需增加物理内存或调整数据库缓冲区参数

### 定时采集建议

```bash
# 添加 crontab 定时任务（每天业务高峰期采集）
# 例如：每天 9:00-10:00 和 14:00-15:00 各采集一次
# 编辑 crontab
crontab -e

# 添加以下行
0 9 * * 1-5 /usr/local/bin/nmon -f -s 30 -c 120 -m /data/nmon/morning/
0 14 * * 1-5 /usr/local/bin/nmon -f -s 30 -c 120 -m /data/nmon/afternoon/

# 定期清理旧文件（保留 7 天）
0 1 * * * find /data/nmon/ -name "*.nmon" -mtime +7 -delete
```

---

## Prometheus 集成

DEM 提供了 Prometheus 兼容的 metrics 端点，可直接接入 Prometheus + Grafana 监控体系。

### 配置步骤

#### 第一步：确认 DEM 版本支持

- DEM 4.0 及以上版本支持 Prometheus 集成

#### 第二步：在 Prometheus 中配置抓取

编辑 `prometheus.yml`：

```yaml
global:
  scrape_interval: 15s     # 抓取间隔
  evaluation_interval: 15s # 规则评估间隔

scrape_configs:
  - job_name: 'dm_dem'
    static_configs:
      - targets:
        - '192.0.2.100:8080'   # DEM Server 地址
    metrics_path: '/dem/metrics' # DEM metrics 端点
    scheme: 'http'
    # 如果 DEM 需要认证
    # basic_auth:
    #   username: 'admin'
    #   password: '******'
```

#### 第三步：重启 Prometheus

```bash
# 验证配置文件
promtool check config /etc/prometheus/prometheus.yml

# 重启 Prometheus
systemctl restart prometheus

# 验证 metrics 端点可达
curl http://192.0.2.100:8080/dem/metrics
```

#### 第四步：Grafana 可视化

```bash
# 在 Grafana 中添加 Prometheus 数据源
# 导入或创建 DM 监控 Dashboard
```

**可用的 Metrics 指标（示例）：**

```
# 数据库状态
dm_instance_status

# 会话数
dm_active_sessions
dm_total_sessions

# TPS/QPS
dm_transactions_per_sec
dm_queries_per_sec

# 表空间
dm_tablespace_total_bytes
dm_tablespace_used_bytes
dm_tablespace_usage_percent

# 死锁
dm_deadlock_count

# 缓冲池
dm_buffer_pool_hit_ratio
dm_buffer_pool_total_pages
dm_buffer_pool_free_pages
```

---

## 监控关键指标

### 数据库层面

#### 1. 数据库状态

```sql
-- 查看实例运行状态
SELECT STATUS$ FROM V$INSTANCE;

-- 状态值说明：
-- OPEN：正常打开
-- MOUNT：已挂载但未打开
-- STARTUP：启动中
-- SUSPEND：挂起
```

#### 2. 表空间使用率

```sql
-- 精确计算表空间使用率（%）
SELECT
  NAME AS 表空间,
  TYPE$ AS 类型,
  CAST(TOTAL_SIZE AS DECIMAL)/1024 AS 总大小MB,
  CAST(FREE_SIZE AS DECIMAL)/1024 AS 剩余大小MB,
  CAST((TOTAL_SIZE - FREE_SIZE) AS DECIMAL)/1024 AS 已用MB,
  ROUND(CAST((TOTAL_SIZE - FREE_SIZE) AS DECIMAL)/CAST(TOTAL_SIZE AS DECIMAL)*100, 2) AS 使用率
FROM V$TABLESPACE
ORDER BY 使用率 DESC;
```

#### 3. 会话监控

```sql
-- 活跃会话数
SELECT COUNT(*) AS 活跃会话数
FROM V$SESSIONS
WHERE STATE = 'ACTIVE';

-- 按会话状态统计
SELECT
  STATE AS 状态,
  COUNT(*) AS 数量
FROM V$SESSIONS
GROUP BY STATE;

-- 长事务会话（超过 300 秒未提交）
SELECT
  SESS_ID AS 会话ID,
  USER_NAME AS 用户名,
  CLNT_IP AS 客户端IP,
  APPNAME AS 应用名,
  STATE AS 状态,
  (SYSDATE - SF_GET_SESSION_PARA_VALUE(SESS_ID, 'TRX_START_TIME')) * 86400 AS 事务持续秒数
FROM V$SESSIONS
WHERE STATE = 'ACTIVE'
  AND (SYSDATE - SF_GET_SESSION_PARA_VALUE(SESS_ID, 'TRX_START_TIME')) * 86400 > 300;

-- 查看会话执行的 SQL
SELECT
  S.SESS_ID,
  S.USER_NAME,
  S.SQL_TEXT,
  S.STATE,
  S.LAST_SEND_TIME
FROM V$SESSIONS S
WHERE S.STATE = 'ACTIVE';
```

#### 4. 锁等待

```sql
-- 查看锁等待链
SELECT
  L.LTYPE AS 锁类型,
  L.BLOCKED AS 是否被阻塞,
  O.NAME AS 对象名,
  S.SESS_ID AS 会话ID,
  S.USER_NAME AS 用户名,
  S.CLNT_IP AS 客户端IP
FROM V$LOCK L
LEFT JOIN V$SESSIONS S ON L.TRX_ID = S.TRX_ID
LEFT JOIN SYSOBJECTS O ON L.TABLE_ID = O.ID
WHERE L.BLOCKED = 1;

-- 查看死锁历史
SELECT * FROM V$DEADLOCK_HISTORY;
```

#### 5. SQL 性能分析

```sql
-- 通过动态视图分析 SQL 执行情况
SELECT
  TRX_ID,
  SESS_ID,
  SQL_TEXT,
  EXEC_TIME,          -- 执行时间（秒）
  ROW_COUNT,          -- 影响行数
  N_LOGIC_READS,      -- 逻辑读次数
  N_PHY_READS,        -- 物理读次数
  START_TIME
FROM V$SQL_HISTORY
WHERE EXEC_TIME > 1   -- 过滤执行超过 1 秒的 SQL
ORDER BY EXEC_TIME DESC;

-- 或者通过 DMLOG 分析 SQL 日志文件获取更全面的统计分析
```

#### 6. 缓冲池命中率

```sql
-- 查看缓冲池命中率
SELECT
  NAME AS 缓冲池名,
  N_PAGES AS 总页数,
  N_FREE AS 空闲页数,
  HIT_RATE AS 命中率
FROM V$BUFFERPOOL;

-- 命中率 > 95% 为正常，< 90% 需关注
```

#### 7. 归档状态

```sql
-- 查看归档状态
SELECT
  ARCH_MODE AS 归档模式,
  ARCH_STATUS AS 归档状态,
  ARCH_DEST AS 归档路径
FROM V$ARCHIVE_STATUS;

-- 归档空间使用情况
-- 可通过检查归档目录的磁盘空间实现：
-- df -h /data/dmarch/
```

#### 8. 备份状态

```sql
-- 查看最近备份记录
SELECT
  BACKUP_NAME,
  BACKUP_TYPE,
  BACKUP_TIME,
  BACKUP_SIZE,
  BACKUP_PATH,
  STATUS$
FROM V$BACKUPSET
ORDER BY BACKUP_TIME DESC
LIMIT 10;
```

---

## 日常巡检清单

建议每日/每周执行以下检查：

### 每日检查

| 序号 | 检查项 | SQL/命令 | 正常标准 |
|------|--------|---------|---------|
| 1 | 数据库运行状态 | `SELECT STATUS$ FROM V$INSTANCE;` | `OPEN` |
| 2 | 表空间使用率 | [上述 SQL](#2-表空间使用率) | < 80% |
| 3 | 归档空间 | `df -h /data/dmarch/` | < 80% |
| 4 | 备份是否成功 | `SELECT * FROM V$BACKUPSET ORDER BY BACKUP_TIME DESC LIMIT 1;` | 有当日备份 |
| 5 | 告警日志 | `tail -100 $DM_HOME/log/dm_实例名_alert.log` | 无异常错误 |
| 6 | 活跃会话数 | [上述 SQL](#3-会话监控) | 无异常突增 |
| 7 | 死锁检测 | `SELECT COUNT(*) FROM V$DEADLOCK_HISTORY WHERE DEADLOCK_TIME > SYSDATE - 1;` | 0 |

### 每周检查

| 序号 | 检查项 | 说明 |
|------|--------|------|
| 1 | SQL 性能分析 | 通过 DMLOG 或 V$SQL_HISTORY 找出 TOP 10 慢 SQL |
| 2 | 系统资源趋势 | 通过 NMON 或 DEM 分析 CPU/内存/IO 一周趋势 |
| 3 | 用户和权限审计 | 检查是否有未授权的用户或权限变更 |
| 4 | 备份恢复演练 | 在测试环境验证最近的备份是否可正常恢复 |
| 5 | 参数审查 | 检查关键参数是否异常（缓冲区、线程数、连接数等） |

### 告警阈值建议

| 指标 | 警告 | 严重 |
|------|------|------|
| 表空间使用率 | > 80% | > 90% |
| 归档空间使用率 | > 75% | > 90% |
| 数据库连接数 | > 70% 最大连接 | > 85% 最大连接 |
| 缓冲池命中率 | < 90% | < 80% |
| CPU 使用率 | > 80% 持续 5min | > 95% |
| IO Wait | > 10% | > 25% |
| 长事务（> 5min） | 1-5 个 | > 5 个 |
| 死锁数量 | ≥ 1 次 | ≥ 5 次 |

---

## 附录：快速诊断脚本

以下是一个综合诊断 SQL 脚本，可用于快速了解数据库健康状态：

```sql
-- DM 数据库健康检查脚本
-- 执行方式：disql SYSDBA/密码@IP:5236 \` 诊断.sql

PROMPT ========== 1. 数据库状态 ==========
SELECT STATUS$ AS "数据库状态" FROM V$INSTANCE;

PROMPT ========== 2. 数据库版本 ==========
SELECT BANNER AS "版本信息" FROM V$VERSION WHERE ROWNUM = 1;

PROMPT ========== 3. 表空间使用率 ==========
SELECT
  NAME AS "表空间",
  ROUND(CAST((TOTAL_SIZE - FREE_SIZE) AS DECIMAL)/CAST(TOTAL_SIZE AS DECIMAL)*100, 2) AS "使用率%",
  CAST(TOTAL_SIZE/1024 AS DECIMAL) AS "总大小MB"
FROM V$TABLESPACE
ORDER BY "使用率%" DESC;

PROMPT ========== 4. 活跃会话TOP 10 ==========
SELECT
  SESS_ID AS "会话ID",
  USER_NAME AS "用户",
  STATE AS "状态",
  CLNT_IP AS "客户端IP",
  APPNAME AS "应用"
FROM V$SESSIONS
WHERE STATE = 'ACTIVE'
LIMIT 10;

PROMPT ========== 5. 缓冲池命中率 ==========
SELECT
  NAME AS "缓冲池",
  HIT_RATE AS "命中率%"
FROM V$BUFFERPOOL;

PROMPT ========== 6. 锁等待 ==========
SELECT COUNT(*) AS "锁等待数" FROM V$LOCK WHERE BLOCKED = 1;

PROMPT ========== 7. 归档状态 ==========
SELECT ARCH_MODE AS "归档模式", ARCH_STATUS AS "归档状态" FROM V$ARCHIVE_STATUS;

PROMPT ========== 8. 最近备份 ==========
SELECT
  BACKUP_NAME AS "备份名称",
  BACKUP_TIME AS "备份时间"
FROM V$BACKUPSET
ORDER BY BACKUP_TIME DESC
LIMIT 3;

PROMPT ========== 诊断完成 ==========
```
