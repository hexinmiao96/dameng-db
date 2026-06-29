# 达梦数据库安装部署完整指南

> 适用版本：DM8 | 涵盖 Windows 和 Linux 平台

---

## 目录

- [Windows 安装](#windows-安装)
  - [软件安装](#软件安装)
  - [配置实例](#配置实例)
  - [服务启停](#服务启停)
- [Linux 安装](#linux-安装)
  - [环境准备](#环境准备)
  - [软件安装（命令行）](#软件安装命令行)
  - [初始化实例](#初始化实例)
  - [dminit 参数详解](#dminit-参数详解)
  - [服务注册与启停](#服务注册与启停)

---

## Windows 安装

### 软件安装

**前置条件：**
- 操作系统：Windows 7/10/11 或 Windows Server 2008/2012/2016/2019/2022（64位）
- 内存：至少 1GB，推荐 4GB+
- 磁盘空间：至少 2GB 可用空间
- 建议关闭杀毒软件以避免安装过程中文件被拦截

**安装步骤：**

1. **装载 ISO 镜像**
   - 右键 ISO 文件 → "装载"
   - 或使用虚拟光驱工具（如 Daemon Tools）挂载
   - 双击 `setup.exe` 启动安装向导

2. **选择语言和时区**
   - 选择"简体中文"和"中国标准时间"（Asia/Shanghai）
   - 点击"确定"继续

3. **接受许可协议**
   - 阅读《达梦数据库管理系统使用许可协议》
   - 勾选"接受"并点击"下一步"

4. **验证 Key 文件**（可跳过）
   - 如果有正式授权文件（dm.key），点击"浏览"导入
   - 若为测试/试用环境，可跳过此步骤直接"下一步"
   - 跳过 Key 文件时会有提示，确认即可

5. **选择安装类型**
   - **典型安装**（推荐）：安装完整的 DM 数据库服务器、客户端工具、驱动和文档
   - **服务器安装**：仅安装数据库服务器组件
   - **客户端安装**：仅安装客户端工具和驱动（不包含数据库引擎）
   - **自定义安装**：自行选择要安装的组件

6. **选择安装路径**
   - 路径要求：**仅限英文字母、数字和下划线，禁止使用空格和中文字符**
   - 推荐路径：
     - `D:\dmdba\dmdbms`（非系统盘，推荐）
     - `C:\dmdba\dmdbms`（系统盘）
   - **警告**：路径中包含空格或中文会导致数据库初始化异常

7. **确认安装**
   - 检查安装摘要信息：安装路径、所需磁盘空间、安装类型
   - 点击"安装"开始执行
   - 等待约 1-2 分钟完成文件复制

8. **初始化配置**
   - 安装完成后，弹出"是否初始化数据库"对话框
   - **选择"初始化"**进入数据库配置助手（DBCA）

### 配置实例

#### 第一步：选择操作类型
- 选择 **"创建数据库实例"**
- 点击"下一步"

#### 第二步：选择模板
- **一般用途**（推荐）：适用于 OLTP 与 OLAP 混合场景
- **OLTP**：联机事务处理，适合高并发小事务
- **OLAP**：联机分析处理，适合大数据量分析查询
- 点击"下一步"

#### 第三步：指定数据目录
- 数据文件存放目录，如 `D:\dmdba\dmdbms\data`
- 目录路径同样需要遵循英文路径规范
- 确保目标磁盘有足够的可用空间
- 点击"下一步"

#### 第四步：设置基本信息
| 参数 | 说明 | 推荐值 |
|------|------|--------|
| 数据库名（DB_NAME） | 数据库全局名称 | `DAMENG`（默认） |
| 实例名（INSTANCE_NAME） | 数据库实例标识 | `DMSERVER`（默认） |
| 端口号（PORT_NUM） | 数据库监听端口 | `5236`（默认） |

> 若同一台服务器部署多个实例，端口号不可重复

#### 第五步：配置初始化参数

| 参数 | 默认值 | 推荐生产值 | 不可变？ | 说明 |
|------|--------|-----------|---------|------|
| **簇大小（EXTENT_SIZE）** | 16页 | 32页 | ✅ **是** | 每次分配空间的连续页数。32页=更大的分配单元，减少空间管理开销 |
| **页大小（PAGE_SIZE）** | 8K | **32K** | ✅ **是** | 决定字段长度的上限。一旦设定无法更改！详见[页大小影响表](#页大小影响) |
| **日志文件大小（LOG_SIZE）** | 256M | **2048M** | ❌ 否 | REDO日志文件大小，影响恢复性能和归档效率 |
| **大小写敏感（CASE_SENSITIVE）** | Y | **视源库而定** | ✅ **是** | MySQL迁移→**N**，Oracle迁移→**Y** |
| **字符集（CHARSET）** | GB18030(0) | 0或1 | ❌ 否（有变更方式） | **0**=GB18030（中文推荐），**1**=UTF-8（国际化） |
| **空格填充（BLANK_PAD_MODE）** | N | Y/视源库 | ✅ **是** | Oracle兼容模式需设为Y |

**关键决策指南：**

- **页大小（PAGE_SIZE）** 是最关键的参数，一旦设定无法修改，必须提前规划：
  - 有超过 3878 字节的 `VARCHAR` 字段 → 至少选 16K
  - 需要存储大量 JSON/XML 文本 → 选 32K
  - 纯 OLTP 小字段场景 → 8K 足够
  - **推荐生产环境直接选 32K**，避免扩容烦恼

- **大小写敏感（CASE_SENSITIVE）** 决定了 SQL 中对象名是否区分大小写：
  - 设为 `Y`：`SELECT * FROM TEST;` 与 `SELECT * FROM test;` 不同
  - 设为 `N`：以上两条 SQL 等价
  - 若从 MySQL 迁移数据，强烈建议选 `N`
  - 若从 Oracle 迁移数据，必须选 `Y`

- **字符集（CHARSET）** 选择：
  - 国内业务 → **GB18030（0）**，空间效率更高
  - 国际化/多语言业务 → **UTF-8（1）**
  - ⚠️ 一旦创建数据库后修改极为复杂，务必慎重
  - ⚠️ GB18030 和 UTF-8 需要按源库字符集、业务样本和驱动编码一起核对，不要只按默认值选择

#### 第六步：设置系统管理员密码

| 管理员账户 | 说明 | 密码要求 |
|-----------|------|----------|
| **SYSDBA** | 系统管理员，拥有最高权限 | 必须包含大写字母 + 小写字母 + 数字 |
| **SYSAUDITOR** | 审计管理员，管理审计日志 | 同上，且不能与 SYSDBA 相同 |
| **SYSSSO** | 安全管理员（仅安全版） | 同上 |

> **密码示例：** 使用符合现场密码策略的强密码，不要在文档或仓库中写入真实密码。

> ⚠️ 密码不允许与用户名重名，如 `SYSDBA` 不能作为 SYSDBA 的密码

#### 第七步：创建示例库（可选但推荐）
- 勾选 **"创建示例数据库"**
- DM 提供两个示例模式：
  - `BOOKSHOP`：图书销售系统示例
  - `DMHR`：人力资源管理示例
- 示例库帮助学习 SQL 语法和功能测试

#### 第八步：确认摘要并执行
- 仔细检查所有配置参数
- **特别注意页大小和大小写敏感值**，因为这些不可逆
- 确认无误后点击"执行"
- 等待初始化完成（通常 1-2 分钟）

### 服务启停

#### 方式一：DM 服务查看器（图形界面）

1. 从开始菜单找到"达梦数据库" → "DM 服务查看器"
2. 在服务列表中选中要操作的数据库服务（如 `DmServiceDMSERVER`）
3. 右键或使用工具栏按钮进行：**启动** / **停止** / **重启**

#### 方式二：Windows 服务管理

1. 运行 `services.msc` 打开 Windows 服务管理器
2. 找到 `DmServiceDMSERVER` 服务
3. 右键选择"启动"、"停止"或"重新启动"

#### 方式三：命令行

```cmd
:: 启动服务
net start DmServiceDMSERVER

:: 停止服务
net stop DmServiceDMSERVER
```

#### 方式四：DmService 可执行文件

```cmd
DmServiceDMSERVER.exe start
DmServiceDMSERVER.exe stop
DmServiceDMSERVER.exe restart
```

---

## Linux 安装

### 环境准备

> 详细的环境准备步骤请参阅 `ops.md`，以下是核心步骤摘要。

#### 1. 创建 dmdba 用户

```bash
# 创建用户组
groupadd dinstall -g 2001

# 创建 dmdba 用户（注意：禁止使用 root 安装）
useradd -g dinstall dmdba -u 1001

# 设置密码
passwd dmdba

# 创建安装目录并授权
mkdir -p /opt/dmdbms
chown -R dmdba:dinstall /opt/dmdbms
```

#### 2. 防火墙端口放行和 SELinux

```bash
# 生产环境推荐最小端口放行，避免直接关闭防火墙。
firewall-cmd --permanent --add-port=5236/tcp
firewall-cmd --reload
firewall-cmd --list-ports

# 测试环境为定位网络问题，可临时停止 firewalld；生产不建议长期关闭。
# systemctl stop firewalld

# 关闭 SELinux（临时 + 永久）
setenforce 0
sed -i 's/SELINUX=enforcing/SELINUX=disabled/g' /etc/selinux/config

# 确认 SELinux 状态
getenforce  # 应输出 Disabled 或 Permissive
```

#### 3. 关闭 swap

```bash
# 临时关闭
swapoff -a

# 永久关闭（注释掉 swap 行）
sed -i '/swap/s/^/#/' /etc/fstab
```

#### 4. 关闭透明大页（Transparent HugePages）

```bash
# 临时关闭
echo never > /sys/kernel/mm/transparent_hugepage/enabled
echo never > /sys/kernel/mm/transparent_hugepage/defrag

# 永久关闭（添加到 /etc/rc.d/rc.local 并赋予执行权限）
echo 'echo never > /sys/kernel/mm/transparent_hugepage/enabled' >> /etc/rc.d/rc.local
echo 'echo never > /sys/kernel/mm/transparent_hugepage/defrag' >> /etc/rc.d/rc.local
chmod +x /etc/rc.d/rc.local
```

#### 5. 配置系统资源限制

编辑 `/etc/security/limits.conf`，添加以下内容:

```bash
cat >> /etc/security/limits.conf << 'EOF'
# DM Database resource limits
dmdba   soft   nofile   65536
dmdba   hard   nofile   65536
dmdba   soft   nproc    65536
dmdba   hard   nproc    65536
dmdba   soft   core     unlimited
dmdba   hard   core     unlimited
EOF
```

验证：

```bash
su - dmdba
ulimit -a
# 检查 open files 和 max user processes 的值是否已更新
```

#### 6. 配置环境变量

编辑 `~/.bash_profile`（dmdba 用户），添加：

```bash
cat >> ~/.bash_profile << 'EOF'
# DM Database Environment
export DM_HOME=/opt/dmdbms
export LD_LIBRARY_PATH=$DM_HOME/bin:$LD_LIBRARY_PATH
export PATH=$DM_HOME/bin:$DM_HOME/tool:$PATH
EOF

# 使配置生效
source ~/.bash_profile
```

### 软件安装（命令行）

#### 第一步：获取安装包并挂载 ISO

```bash
# 上传 DM8 安装包到服务器（如 /opt/setup/）
# 假设文件名为 dm8_20240618_x86_rh7_64.iso

# 挂载 ISO 镜像
mkdir -p /mnt/dm
mount -o loop /opt/setup/dm8_20240618_x86_rh7_64.iso /mnt/dm
```

#### 第二步：命令行安装

```bash
# 切换到 dmdba 用户
su - dmdba

# 执行命令行安装
/mnt/dm/DMInstall.bin -i
```

**安装过程交互：**

```
请选择安装语言(C/c:中文 E/e:英文) [C/c]:
→ 输入 C（或直接回车）

请选择时区 [21]: G21 标准时区                        → 直接回车
1. 标准时区: G21 标准时区
2. 自定义时区
请选择时区类型 [1]:
→ 输入 1（或直接回车）

是否输入Key文件路径？ (Y/y:是 N/n:否) [N/n]:
→ 输入 N（测试环境跳过，正式环境选 Y 并输入路径）

请选择安装类型:
1. 典型安装（推荐）
2. 服务器安装
3. 客户端安装
4. 自定义安装
请选择安装类型 [1]:
→ 输入 1（或直接回车）

请选择安装路径:
→ 输入 /opt/dmdbms

确认安装路径(/opt/dmdbms)? (Y/y:是 N/n:否) [Y/y]:
→ 输入 Y

是否确认安装？ (Y/y:是 N/n:否) [Y/y]:
→ 输入 Y
```

安装过程持续约 1-2 分钟。

#### 第三步：以 root 身份注册服务

安装完成时，终端会提示需要以 root 执行脚本：

```bash
# 切换到 root 用户执行注册脚本
su - root
/opt/dmdbms/script/root/root_installer.sh

# 执行后输出类似：
# 移动 /opt/dmdbms/bin/dm_svc.conf 到 /etc 目录
# 创建 DmAPService 服务...
# 启动 DmAPService 服务...
```

### 初始化实例

#### 使用 dminit 命令行工具

```bash
# 切换到 dmdba 用户
su - dmdba

# 创建数据目录
mkdir -p /opt/dmdbms/data

# 进入 bin 目录
cd /opt/dmdbms/bin

# 初始化数据库实例
./dminit PATH=/opt/dmdbms/data \
  PAGE_SIZE=32 \
  EXTENT_SIZE=32 \
  LOG_SIZE=2048 \
  CHARSET=0 \
  CASE_SENSITIVE=Y \
  DB_NAME=DAMENG \
  INSTANCE_NAME=DMSERVER \
  PORT_NUM=5236 \
  SYSDBA_PWD=<StrongPassword> \
  SYSAUDITOR_PWD=<StrongPassword>
```

**初始化成功输出示例：**

```
initdb V8
db version: 0x7000c
file dm.key not found, use default license!
License will expire on 2025-06-18

 log file path: /opt/dmdbms/data/DAMENG/DAMENG01.log

 log file path: /opt/dmdbms/data/DAMENG/DAMENG02.log

write to dir [/opt/dmdbms/data/DAMENG].
create dm database success. 2024-06-18 10:30:00
```

### dminit 参数详解

| 参数 | 说明 | 可选值 | 默认值 | 是否不可变 |
|------|------|--------|--------|-----------|
| **PATH** | 数据文件存放路径 | 绝对路径 | 必填参数 | — |
| **PAGE_SIZE** | 页大小（K） | 4/8/16/32 | 8 | ✅ **不可变** |
| **EXTENT_SIZE** | 簇大小（页数） | 16/32/64 | 16 | ✅ **不可变** |
| **LOG_SIZE** | REDO日志文件大小（M） | 256-8192（64位）| 256 | ❌ 可调整 |
| **CASE_SENSITIVE** | 大小写敏感 | Y/N | Y | ✅ **不可变** |
| **CHARSET** | 字符集编码 | 0=GB18030<br>1=UTF-8<br>2=EUC-KR | 0 | ❌ 有变更方式但复杂 |
| **DB_NAME** | 数据库名称 | 最大128字符 | DAMENG | ✅ **不可变** |
| **INSTANCE_NAME** | 实例名称 | 最大128字符 | DMSERVER | ✅ **不可变** |
| **PORT_NUM** | 监听端口号 | 1024-65535 | 5236 | ❌ 可通过 dm.ini 修改 |
| **SYSDBA_PWD** | SYSDBA 管理员密码 | 大写+小写+数字 | — | ❌ 可 ALTER USER 修改 |
| **SYSAUDITOR_PWD** | SYSAUDITOR 审计员密码 | 大写+小写+数字 | — | ❌ 可 ALTER USER 修改 |
| **BLANK_PAD_MODE** | 空格填充模式 | Y/N | N | ✅ **不可变** |
| **LENGTH_IN_CHAR** | VARCHAR 是否以字符为单位 | Y/N | N | ✅ **不可变** |
| **PAGE_CHECK** | 页校验模式 | 0/1/2/3 | 0 | ❌ 可调整 |
| **UNICODE_FLAG** | 字符集类型（与 CHARSET 配合） | 0/1 | 0 | ✅ **不可变** |
| **USE_NEW_HASH** | 使用新哈希算法 | Y/N | Y | ✅ **不可变** |

#### 参数详细说明

**PAGE_SIZE（页大小）**

数据库的最小存储单元。**一旦设定不可更改**，必须根据业务需求提前规划：

| 页大小 | VARCHAR 最大定义长度 | 每行非字段最大长度 | 数据文件最小/最大 |
|--------|---------------------|-------------------|-------------------|
| 4K | 1938 字节 | 2047 字节 | 16M / 8T |
| 8K | 3878 字节 | 4095 字节 | 32M / 16T |
| 16K | 8000 字节 | 8195 字节 | 64M / 32T |
| 32K | 8188 字节 | 16176 字节 | 128M / 64T |

> **建议：** 生产环境直接选用 32K 页大小，以获得最大的灵活性和最好的性能。

**EXTENT_SIZE（簇大小）**

数据库中空间分配的最小单元（以页为单位）。较大的簇大小减少空间管理开销，适合大数据量场景；较小的簇大小节省空间，适合小数据量场景。

**LOG_SIZE（日志文件大小）**

每个 REDO 日志文件的大小（单位：MB）。日志文件越大，切换频率越低，但恢复所需时间可能更长。生产环境建议 2048M 以上。

**CASE_SENSITIVE（大小写敏感）**

- 设为 `Y` 时，标识符区分大小写（Oracle 风格），如 `"Test"` 不同于 `"TEST"`
- 设为 `N` 时，标识符不区分大小写（MySQL/SQL Server 风格）
- 迁移场景的关键决定因素：MySQL→N，Oracle→Y

**CHARSET / UNICODE_FLAG（字符集）**

| CHARSET | UNICODE_FLAG | 实际字符集 | 适用场景 |
|---------|-------------|-----------|----------|
| 0 | 0 | GB18030 | 国内中文业务 |
| 0 | 1 | UTF-8 | 国际化多语言 |
| 1 | 0 | UTF-8 | 同上 |
| 2 | 0 | EUC-KR | 韩语业务 |

**BLANK_PAD_MODE（空格填充模式）**

- 设为 `Y`：字符串比较时空格填充到相等长度再比较（Oracle 兼容模式）
- 设为 `N`：按实际长度比较（标准 SQL 模式）
- 从 Oracle 迁移时务必设为 `Y`

### 服务注册与启停

#### 注册数据库服务

```bash
# 使用 dm_service_installer.sh 注册服务
cd /opt/dmdbms/bin

./dm_service_installer.sh -t dmserver \
  -dm_ini /opt/dmdbms/data/DAMENG/dm.ini \
  -p DMSERVER
```

**参数说明：**

- `-t dmserver`：服务类型，注册为数据库服务器
- `-dm_ini`：指定 dm.ini 配置文件路径
- `-p`：服务名前缀，生成的服务名为 `DmServiceDMSERVER`

**注册其他类型服务（可选）：**

```bash
# 注册 DMAP 服务（辅助插件服务）
./dm_service_installer.sh -t dmap

# 注册守护进程服务（数据守护集群）
./dm_service_installer.sh -t dmwatcher \
  -watcher_ini /opt/dmdbms/data/DAMENG/dmwatcher.ini \
  -p DMWATCHER

# 注册监视器服务（数据守护集群）
./dm_service_installer.sh -t dmmonitor \
  -monitor_ini /opt/dmdbms/data/DAMENG/dmmonitor.ini \
  -p DMMONITOR
```

#### 服务启停

```bash
# ========== 方式一：systemctl（推荐） ==========
# 如果服务已注册到 systemd
systemctl start DmServiceDMSERVER
systemctl stop DmServiceDMSERVER
systemctl restart DmServiceDMSERVER
systemctl status DmServiceDMSERVER

# 设置开机自启
systemctl enable DmServiceDMSERVER

# ========== 方式二：服务脚本 ==========
# 启动
/opt/dmdbms/bin/DmServiceDMSERVER start

# 停止
/opt/dmdbms/bin/DmServiceDMSERVER stop

# 重启
/opt/dmdbms/bin/DmServiceDMSERVER restart

# 查看状态
/opt/dmdbms/bin/DmServiceDMSERVER status

# ========== 方式三：直接使用 dm.ini ==========
# 前台启动（调试用，Ctrl+C 停止）
/opt/dmdbms/bin/dmserver /opt/dmdbms/data/DAMENG/dm.ini

# 后台启动（结合 nohup）
nohup /opt/dmdbms/bin/dmserver /opt/dmdbms/data/DAMENG/dm.ini &
```

#### 连接验证

```bash
# 使用 disql 命令行工具连接
/opt/dmdbms/bin/disql SYSDBA/<StrongPassword>@localhost:5236

# 成功连接后输出：
# 服务器[localhost:5236]:处于普通打开状态
# 登录使用时间 : 3.456(ms)
# SQL>
```

**常用 disql 命令：**

```sql
-- 查看数据库版本
SELECT * FROM V$VERSION;

-- 查看数据库状态
SELECT STATUS$ FROM V$INSTANCE;

-- 查看初始化参数
SELECT NAME, VALUE FROM V$PARAMETER WHERE NAME IN ('PAGE_SIZE','EXTENT_SIZE','CASE_SENSITIVE');

-- 退出
EXIT;
```

#### 卸载 DM 数据库

```bash
# 1. 停止服务
/opt/dmdbms/bin/DmServiceDMSERVER stop

# 2. 以 root 执行卸载脚本
su - root
/opt/dmdbms/root/uninstall.sh

# 3. 清理残留文件和用户（可选）
rm -rf /opt/dmdbms
rm -rf /opt/dmdbms/data
userdel -r dmdba
groupdel dinstall
```

---

## 附录：安装常见问题

### Q1: 安装时提示 "权限不足"
- Windows：以管理员身份运行 setup.exe
- Linux：确保 dmdba 用户对安装目录有写权限（`chown -R dmdba:dinstall /opt/dmdbms`）

### Q2: 初始化失败，提示 "page size too small"
- 检查是否有字段定义长度超过当前页大小限制
- 重新选择更大的页大小（建议 32K）

### Q3: 连接报错 "网络通信异常"
- 检查防火墙是否放行端口（如 5236）
- 检查数据库服务是否已启动：`DmServiceDMSERVER status`
- 检查监听端口是否正常：`netstat -tlnp | grep 5236`

### Q4: 如何确认当前数据库的页大小和大小写敏感？
```sql
SELECT
  (SELECT PARA_VALUE FROM V$DM_INI WHERE PARA_NAME='CASE_SENSITIVE') AS 大小写敏感,
  (SELECT CAST(PAGE/1024 AS VARCHAR)+'K' FROM V$DATABASE) AS 页大小,
  (SELECT SF_GET_EXTENT_SIZE()) AS 簇大小页数;
```

### Q5: 正式环境 Key 文件如何获取？
- 联系达梦销售购买正式授权
- 获得 dm.key 文件后放入 `$DM_HOME/bin/` 目录
- 可通过 `SELECT * FROM V$LICENSE;` 查看授权信息
