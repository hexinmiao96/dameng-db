# 达梦数据库集群架构概览

达梦数据库提供从主备高可用到分布式集群的完整集群解决方案，覆盖单机房到跨地域多活的全场景。

---

## 一、数据守护集群（DMDataWatch）

**DataWatch** 是达梦最高频使用的高可用集群方案，基于日志实时同步实现主备切换。

### 架构原理

```
        ┌──────────────┐
        │   应用客户端   │
        └──────┬───────┘
               │
        ┌──────▼───────┐        实时 REDO 日志同步
        │   主库(Primary) │ ──────────────────────► │   备库(Standby)   │
        │  读写服务       │ ◄────────────────────── │  只读/备用         │
        └──────┬───────┘        确认消息回执            └──────────────────┘
               │
        ┌──────▼───────┐
        │  守护进程 Monitor│ ─── 监控主备健康状态
        └──────────────┘     ─── 触发故障自动切换
```

### 核心特性

| 特性 | 说明 |
|------|------|
| **同步模式** | 实时主备（高性能）/ 实时主备（高安全）/ 异步 |
| **故障切换** | 秒级自动检测并切换，业务中断时间最小化 |
| **数据一致性** | 实时主备模式保证主备数据**完全一致**，零丢失 |
| **备库临时表** | 备库支持创建和使用临时表（DML 操作限于临时表） |
| **自动恢复** | 旧主恢复后自动作为新备库加入，无需人工干预 |

### 典型部署拓扑

```
单机主备（最简部署）：
  物理机 A：主库 + 守护进程1
  物理机 B：备库 + 守护进程2

一主多备（更高可靠）：
  物理机 A：主库
  物理机 B：实时备库1
  物理机 C：异步备库2（异地灾备，允许少量延迟）
```

### 使用场景

- 绝大多数生产系统的标准高可用方案
- 对 RPO（恢复点目标）要求为 0 的核心业务
- 只需要主备切换、不需要自动读写分离的场景

---

## 二、读写分离集群（DMRWC）

**RWC（Read-Write Cluster）** 在 DataWatch 的高可用基础上，增加了**自动读写分离**能力。

### 架构原理

```
                    ┌──────────────┐
                    │  客户端/应用   │
                    └──────┬───────┘
                           │ 连接 DM 接口层
                    ┌──────▼───────┐
                    │  RWC 分发层   │ ← 自动判断读写
                    └──┬────────┬──┘
              ┌────────┘        └────────┐
              ▼                           ▼
     ┌────────────┐              ┌────────────┐
     │   主库      │  日志同步   │   备库      │
     │  写操作     │ ──────────► │  读操作     │
     │  部分读操作  │              │  主要读操作  │
     └────────────┘              └────────────┘
```

### 核心特性

| 特性 | 说明 |
|------|------|
| **读写自动分离** | 接口层自动将读操作路由到备库，写操作路由到主库 |
| **事务一致性** | 事务内的多语句始终保持在同一节点执行，不跨节点 |
| **强一致性读** | 基于主备强一致同步前提，备库数据与主库实时一致 |
| **透明性** | 应用层无需任何代码修改，连接串配置即可 |
| **高可用** | 继承 DataWatch 的主备切换能力 |

### 读写分发策略

```
自动分发规则：
  SELECT 查询          → 备库（分散主库压力）
  INSERT/UPDATE/DELETE → 主库
  SELECT ... FOR UPDATE → 主库（涉及锁）
  显式事务块           → 事务内全部在主库或全部在备库
  调用存储过程/函数    → 主库（可能含写操作）
  DDL 语句             → 主库
```

### 使用场景

- **读多写少**的业务系统（如 OA 办公、报表查询、内容管理系统）
- 希望通过备库分担查询负载来提升整体吞吐
- 应用层不便/不愿修改代码实现分库逻辑

---

## 三、共享存储集群（DMDSC）

**DSC（DM Shared Cluster）** 对标 Oracle RAC，实现多实例同时访问同一份共享存储的数据库。

### 架构原理

```
   ┌──────────┐    ┌──────────┐    ┌──────────┐
   │ 实例 1    │    │ 实例 2    │    │ 实例 N    │
   │ 内存+进程  │    │ 内存+进程  │    │ 内存+进程  │
   └─────┬─────┘    └─────┬─────┘    └─────┬─────┘
         │                │                │
         └────────────────┼────────────────┘
                          │
              ┌───────────▼───────────┐
              │   共享存储（SAN/分布式） │
              │   ├── 数据文件          │
              │   ├── 控制文件          │
              │   ├── REDO 日志（每实例独立）│
              │   └── 归档日志          │
              └───────────────────────┘
```

### 核心特性

| 特性 | 说明 |
|------|------|
| **多实例单库** | 多节点同时读写同一份数据，无数据冗余 |
| **金融级高可用** | 单实例故障时其他实例自动接管，业务无感知 |
| **自动负载均衡** | 新连接自动分配到负载较低的实例 |
| **缓存融合** | DMCSS 同步各实例 Buffer Pool，保证数据一致性 |
| **国产平台支持** | 支持国产 CPU 和操作系统平台 |

### 各组件职责

| 组件 | 全称 | 职责 |
|------|------|------|
| **DMCSS** | DM Cluster Synchronization Service | 心跳检测、集群成员管理、投票仲裁 |
| **DCR** | DM Cluster Registry | 集群配置信息存储（磁盘上的配置文件） |
| **DB** | Database Instance | 数据库实例进程，提供 SQL 服务 |
| **ASM** | Automatic Storage Management | 自动存储管理（可选，管理共享磁盘组） |

### 使用场景

- **金融、证券、电信**等对业务连续性要求极高的行业
- 无法接受计划内停机（如单机升级需要停库）的场景
- 已有共享存储设备（SAN 光纤存储）投资的环境

---

## 四、新一代分布式集群（DMDPC）

**DPC（DM Distributed Processing Cluster）** 是达梦新一代原生分布式数据库，支持 HTAP（Hybrid Transactional/Analytical Processing）混合负载。

### 架构原理

```
                      ┌──────────────┐
                      │   客户端/应用   │
                      └──────┬───────┘
                             │
                      ┌──────▼───────┐
                      │ 计算节点(SP)   │ ← SQL 解析、计划生成、分布式执行
                      └──┬────────┬──┘
                         │        │
              ┌──────────┘        └──────────┐
              ▼                               ▼
      ┌────────────┐                 ┌────────────┐
      │ 存储节点(BP1)│                 │ 存储节点(BP2)│
      │ 数据分片1+2  │ ◄── 数据同步 ──► │ 数据分片3+4  │
      └────────────┘                 └────────────┘
              ▲                               ▲
              └───────────────┬───────────────┘
                              │
                      ┌───────▼───────┐
                      │  元数据节点(MP) │ ← 集群拓扑、分片信息
                      └───────────────┘
```

### 核心特性

| 特性 | 说明 |
|------|------|
| **HTAP** | 同时支持在线事务处理（OLTP）和在线分析处理（OLAP） |
| **高扩展性** | 在线增减节点，数据自动重分布，线性扩展 |
| **高吞吐量** | 多节点并行计算，聚合查询性能随节点数线性提升 |
| **完全兼容** | 继承 DM8 的 SQL 语法和生态，应用无需任何改造 |
| **自动分片** | 支持 Hash / Range / List 分片策略，对应用透明 |

### 组件角色

| 组件 | 缩写 | 职责 |
|------|------|------|
| **计算节点** | SP (SQL Processor) | 接收客户端请求，SQL 解析、执行计划生成、子任务分发、结果归并 |
| **存储节点** | BP (Base Processor) | 实际存储数据分片，执行计算节点下发的本地子查询 |
| **元数据节点** | MP (Meta Processor) | 存储集群元数据（表结构、分片信息、节点拓扑） |

### 分片策略

```sql
-- Hash 分片（默认，数据均匀分布）
CREATE TABLE orders (
  order_id   INT,
  cust_id    INT,
  amount     DECIMAL(10,2)
) DISTRIBUTE BY HASH(order_id);

-- Range 分片（按范围分片，适合按时间范围查询的场景）
CREATE TABLE logs (
  log_id     INT,
  log_time   DATETIME,
  content    VARCHAR(4000)
) DISTRIBUTE BY RANGE(log_time) (
  VALUES LESS THAN ('2024-01-01') ON BP1,
  VALUES LESS THAN ('2024-07-01') ON BP2,
  VALUES LESS THAN (MAXVALUE)     ON BP3
);

-- 复制表（每个节点存全量，适合小表高频 JOIN）
CREATE TABLE dim_region (
  region_id  INT,
  name       VARCHAR(100)
) DISTRIBUTE BY REPLICATION;
```

### 使用场景

- 海量在线交易系统（如电商订单、支付流水）
- 需要实时分析能力的大数据场景（HTAP）
- 数据量达到 TB-PB 级，单机无法承载
- 需要弹性扩缩容的云原生环境

---

## 五、集群方案选型矩阵

| 场景需求 | DataWatch | RWC | DSC | DPC |
|---------|:---------:|:---:|:---:|:---:|
| 主备高可用 | ✅ 最佳 | ✅ | ✅ | ✅ |
| 自动读写分离 | ❌ | ✅ 最佳 | ❌ | ✅ |
| 多活（多节点同时读写） | ❌ | ❌ | ✅ 最佳 | ✅ |
| 弹性水平扩展 | ❌ | ❌ | ❌ | ✅ 最佳 |
| HTAP 混合负载 | ❌ | ❌ | ❌ | ✅ 最佳 |
| 应用代码改造量 | 无 | 无 | 无 | 无 |
| 硬件要求 | 普通 | 普通 | **必须共享存储** | 普通 |
| 部署复杂度 | ⭐ 低 | ⭐⭐ 中 | ⭐⭐⭐ 高 | ⭐⭐⭐⭐ 较高 |

---

## 六、集群部署通用要求

无论部署哪种集群方案，以下要求是所有集群类型共通的：

### 1. 独立心跳网络

```
要求：专用网络接口，≥ 1000Mbps 带宽

目的：
  - 集群节点间心跳检测（Heartbeat）
  - 缓存融合通信（DSC）
  - 数据同步/分片间通信

配置示例：
  eth0: 192.168.1.0/24   —— 业务网络
  eth1: 10.10.10.0/24    —— 心跳网络（专用，不与业务混用）
```

### 2. 各节点时间同步

```bash
# 所有节点必须时间一致，REDO 日志和应用依赖精确时间戳

# 方式一：NTP 服务
yum install -y ntp
systemctl enable ntpd
systemctl start ntpd
# 配置 /etc/ntp.conf 指向同一 NTP 服务器

# 方式二：chronyd（CentOS 7+/RHEL 7+ 推荐）
yum install -y chrony
systemctl enable chronyd
systemctl start chronyd
# 配置 /etc/chrony.conf 添加 server ntp.example.com iburst

# 验证时间同步
chronyc sources -v
# 或
ntpq -p

# 检查各节点时间偏差（应 < 1 秒）
for host in node1 node2 node3; do
  ssh $host "date '+%Y-%m-%d %H:%M:%S'"
done
```

### 3. 防火墙开放集群通信端口

```bash
# 达梦集群常用端口：
#   5236  - 数据库服务端口
#   5237  - 数据库 RPC 端口
#   5238  - DSC 集群 DMCSS 端口
#   5239  - DSC 集群 ASM 端口
#   9741  - DataWatch 守护进程端口
#   9000-9100 - DPC 集群内部通信端口

# firewalld 示例
firewall-cmd --permanent --add-port=5236-5239/tcp
firewall-cmd --permanent --add-port=9741/tcp
firewall-cmd --permanent --add-port=9000-9100/tcp
firewall-cmd --reload

# iptables 示例
iptables -A INPUT -p tcp --dport 5236:5239 -j ACCEPT
iptables -A INPUT -p tcp --dport 9741 -j ACCEPT
service iptables save
```

### 4. 磁盘调度算法设置为 deadline

```bash
# 原因：deadline 调度算法在数据库场景下 IO 延迟最低
# 适用：所有数据库节点的数据磁盘

# 查看当前调度算法
cat /sys/block/sda/queue/scheduler
# 典型输出：noop [cfq] deadline

# 临时设置（以 sda 为例）
echo deadline > /sys/block/sda/queue/scheduler

# 永久设置（修改 GRUB 启动参数）
# 编辑 /etc/default/grub，在 GRUB_CMDLINE_LINUX 增加：
#   elevator=deadline
# 然后：
grub2-mkconfig -o /boot/grub2/grub.cfg
# 重启生效
```

### 5. 其他系统级优化

```bash
# 关闭透明大页（避免内存碎片导致性能抖动）
echo never > /sys/kernel/mm/transparent_hugepage/enabled
echo never > /sys/kernel/mm/transparent_hugepage/defrag

# 永久生效，追加到 /etc/rc.d/rc.local：
echo 'echo never > /sys/kernel/mm/transparent_hugepage/enabled' >> /etc/rc.d/rc.local
echo 'echo never > /sys/kernel/mm/transparent_hugepage/defrag' >> /etc/rc.d/rc.local

# 关闭 SELinux（或设置为 permissive）
setenforce 0
sed -i 's/SELINUX=enforcing/SELINUX=disabled/g' /etc/selinux/config

# 修改 limits.conf 提高文件句柄和进程限制
cat >> /etc/security/limits.conf << EOF
dmdba  soft  nofile  65536
dmdba  hard  nofile  65536
dmdba  soft  nproc   65536
dmdba  hard  nproc   65536
EOF
```

### 部署前检查清单

| 检查项 | 命令 | 通过标准 |
|--------|------|---------|
| 操作系统版本 | `cat /etc/os-release` | 各节点版本一致 |
| CPU 架构 | `uname -m` | 各节点架构相同 |
| 磁盘空间 | `df -h /dm/data` | ≥ 数据预估量的 2 倍 |
| 内存大小 | `free -h` | ≥ 1GB（生产建议 ≥ 8GB） |
| 心跳网络延迟 | `ping -c 10 <对端心跳IP>` | 平均 < 1ms |
| 时间偏差 | `ntpdate -q <NTP服务器>` | < 1 秒 |
| 防火墙端口 | `nc -zv <对端IP> 5236` | 全部 open |
| 磁盘调度算法 | `cat /sys/block/sda/queue/scheduler` | 含 `deadline` 且为当前项 |
| 透明大页 | `cat /sys/kernel/mm/transparent_hugepage/enabled` | `[never]` |

---

## 补充：DMDPC 分布式集群

**DMDPC（DM Distributed Processing Cluster）** 是达梦的新一代分布式集群，采用"无共享对等架构"（Shared-Nothing），支持在线动态扩容到 PB 级，定位 HTAP 混合负载（事务 + 分析）。

### 与 DM MPP 的核心区别

| 维度 | DMDPC | DM MPP |
|------|-------|--------|
| 定位 | 在线事务 + 分析混合 | 大规模分析型 |
| 节点数 | 数十到数百 | 通常 < 100 |
| 扩容能力 | 在线动态增加节点 | 一般静态 |
| 适用场景 | 高可用 + 大数据量 | 数据仓库/报表 |
| 一致性 | 强一致 | 最终一致 |
| SQL 兼容 | 完整 DM8 语法 | 部分裁剪 |

### 关键组件

- **SP（Service Processor）**：全局事务协调器，负责全局锁与分布式事务
- **BP（Business Processor）**：业务执行节点，多副本部署，实际存储数据分片
- **MP（Metadata Processor）**：元数据管理节点，存储集群拓扑与分片信息
- **协调节点 CN**：对外提供服务入口，接收并分发客户端请求

### 部署架构

最小可用 3 节点：1 SP + 1 BP + 1 MP。生产建议：

- SP/MP 单独部署在专用机器
- BP 至少 3 副本，避免单点
- 心跳网络与业务网络分离（≥ 10Gbps）
- 全节点 NTP 时间同步，偏差 < 1 秒
- 关键参数 REDO 日志、归档、备份全部打开

### 适用场景

- 互联网高并发 OLTP（> 10 万 TPS）
- 海量数据在线分析（PB 级）
- 需要在线水平扩容的弹性业务
- 信创云原生环境（K8s + StatefulSet）

### 分片策略与查询路由

DMDPC 支持 Hash、Range、List 三种分片方式，副本数独立配置。SP 节点负责 SQL 解析与执行计划生成，查询自动下推到 BP 节点并行执行，最后 SP 归并结果返回。表关联（JOIN）会优先在同分片内本地化处理，跨分片 JOIN 通过 SP 协调两阶段执行。

### 性能与运维要点

- 分片键选择必须高基数、低倾斜，避免热点 BP
- 大表采用 Range 分片可按时间归档历史数据
- 复制表（Replicated Table）适合小维度表，规避跨分片 JOIN
- 在线扩容会触发数据重分布，建议业务低峰期执行
- MPP_STAT、SP_STAT 视图实时查看各 BP 节点负载

---

## 补充：MPP + DataWatch 二级高可用

在 MPP 集群内部，每个数据节点单独配置 DataWatch 主备，实现"MPP 集群级 HA + 节点级 HA"双保险架构。

典型架构：MPP 4 节点 ×（1 主 1 备）+ 集中监视器 dmmonitor + 第三方仲裁。任一节点的主库故障，DataWatch 秒级自动切换；任一节点机器整体故障，MPP 自动剔除故障节点并重分布查询，避免整个集群不可用。

该方案适合对 RTO（恢复时间目标）≤ 10 秒、RPO = 0 的金融、电信核心系统。

### 故障切换流程

1. dmmonitor 探测到主库心跳超时
2. 通知备库准备接管，关闭主库对外服务
3. 备库应用完所有 REDO 日志，提升为新主
4. 客户端连接自动重定向到新主
5. MPP 上层剔除故障节点，剩余节点继续提供服务

该方案相比单层 DataWatch 多消耗一倍的机器资源，但 RTO 与 RPO 同时达到金融级要求。

---

## 补充：K8s 容器化集群（信创云原生）

达梦提供了基于 Kubernetes Operator 的容器化方案，核心基于 StatefulSet 有状态工作负载，支持：

- 单实例 DM 数据库容器化部署
- DataWatch 主备集群 K8s 部署
- DSC 共享存储集群 K8s 部署（需云盘 / CSI 驱动支持）
- 自动故障恢复（Pod 漂移 + 持久卷数据不丢）
- Operator 自动化运维（备份、扩缩容、滚动升级）

生产建议：先在测试环境验证 1-2 个月，重点验证 CSI 存储 IO 性能、Pod 调度延迟、HPA 弹性策略、网络插件（CNI）与 DM 心跳的兼容性，再切生产。注意：K8s 节点时钟漂移超过 1 秒会导致 DataWatch 脑裂，必须部署 NTP + chronyd。

### StatefulSet 配置要点

- 每个 DM 实例对应一个 Pod，Pod 名固定（dm-cluster-0、dm-cluster-1）
- 使用 PVC 模板绑定持久卷，重建 Pod 数据不丢
- 配 PodDisruptionBudget（PDB）防止滚动升级时全部 Pod 同时被驱逐
- 使用 InitContainer 初始化 dm.ini 与 dmmal.ini
- Headless Service 提供稳定的 Pod DNS 名称供 DataWatch 心跳

### CSI 存储选型建议

- 生产推荐块存储（云盘 SSD / 本地 NVMe），IO 延迟 < 1ms
- DSC 集群需要 ReadWriteMany 模式的共享卷
- 备份存储用对象存储（OSS / S3 兼容）
- 避免使用 NFS，性能抖动会显著拖慢 Redo 写入

---

## 参考资料

- 达梦官方文档中心 - DMDPC 分布式集群：https://eco.dameng.com/document/dm/zh-cn/pm/dpc-introduction
- 达梦官方文档中心 - DM MPP 集群：https://eco.dameng.com/document/dm/zh-cn/pm/mpp
- 达梦 K8s Operator 开源仓库：https://github.com/DamengDB/dm-k8s-operator
- 墨天轮 - DMDPC 部署实战案例：https://www.modb.pro/db/topic/dpc-deployment
