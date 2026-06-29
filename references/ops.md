# 运维管理

达梦数据库运维管理完整指南，涵盖安装部署、系统配置、归档、备份恢复和日常运维命令。

---

## 安装前环境准备（完整清单）

### 硬件要求

#### CPU 检查
```bash
# 查看 CPU 架构、颗数、核数
lscpu | grep -E "Architecture|Socket|Core|CPU\(s\)"
# 或
cat /proc/cpuinfo | grep "model name" | head -1
cat /proc/cpuinfo | grep "physical id" | sort | uniq | wc -l  # 物理CPU颗数
cat /proc/cpuinfo | grep "cpu cores" | uniq                     # 每颗CPU核数
```

#### 内存检查
```bash
free -h                              # 查看总内存
dmidecode -t memory | grep Size     # 查看物理内存插槽（部分系统支持）
```

#### 磁盘规划
建议至少 3 块独立磁盘或分区：

| 磁盘/目录 | 用途 | 建议大小 |
|-----------|------|----------|
| /dmdata | 数据文件存储 | 根据业务数据量，至少预留1年增长空间 |
| /dmbak | 备份文件存储 | 至少为数据盘大小的 3 倍 |
| /dmarch | 归档日志存储 | 每天归档量×保留天数，至少100GB |

```bash
# 查看磁盘信息
lsblk                        # 查看磁盘和分区
df -h                        # 查看挂载和容量
fdisk -l | grep "Disk /dev"  # 查看磁盘详情
```

#### 磁盘格式
**推荐 XFS**：64-bit、日志恢复快、支持高吞吐量、支持动态扩展。
```bash
# 格式化磁盘为 XFS
mkfs.xfs -f /dev/sdb
mkfs.xfs -f /dev/sdc

# 创建目录并挂载
mkdir -p /dmdata /dmbak /dmarch
mount /dev/sdb /dmdata
mount /dev/sdc /dmbak

# 写入 /etc/fstab 实现开机自动挂载
echo "/dev/sdb  /dmdata  xfs  defaults,noatime,nodiratime  0  0" >> /etc/fstab
echo "/dev/sdc  /dmbak   xfs  defaults,noatime,nodiratime  0  0" >> /etc/fstab
```

#### 磁盘调度
**必须设置为 deadline**（ARM 平台强制要求）：
```bash
# 查看当前调度算法
cat /sys/block/sda/queue/scheduler

# 临时设置
echo deadline > /sys/block/sda/queue/scheduler

# 永久设置（/etc/default/grub）
# 在 GRUB_CMDLINE_LINUX 中添加：elevator=deadline
# GRUB_CMDLINE_LINUX="crashkernel=auto rhgb quiet elevator=deadline"
# 然后执行：
grub2-mkconfig -o /boot/grub2/grub.cfg

# 验证
cat /sys/block/sda/queue/scheduler
# 输出中 [deadline] 表示当前生效
```

#### 网络要求
- 集群环境需要独立心跳网络（≥ 1000Mbps）
- 建议使用万兆网络

```bash
# 查看网卡信息
ip addr show
ethtool eth0 | grep Speed
```

---

### 系统配置

#### 1. 防火墙端口放行
```bash
# 生产环境推荐最小端口放行，而不是直接关闭防火墙。
# 单机默认服务端口
firewall-cmd --permanent --add-port=5236/tcp

# 集群环境按实际拓扑继续放行 RPC、DMCSS、ASM、心跳等端口。
# 示例：5237-5239 仅供参考，具体以部署方案为准。
firewall-cmd --permanent --add-port=5237-5239/tcp
firewall-cmd --reload
firewall-cmd --list-ports

# 测试环境为定位网络问题，可临时停止 firewalld；生产不建议长期关闭。
# systemctl stop firewalld
```

#### 2. 关闭 SELinux
```bash
# 临时关闭（立即生效）
setenforce 0

# 查看状态
getenforce   # 应输出 Permissive 或 Disabled

# 永久关闭（需重启）
# 编辑 /etc/selinux/config
# 将 SELINUX=enforcing 改为 SELINUX=disabled

# 一键修改
sed -i 's/^SELINUX=enforcing/SELINUX=disabled/' /etc/selinux/config
sed -i 's/^SELINUX=permissive/SELINUX=disabled/' /etc/selinux/config

# 验证配置文件
cat /etc/selinux/config | grep SELINUX
```

#### 3. 关闭 Swap
```bash
# 临时关闭
swapoff -a

# 永久关闭：注释 /etc/fstab 中的 swap 行
sed -i '/swap/s/^/#/' /etc/fstab

# 或手动编辑 /etc/fstab，在 swap 行前加 #
# /dev/mapper/centos-swap swap  swap  defaults  0 0
# 改为：
# #/dev/mapper/centos-swap swap  swap  defaults  0 0

# 验证
free -h   # Swap 行应全为 0
```

#### 4. 禁用透明大页
```bash
# 临时关闭
echo never > /sys/kernel/mm/transparent_hugepage/enabled

# 查看状态
cat /sys/kernel/mm/transparent_hugepage/enabled
# 应输出：always madvise [never]  （[never] 表示已关闭）

# 永久关闭方法1：/etc/rc.local
echo "echo never > /sys/kernel/mm/transparent_hugepage/enabled" >> /etc/rc.local
chmod +x /etc/rc.local

# 永久关闭方法2：grub（推荐）
# 编辑 /etc/default/grub，在 GRUB_CMDLINE_LINUX 中添加：
# transparent_hugepage=never
# 然后执行：
grub2-mkconfig -o /boot/grub2/grub.cfg

# 永久关闭方法3：tuned profile
tuned-adm profile throughput-performance
```

#### 5. 内存分配策略
```bash
# 设置为 0（启发式过度提交）
echo 0 > /proc/sys/vm/overcommit_memory

# 永久设置
echo "vm.overcommit_memory = 0" >> /etc/sysctl.conf
sysctl -p

# 验证
sysctl vm.overcommit_memory
# 应输出：vm.overcommit_memory = 0
```

#### 6. 关闭 RemoveIPC（重要！）
这是 systemd 的一个特性，会清理已登出用户的 IPC 资源，可能导致达梦数据库的共享内存被意外清理。

```bash
# 编辑 /etc/systemd/logind.conf
# 找到 #RemoveIPC=yes 行，去掉注释并改为 no

sed -i 's/^#RemoveIPC=yes/RemoveIPC=no/' /etc/systemd/logind.conf
sed -i 's/^RemoveIPC=yes/RemoveIPC=no/' /etc/systemd/logind.conf

# 重启 systemd-logind 使配置生效
systemctl restart systemd-logind
systemctl daemon-reload

# 验证
grep RemoveIPC /etc/systemd/logind.conf
# 应输出：RemoveIPC=no
```

---

### 创建用户和目录

```bash
# ====================
# 1. 创建安装用户组
# ====================
groupadd dinstall -g 2001

# ====================
# 2. 创建安装用户
# ====================
useradd -g dinstall dmdba -u 1001

# ====================
# 3. 设置密码
# ====================
passwd dmdba
# 根据提示输入密码

# ====================
# 4. 创建安装目录
# ====================
mkdir -p /opt/dmdbms
chown dmdba:dinstall /opt/dmdbms -R
chmod 755 /opt/dmdbms -R

# ====================
# 5. 挂载 DM 安装镜像（ISO文件安装方式）
# ====================
mount -o loop /path/to/dm8_*.iso /mnt
# 以 dmdba 用户身份进入 /mnt 执行 ./DMInstall.bin

# ====================
# 6. 如果是命令行安装
# ====================
su - dmdba
cd /mnt
./DMInstall.bin -i
# 按提示选择安装语言、时区、安装类型（典型/完整）、安装路径等
```

---

### limits.conf（资源限制）

达梦数据库对系统资源（特别是文件描述符和进程数）有较高要求，必须适当放宽。

```bash
# 编辑 /etc/security/limits.conf
vi /etc/security/limits.conf
```

在文件末尾添加：

```
# 达梦数据库 dmdba 用户资源限制
dmdba  soft  nproc     65536
dmdba  hard  nproc     65536
dmdba  soft  nofile    65536
dmdba  hard  nofile    65536
dmdba  soft  core      unlimited
dmdba  hard  core      unlimited
dmdba  soft  data      unlimited
dmdba  hard  data      unlimited
dmdba  soft  memlock   unlimited
dmdba  hard  memlock   unlimited
dmdba  soft  stack     65536
dmdba  hard  stack     65536
```

**参数说明：**

| 参数 | 含义 | 说明 |
|------|------|------|
| nproc | 最大进程数 | 达梦多线程模型需要大量线程，必须调大 |
| nofile | 最大打开文件数 | 数据文件、日志、网络连接等都需要文件描述符 |
| core | core dump 大小 | 便于故障排查 |
| data | 数据段大小 | unlimited 避免内存分配受限 |
| memlock | 锁定内存大小 | 支持内存页锁定 |
| stack | 栈大小 | 线程栈空间 |

```bash
# 验证是否生效（切换 dmdba 用户执行）
su - dmdba
ulimit -a

# 重点检查
ulimit -n    # 应为 65536
ulimit -u    # 应为 65536
```

---

### system.conf（systemd 全局限制）

```bash
# 编辑 /etc/systemd/system.conf
vi /etc/systemd/system.conf
```

取消注释并修改以下行：

```
DefaultLimitNOFILE=65536
DefaultLimitNPROC=10240
DefaultTasksMax=infinity
```

```bash
# 同样编辑 /etc/systemd/user.conf
vi /etc/systemd/user.conf
# 添加相同的配置

# 重载 systemd
systemctl daemon-reload
```

---

### nproc.conf

某些 Linux 发行版有独立的 nproc 限制文件：

```bash
# 编辑 /etc/security/limits.d/nproc.conf
vi /etc/security/limits.d/nproc.conf
```

添加：
```
dmdba  soft  nproc  65536
dmdba  hard  nproc  65536
```

如果该文件不存在：
```bash
echo "dmdba  soft  nproc  65536" > /etc/security/limits.d/nproc.conf
echo "dmdba  hard  nproc  65536" >> /etc/security/limits.d/nproc.conf
```

---

### 环境变量

以 dmdba 用户身份编辑环境变量：

```bash
su - dmdba
vi ~/.bash_profile
```

添加以下内容：

```bash
# 达梦数据库环境变量
export DM_HOME="/opt/dmdbms"
export LD_LIBRARY_PATH="$LD_LIBRARY_PATH:$DM_HOME/bin"
export PATH=$PATH:$DM_HOME/bin:$DM_HOME/tool

# glibc ≥ 2.10 必须加此参数，减少内存碎片
export MALLOC_ARENA_MAX=4

# 可选：设置字符集和语言
export LANG=zh_CN.UTF-8
export NLS_LANG=AMERICAN_AMERICA.UTF8

# 可选：命令行提示符中显示当前数据库实例
# export PS1="[\u@\h \W]\$ "
```

使环境变量立即生效：
```bash
source ~/.bash_profile

# 验证
echo $DM_HOME
echo $PATH
which disql
```

---

### 时间同步

数据库对时间一致性要求很高，集群环境更是强制要求节点间时间同步。

```bash
# ====================
# 1. 安装 chrony（推荐，替代 ntpd）
# ====================
yum install -y chrony

# ====================
# 2. 配置时间服务器
# ====================
vi /etc/chrony.conf

# 添加 NTP 服务器（将 NTP_SERVER_IP 替换为实际的时间服务器地址）
server NTP_SERVER_IP iburst

# 允许本网段访问（如果需要作为时间服务器）
allow 192.168.0.0/16

# ====================
# 3. 启动并设置开机自启
# ====================
systemctl restart chronyd
systemctl enable chronyd
systemctl status chronyd   # 确认运行中

# ====================
# 4. 验证时间同步
# ====================
chronyc sources -v
# 应看到 NTP 服务器状态为 ^* (已同步)
# chronyc tracking

# ====================
# 5. 手动强制同步
# ====================
chronyc makestep
```

---

## 归档配置

归档日志是数据库恢复的基础，**开启归档是备份的前提条件**。

### 什么是归档

归档日志 = 在线重做日志的历史副本。当数据库以归档模式运行时，已写满的在线重做日志会被复制到归档目录，实现：
- 数据库恢复到任意时间点
- 在线热备份
- 主备数据同步

### 开启归档

#### 方式一：SQL 命令（推荐，不中断服务）

```sql
-- Step 1: 切换到 MOUNT 状态
ALTER DATABASE MOUNT;

-- Step 2: 开启归档模式
ALTER DATABASE ARCHIVELOG;

-- Step 3: 添加归档配置
ALTER DATABASE ADD ARCHIVELOG 
    'DEST=/dmarch,        -- 归档目标路径
     TYPE=LOCAL,           -- 归档类型：LOCAL(本地) / REMOTE(远程) / LOCAL_REMOTE
     FILE_SIZE=2048,       -- 单个归档文件大小(MB)
     SPACE_LIMIT=102400';  -- 归档空间上限(MB)，0=不限制

-- Step 4: 打开数据库
ALTER DATABASE OPEN;

-- Step 5: 验证
SELECT ARCH_MODE FROM V$DATABASE;  -- 应为 'Y'
SELECT * FROM V$ARCHIVED_LOG ORDER BY FIRST_TIME DESC;
```

#### 方式二：修改配置文件（需重启）

**Step 1：修改 `dm.ini`**
```ini
ARCH_INI = 1   -- 启用归档配置文件
```

**Step 2：创建 `dmarch.ini`（与 dm.ini 同级目录）**
```ini
[ARCHIVE_LOCAL1]
ARCH_TYPE = LOCAL            -- 本地归档
ARCH_DEST = /dmarch          -- 归档目录
ARCH_FILE_SIZE = 2048        -- 单个归档文件大小(MB)
ARCH_SPACE_LIMIT = 102400    -- 归档空间上限(MB)，0=无限
ARCH_FLUSH_BUF_SIZE = 0      -- 归档缓冲区大小，0=自动
ARCH_HANG_FLAG = 1           -- 归档失败时的行为：1=挂起等待，0=报错
```

**Step 3：重启数据库**
```bash
# 停止
DmServiceDMSERVER stop

# 启动
DmServiceDMSERVER start
```

### 归档管理命令

```sql
-- ====================
-- 查看归档状态
-- ====================
SELECT 
    ARCH_MODE,        -- 是否归档模式
    ARCH_STATUS       -- 归档状态
FROM V$DATABASE;

-- ====================
-- 查看归档文件列表
-- ====================
SELECT 
    NAME,             -- 归档文件路径
    SEQUENCE#,        -- 归档序号
    FIRST_TIME,       -- 第一条日志的时间
    NEXT_TIME,        -- 最后一条日志的时间
    BLOCKS,           -- 块数
    BLOCK_SIZE,       -- 块大小
    STATUS            -- 状态：A=Active, D=Deleted
FROM V$ARCHIVED_LOG
ORDER BY SEQUENCE# DESC;

-- ====================
-- 手动切换归档日志
-- ====================
ALTER SYSTEM SWITCH LOGFILE;

-- ====================
-- 清理过期归档（谨慎操作！）
-- ====================
SELECT SF_ARCHIVELOG_DELETE_BEFORE_TIME(SYSDATE - 7);  -- 删除7天前的归档
```

---

## 备份

### 备份前提

> ⚠️ **必须开启归档！** 在非归档模式下，只能进行脱机备份（需要关闭数据库）。

### 备份类型说明

| 备份类型 | 说明 | 适用场景 |
|----------|------|----------|
| 全量备份 | 备份整个数据库 | 所有场景的基础 |
| 增量备份 | 仅备份自上次全量/增量后的变化（基于日志LSN） | 数据量大时减少备份时间和空间 |
| 联机备份 | 数据库运行中备份 | 生产环境，不能停机 |
| 脱机备份 | 关闭数据库后备份 | 特殊维护窗口 |

### 备份参数

| 参数 | 含义 | 建议值 | 说明 |
|------|------|--------|------|
| COMPRESS_LEVEL | 压缩级别 | 1 | 范围 1~9，建议 1（平衡压缩速度和压缩率） |
| PARALLEL_NUM | 并行度 | 2 | 范围 0~9，建议 2（减少对业务的影响，又保证备份速度） |

---

### 策略一：全量备份 + 定时删除（数据量 < 100GB）

适合中小规模数据库。每天 23:00 全量备份，自动删除 30 天前的备份文件。

```sql
-- ====================
-- 初始化作业系统
-- ====================
CALL SP_INIT_JOB_SYS(1);

-- ====================
-- 创建作业
-- ====================
CALL SP_CREATE_JOB(
    'bakall_delall',     -- 作业名称
    1,                    -- 作业启用标志：1=启用
    0,                    -- 允许创建者之外的用户修改：0=不允许
    '',                   -- 作业描述
    0,                    -- 启动方式：0=按调度
    0,                    -- 并行数
    '',                   -- 执行用户
    0,                    -- 优先级
    '全量备份+删除'       -- 作业说明
);

-- ====================
-- 配置作业步骤
-- ====================
CALL SP_JOB_CONFIG_START('bakall_delall');

-- 步骤1：全量备份
-- SP_ADD_JOB_STEP参数：
--   作业名, 步骤名, 步骤类型(6=备份/还原), 步骤描述串, 
--   失败后重试次数, 重试间隔(秒), 失败后行为(0=继续), 
--   成功后行为(0=继续), 错误处理, 保留参数
CALL SP_ADD_JOB_STEP(
    'bakall_delall', 
    'bakall', 
    6,   -- 步骤类型 6 = 备份/还原
    '01020000/opt/dmdbms/data/DAMENG/bak',  -- 备份描述串
    3,   -- 重试次数
    1,   -- 重试间隔(秒)
    0,   -- 失败后行为：0=不处理(继续执行下一步)
    0,   -- 成功后行为：0=不处理(继续执行下一步)
    NULL, 
    0
);

-- 步骤2：删除过期备份
CALL SP_ADD_JOB_STEP(
    'bakall_delall', 
    'delall', 
    0,   -- 步骤类型 0 = 执行SQL脚本
    'SF_BAKSET_BACKUP_DIR_ADD(''DISK'',''/opt/dmdbms/data/DAMENG/bak'');
     CALL SP_DB_BAKSET_REMOVE_BATCH(''DISK'',SYSDATE-30);',
    1, 
    1, 
    0, 
    0, 
    NULL, 
    0
);

-- ====================
-- 配置调度
-- ====================
CALL SP_ADD_JOB_SCHEDULE(
    'bakall_delall',              -- 作业名
    'bakall_delall_time01',       -- 调度名称
    1,                             -- 启用标志：1=启用
    1,                             -- 调度频率类型：1=按天
    1,                             -- 发生时：每天
    0,                             -- 发生分：0分
    0,                             -- 发生秒：0秒
    '23:00:00',                    -- 发生时间：23:00:00
    NULL,                          -- 结束时间
    '2019-01-01 01:01:01',        -- 起始时间
    NULL,                          -- 描述
    ''                             -- 保留参数
);

-- ====================
-- 提交作业
-- ====================
CALL SP_JOB_CONFIG_COMMIT('bakall_delall');

-- ====================
-- 验证作业是否创建成功
-- ====================
SELECT * FROM DBA_JOBS WHERE JOB_NAME = 'bakall_delall';
SELECT * FROM DBA_SCHEDULER_JOB_LOG WHERE JOB_NAME = 'bakall_delall' ORDER BY LOG_DATE DESC;
```

---

### 策略二：全量备份 + 增量备份 + 定时删除（100GB < 数据量 < 3TB）

适合中大规的数据库。每月第一个周六全量备份，每天（除周六）增量备份，自动删除 30 天增量、40 天全量。

```sql
-- ====================
-- 初始化作业系统
-- ====================
CALL SP_INIT_JOB_SYS(1);

-- ====================
-- 作业1：每月全量备份（第一个周六 23:00）
-- ====================
CALL SP_CREATE_JOB('bakall', 1, 0, '', 0, 0, '', 0, '每月全量备份');
CALL SP_JOB_CONFIG_START('bakall');

CALL SP_ADD_JOB_STEP(
    'bakall', 'bakall', 6, 
    '01020000/opt/dmdbms/data/DAMENG/bak/all', 
    1, 1, 0, 0, NULL, 0
);

CALL SP_ADD_JOB_SCHEDULE(
    'bakall', 'bakall_time01', 
    1,              -- 启用
    4,              -- 调度频率类型：4=按周
    1,              -- 间隔：1周
    7,              -- 执行日：7=周六
    0,              -- 执行时
    '23:00:00',     -- 发生时间
    NULL, 
    '2019-01-01 01:01:01', 
    NULL, ''
);

CALL SP_JOB_CONFIG_COMMIT('bakall');

-- ====================
-- 作业2：每日增量备份 + 删除过期备份（除周六外每天 23:00）
-- ====================
CALL SP_CREATE_JOB('bakadd_delbak', 1, 0, '', 0, 0, '', 0, '每日增量备份+删除');
CALL SP_JOB_CONFIG_START('bakadd_delbak');

-- 增量备份步骤
CALL SP_ADD_JOB_STEP(
    'bakadd_delbak', 'bakadd', 6, 
    '11020000/opt/dmdbms/data/DAMENG/bak/all|/opt/dmdbms/data/DAMENG/bak/add', 
    3, 1, 0, 0, NULL, 0
);

-- 删除过期备份步骤（30天增量、40天全量）
CALL SP_ADD_JOB_STEP(
    'bakadd_delbak', 'delbak', 0,
    'BEGIN
        SF_BAKSET_BACKUP_DIR_ADD(''DISK'',''/opt/dmdbms/data/DAMENG/bak/add'');
        SP_DB_BAKSET_REMOVE_BATCH(''DISK'',SYSDATE-30);
        SF_BAKSET_BACKUP_DIR_ADD(''DISK'',''/opt/dmdbms/data/DAMENG/bak/all'');
        SP_DB_BAKSET_REMOVE_BATCH(''DISK'',SYSDATE-40);
    END;',
    1, 1, 0, 0, NULL, 0
);

-- 调度：每周除周六外每天执行
--  类型=按周，间隔=1，执行日=63（位图：63=周六以外的所有天）
CALL SP_ADD_JOB_SCHEDULE(
    'bakadd_delbak', 'bakadd_delbak_time01', 
    1, 2, 1, 63, 0, '23:00:00', NULL, '2019-01-01 01:01:01', NULL, ''
);

CALL SP_JOB_CONFIG_COMMIT('bakadd_delbak');

-- ====================
-- 验证作业
-- ====================
SELECT JOB_NAME, JOB_DESC, ENABLE FROM DBA_JOBS WHERE JOB_NAME IN ('bakall', 'bakadd_delbak');

-- ====================
-- 作业管理
-- ====================
-- 查看作业执行历史
SELECT JOB_NAME, START_TIME, END_TIME, STATUS, ERROR_INFO 
FROM DBA_SCHEDULER_JOB_LOG 
WHERE JOB_NAME LIKE 'bak%' 
ORDER BY START_TIME DESC;

-- 手动启动作业（测试用）
CALL SP_DBMS_JOB_RUN('bakall_delall');

-- 立即删除指定作业
CALL SP_DBMS_JOB_REMOVE('bakall_delall');
```

### 备份描述串参数说明

备份作业步骤中的描述串格式为 17 字节：

| 字节位置 | 参数 | 取值 | 说明 |
|----------|------|------|------|
| 第1个字节 | 备份类型 | 0=全量 / 1=增量 | 0=全量备份，1=增量备份 |
| 第2个字节 | 备份方式 | 1=联机 / 2=脱机 | 联机备份需归档模式 |
| 第3个字节 | 备份介质 | 1=磁盘 / 2=磁带 | 通常为地址 |
| 第4个字节 | 备份时是否检查日志 | 0=不检查 / 1=检查 | 建议 0 |
| 第5-7个字节 | 备份内容的级别 | 000=库级 / 001=表空间 | 通常为 000 |
| 第8-10个字节 | 备份压缩级别 | 001~009 | 建议 001(最快压缩) |
| 第11-17个字节 | 备份并行度 | 001~009 | 建议 002(2并行) |

剩余字节为备份路径/备份集信息等。

---

## 还原与恢复

### 基本还原流程

```sql
-- ====================
-- 1. 检查备份集
-- ====================
SELECT BACKUP_NAME, BACKUP_TYPE, BACKUP_TIME, BASE_NAME, STATUS
FROM V$BACKUPSET;

-- ====================
-- 2. 全量还原（需在 MOUNT 状态）
-- ====================
-- 查看备份信息
SELECT SF_BAKSET_CHECK('DISK', '/opt/dmdbms/data/DAMENG/bak');

-- 还原数据库
RESTORE DATABASE FROM BACKUPSET '/opt/dmdbms/data/DAMENG/bak/备份集名';

-- 恢复数据库到最新状态
RECOVER DATABASE FROM BACKUPSET '/opt/dmdbms/data/DAMENG/bak/备份集名';

-- 更新 DB_MAGIC
RECOVER DATABASE UPDATE DB_MAGIC;

-- 打开数据库
ALTER DATABASE OPEN;
```

### 恢复到指定时间点

```sql
-- 恢复到指定时间
RECOVER DATABASE UNTIL TIME '2025-06-01 12:00:00';

-- 恢复到指定 LSN
RECOVER DATABASE UNTIL LSN 12345678;
```

---

## 常用运维命令

### 数据库状态管理

```sql
-- ====================
-- 查看数据库状态
-- ====================
SELECT STATUS$ FROM V$INSTANCE;
-- 返回值：1=启动，2=启动中，3=MOUNT，4=OPEN，5=挂起，6=关闭

-- ====================
-- 查看数据库基本信息
-- ====================
SELECT 
    NAME,              -- 数据库名称
    ARCH_MODE,         -- 归档模式
    STATUS$,           -- 状态
    TOTAL_SIZE,        -- 总大小(页数)
    CREATE_TIME        -- 创建时间
FROM V$DATABASE;

-- ====================
-- 查看数据库版本
-- ====================
SELECT * FROM V$VERSION;
SELECT ID_CODE;  -- 更简洁的版本信息
```

### 表空间管理

```sql
-- ====================
-- 查看表空间使用率（最常用的运维SQL之一）
-- ====================
SELECT 
    t.NAME AS TABLESPACE_NAME,
    ROUND(t.TOTAL_SIZE * SF_GET_PAGE_SIZE() / 1024.0 / 1024.0, 2) AS TOTAL_MB,
    ROUND(t.FREE_SIZE * SF_GET_PAGE_SIZE() / 1024.0 / 1024.0, 2) AS FREE_MB,
    ROUND((t.TOTAL_SIZE - t.FREE_SIZE) * SF_GET_PAGE_SIZE() / 1024.0 / 1024.0, 2) AS USED_MB,
    ROUND((t.TOTAL_SIZE - t.FREE_SIZE) * 100.0 / t.TOTAL_SIZE, 2) AS USED_PCT,
    t.PATH AS DATA_PATH
FROM V$TABLESPACE t
ORDER BY USED_PCT DESC;

-- ====================
-- 查看数据文件
-- ====================
SELECT 
    TS.NAME AS TABLESPACE_NAME,
    DF.PATH AS FILE_PATH,
    ROUND(DF.TOTAL_SIZE * SF_GET_PAGE_SIZE() / 1024.0 / 1024.0, 2) AS TOTAL_MB,
    ROUND(DF.FREE_SIZE * SF_GET_PAGE_SIZE() / 1024.0 / 1024.0, 2) AS FREE_MB,
    DF.AUTO_EXTEND,              -- 是否自动扩展
    ROUND(DF.MAX_SIZE * SF_GET_PAGE_SIZE() / 1024.0 / 1024.0, 2) AS MAX_MB
FROM V$DATAFILE DF
JOIN V$TABLESPACE TS ON DF.GROUP_ID = TS.ID;

-- ====================
-- 扩展表空间
-- ====================
-- 增加数据文件大小
ALTER TABLESPACE "MAIN" RESIZE DATAFILE '/dmdata/MAIN.DBF' TO 2048;

-- 为表空间添加新数据文件
ALTER TABLESPACE "MAIN" ADD DATAFILE '/dmdata/MAIN02.DBF' SIZE 1024 AUTOEXTEND ON NEXT 128 MAXSIZE 10240;
```

### 会话管理

```sql
-- ====================
-- 查看所有会话
-- ====================
SELECT 
    SESS_ID,
    USER_NAME,
    STATE,
    CLNT_HOST,
    CLNT_IP,
    CLNT_TYPE,
    APPNAME,
    (SYSDATE - LOGIN_TIME) * 86400 AS LOGIN_SECONDS,
    SQL_TEXT
FROM V$SESSIONS
ORDER BY LOGIN_TIME DESC;

-- ====================
-- 查看活动会话（正在执行SQL）
-- ====================
SELECT 
    SESS_ID,
    USER_NAME,
    STATE,
    CLNT_IP,
    SQL_TEXT,
    (SYSDATE - LAST_RECV_TIME) * 86400 AS ELAPSED_SECONDS
FROM V$SESSIONS 
WHERE STATE = 'ACTIVE'
  AND SQL_TEXT IS NOT NULL
ORDER BY LAST_RECV_TIME;

-- ====================
-- 查看会话的等待事件
-- ====================
SELECT 
    SESS_ID,
    EVENT_NAME,
    WAIT_TIME,
    WAIT_COUNT
FROM V$SESSION_WAIT
WHERE SESS_ID IN (SELECT SESS_ID FROM V$SESSIONS WHERE STATE = 'ACTIVE');

-- ====================
-- 杀死指定会话
-- ====================
SP_CLOSE_SESSION(123456789);  -- 替换为实际 sess_id
```

### 锁管理

```sql
-- ====================
-- 查看当前所有锁
-- ====================
SELECT 
    L.SESS_ID,
    S.USER_NAME,
    L.TABLE_ID,
    O.NAME AS TABLE_NAME,
    L.LTYPE,
    L.LMODE,
    L.BLOCKED
FROM V$LOCK L
LEFT JOIN V$SESSIONS S ON L.SESS_ID = S.SESS_ID
LEFT JOIN SYSOBJECTS O ON L.TABLE_ID = O.ID
ORDER BY L.BLOCKED DESC;

-- ====================
-- 只查看锁等待（被阻塞的事务）
-- ====================
SELECT 
    L_TID AS WAITING_TXN_ID,
    BLK_TID AS BLOCKING_TXN_ID,
    TABLE_ID,
    L_TYPE
FROM V$LOCK 
WHERE BLOCKED = 1;

-- ====================
-- 查看事务信息
-- ====================
SELECT 
    ID AS TXN_ID,
    SESS_ID,
    STATUS,
    BEGIN_TIME,
    ISOLATION_LEVEL
FROM V$TRX
WHERE STATUS = 'ACTIVE'
ORDER BY BEGIN_TIME;

-- ====================
-- 查看事务等待关系
-- ====================
SELECT 
    W.SESS_ID AS WAITING_SESS,
    W.USER_NAME AS WAITING_USER,
    B.SESS_ID AS BLOCKING_SESS,
    B.USER_NAME AS BLOCKING_USER
FROM V$TRXWAIT TW
JOIN V$SESSIONS W ON TW.ID = W.TRX_ID
JOIN V$SESSIONS B ON TW.WAIT_FOR_ID = B.TRX_ID;
```

### SQL与性能监控

```sql
-- ====================
-- 查看慢SQL（执行中的长时间SQL）
-- ====================
SELECT 
    SESS_ID,
    SQL_TEXT,
    EXEC_TIME,
    REMAIN_TIME,
    START_TIME,
    N_LOGIC_READS,
    N_READS
FROM V$LONG_EXEC_SQLS
WHERE EXEC_TIME > 5000   -- 超过5秒的SQL
ORDER BY EXEC_TIME DESC;

-- ====================
-- 查看SQL执行历史
-- ====================
SELECT 
    SESS_ID,
    SQL_TXT,
    EXEC_TIME,
    DISK_READS,
    BUF_READS,
    LAST_RECV_TIME
FROM V$SQL_HISTORY
WHERE LAST_RECV_TIME > SYSDATE - 1   -- 最近24小时
ORDER BY BUF_READS DESC;

-- ====================
-- 查看 TOP SQL（按逻辑读排序）
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

-- ====================
-- 查看缓存命中率
-- ====================
SELECT 
    NAME AS BUFFER_POOL_NAME,
    N_READ,          -- 物理读次数
    N_HIT,           -- 逻辑读命中次数
    ROUND(N_HIT * 100.0 / NULLIF(N_HIT + N_READ, 0), 2) AS HIT_RATIO_PCT
FROM V$BUFFERPOOL;

-- ====================
-- 查看内存池使用情况
-- ====================
SELECT 
    NAME,
    TYPE,
    ROUND(TOTAL_SIZE / 1024.0 / 1024.0, 2) AS TOTAL_MB,
    ROUND(USED_SIZE / 1024.0 / 1024.0, 2) AS USED_MB,
    ROUND(USED_SIZE * 100.0 / NULLIF(TOTAL_SIZE, 0), 2) AS USED_PCT
FROM V$MEM_POOL
ORDER BY USED_SIZE DESC;
```

### 参数管理

```sql
-- ====================
-- 查看参数
-- ====================
-- 按名称模糊查询
SELECT 
    PARA_NAME,
    PARA_VALUE,
    PARA_TYPE,          -- READ ONLY / SESSION / SYS / IN FILE
    DESCRIPTION
FROM V$DM_INI 
WHERE PARA_NAME LIKE 'BUFFER%';

-- 查询所有会话级参数
SELECT * FROM V$PARAMETER WHERE TYPE = 'SESSION';

-- 查询所有系统级参数
SELECT * FROM V$PARAMETER WHERE TYPE = 'SYS';

-- ====================
-- 查看参数当前值
-- ====================
-- SF_GET_PARA_VALUE(1, ...) 查询内存中的动态值
-- SF_GET_PARA_VALUE(2, ...) 查询配置文件中的静态值
SELECT 
    SF_GET_PARA_VALUE(1, 'BUFFER') AS MEMORY_VALUE,
    SF_GET_PARA_VALUE(2, 'BUFFER') AS FILE_VALUE;
```

### 日志与告警

```sql
-- ====================
-- 查看数据库告警日志位置
-- ====================
-- 告警日志在数据库安装目录的 log 子目录下
-- 文件名格式：dm_DB_NAME_YYYYMM.log

-- ====================
-- 查看数据库告警信息（SQL方式）
-- ====================
-- 查看最近的错误信息
SELECT * FROM V$ALERT_LOG_ERR ORDER BY LOG_DATE DESC LIMIT 50;

-- ====================
-- 查看数据库启动历史
-- ====================
SELECT STARTUP_TIME, STATUS$ FROM V$INSTANCE_STARTUP_HIST ORDER BY STARTUP_TIME DESC;
```

### 日常巡检脚本

建议将此脚本保存为 `dm_daily_check.sql`，每日执行：

```sql
-- ====================
-- 达梦数据库日常巡检脚本
-- ====================

-- 1. 数据库状态
SELECT 'DATABASE_STATUS' AS CHECK_ITEM, STATUS$ AS VALUE FROM V$INSTANCE;

-- 2. 归档模式
SELECT 'ARCHIVE_MODE' AS CHECK_ITEM, ARCH_MODE AS VALUE FROM V$DATABASE;

-- 3. 表空间使用率（重点关注 >80% 的）
SELECT 'TABLESPACE_USAGE' AS CHECK_ITEM, 
    NAME || ': ' || ROUND((TOTAL_SIZE-FREE_SIZE)*100.0/TOTAL_SIZE,1) || '%' AS VALUE
FROM V$TABLESPACE
WHERE (TOTAL_SIZE - FREE_SIZE) * 100.0 / TOTAL_SIZE > 80;

-- 4. 最近备份信息
SELECT 'LAST_BACKUP' AS CHECK_ITEM, 
    MAX(BACKUP_TIME) AS VALUE
FROM V$BACKUPSET;

-- 5. 当前活动会话数
SELECT 'ACTIVE_SESSIONS' AS CHECK_ITEM, COUNT(*) AS VALUE
FROM V$SESSIONS WHERE STATE = 'ACTIVE';

-- 6. 是否存在锁等待
SELECT 'LOCK_WAIT' AS CHECK_ITEM, 
    CASE WHEN COUNT(*) > 0 THEN 'YES(' || COUNT(*) || ')' ELSE 'NO' END AS VALUE
FROM V$LOCK WHERE BLOCKED = 1;

-- 7. 慢SQL（执行超过30秒的）
SELECT 'LONG_RUNNING_SQL' AS CHECK_ITEM, COUNT(*) AS VALUE
FROM V$LONG_EXEC_SQLS WHERE EXEC_TIME > 30000;

-- 8. 归档空间使用
SELECT 'ARCHIVE_DIR' AS CHECK_ITEM,
    COUNT(*) || ' files' AS VALUE
FROM V$ARCHIVED_LOG WHERE STATUS = 'A';
```
