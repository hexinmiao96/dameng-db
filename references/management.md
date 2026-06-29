# 达梦数据库表空间与用户管理

> 达梦数据库的表空间、用户与模式管理，涵盖创建、修改、授权与迁移最佳实践。

---

## 目录

- [表空间管理](#表空间管理)
  - [创建表空间](#创建表空间)
  - [修改表空间](#修改表空间)
  - [删除表空间](#删除表空间)
  - [查询表空间信息](#查询表空间信息)
  - [页大小的影响](#页大小的影响)
- [用户管理](#用户管理)
  - [创建用户](#创建用户)
  - [用户授权](#用户授权)
  - [密码策略](#密码策略)
  - [用户锁定与解锁](#用户锁定与解锁)
  - [修改与删除用户](#修改与删除用户)
- [模式（Schema）](#模式schema)
  - [模式与用户的关系](#模式与用户的关系)
  - [MySQL 到 DM 的迁移映射](#mysql-到-dm-的迁移映射)

---

## 表空间管理

表空间是数据库中最大的逻辑存储单元，数据库由一个或多个表空间组成。

DM 默认表空间：

| 表空间名 | 说明 |
|---------|------|
| `SYSTEM` | 系统表空间，存储数据字典和系统元数据 |
| `ROLL` | 回滚表空间，存储回滚记录 |
| `TEMP` | 临时表空间，存放排序、分组等临时数据 |
| `MAIN` | 主表空间，默认的用户数据存放位置 |
| `HMAIN` | HUGE 表空间（列存储） |

### 创建表空间

#### 基本语法

```sql
CREATE TABLESPACE "表空间名"
  DATAFILE '数据文件路径'
  SIZE 初始大小
  AUTOEXTEND ON NEXT 自动扩展步长 MAXSIZE 最大大小
  CACHE = NORMAL;  -- 缓存策略：NORMAL/READ/KEEP
```

#### 示例：创建普通表空间

```sql
-- 创建用户业务表空间
CREATE TABLESPACE "TEST"
  DATAFILE '/data/dmdata/DAMENG/TEST.DBF'
  SIZE 128
  AUTOEXTEND ON NEXT 100 MAXSIZE 10240
  CACHE = NORMAL;
```

**参数说明：**

| 参数 | 说明 | 单位 |
|------|------|------|
| `SIZE 128` | 数据文件初始大小 | MB |
| `AUTOEXTEND ON` | 启用自动扩展 | — |
| `NEXT 100` | 每次自动扩展的增量 | MB |
| `MAXSIZE 10240` | 数据文件最大大小（10GB） | MB |
| `CACHE = NORMAL` | 缓存策略（适用大部分场景） | — |

**缓存策略对比：**

| 策略 | 说明 | 适用场景 |
|------|------|---------|
| `NORMAL` | 正常缓存，LRU 淘汰 | 通用业务数据（推荐） |
| `KEEP` | 优先保留在缓冲区 | 高频访问的热数据 |
| `RECYCLE` | 尽快从缓冲区淘汰 | 较少访问的冷数据 |

#### 示例：创建加密表空间

```sql
CREATE TABLESPACE "TEST_SECURE"
  DATAFILE '/data/dmdata/DAMENG/TEST_SECURE.DBF'
  SIZE 128
  AUTOEXTEND ON NEXT 100 MAXSIZE 10240
  CACHE = NORMAL
  ENCRYPT WITH RC4;
```

**支持的加密算法：**

| 算法 | 安全性 | 性能影响 |
|------|--------|---------|
| `RC4` | 基础 | 低 |
| `AES128` | 中 | 中 |
| `AES192` | 高 | 中 |
| `AES256` | 最高 | 较高 |

> ⚠️ 加密表空间创建后不可解密，需妥善保管加密密钥。

#### 重要约束

**数据文件最小值：** `4096 × 页大小`

| 页大小 | 数据文件最小值 |
|--------|--------------|
| 4K | 16 MB |
| 8K | 32 MB |
| 16K | 64 MB |
| 32K | 128 MB |

> 例如：32K 页大小的数据库，数据文件最小为 128MB。

#### 创建多数据文件表空间

```sql
CREATE TABLESPACE "LARGE_DATA"
  DATAFILE '/data/dmdata/DAMENG/LARGE_DATA01.DBF' SIZE 10240,
  DATAFILE '/data/dmdata/DAMENG/LARGE_DATA02.DBF' SIZE 10240,
  DATAFILE '/data/dmdata/DAMENG/LARGE_DATA03.DBF' SIZE 10240
  AUTOEXTEND ON NEXT 500 MAXSIZE 102400
  CACHE = NORMAL;
```

### 修改表空间

#### 扩展数据文件

```sql
-- 修改自动扩展配置
ALTER TABLESPACE "TEST"
  DATAFILE '/data/dmdata/DAMENG/TEST.DBF'
  AUTOEXTEND ON NEXT 100 MAXSIZE 10240;
```

#### 增加数据文件

```sql
-- 为已有表空间添加新的数据文件
ALTER TABLESPACE "TEST"
  ADD DATAFILE '/data/dmdata/DAMENG/TEST02.DBF'
  SIZE 1024
  AUTOEXTEND ON NEXT 100 MAXSIZE 10240;
```

#### 手动扩展数据文件大小

```sql
-- 直接扩大数据文件到指定大小（不能缩小）
ALTER TABLESPACE "TEST"
  DATAFILE '/data/dmdata/DAMENG/TEST.DBF'
  RESIZE 5120;  -- 扩展到 5GB
```

#### 修改表空间状态

```sql
-- 设置表空间为只读
ALTER TABLESPACE "TEST" READONLY;

-- 设置表空间为读写
ALTER TABLESPACE "TEST" READWRITE;

-- 设置表空间为脱机（OFFLINE）
ALTER TABLESPACE "TEST" OFFLINE;

-- 设置表空间为联机（ONLINE）
ALTER TABLESPACE "TEST" ONLINE;
```

> ⚠️ SYSTEM、ROLL、TEMP 表空间不允许脱机或只读。

### 删除表空间

```sql
-- 删除空表空间（不含任何对象）
DROP TABLESPACE "TEST_EMPTY";

-- 删除表空间及其中的所有对象（级联删除）
DROP TABLESPACE "TEST" INCLUDING CONTENTS;

-- 删除表空间及数据文件
DROP TABLESPACE "TEST" INCLUDING CONTENTS AND DATAFILES;

-- 如果数据文件还在使用中，强制删除
DROP TABLESPACE "TEST" INCLUDING CONTENTS AND DATAFILES CASCADE CONSTRAINTS;
```

> 🛑 **危险操作！** `INCLUDING CONTENTS` 会删除表空间内所有表、索引、存储过程等对象，务必三思后行。

### 查询表空间信息

```sql
-- 查看所有表空间概览
SELECT
  NAME AS 表空间名,
  TYPE$ AS 类型,
  STATUS$ AS 状态,
  CAST(TOTAL_SIZE AS DECIMAL)/1024 AS 总大小MB,
  CAST(FREE_SIZE AS DECIMAL)/1024 AS 可用MB,
  CAST((TOTAL_SIZE - FREE_SIZE) AS DECIMAL)/CAST(TOTAL_SIZE AS DECIMAL) * 100 AS 使用率百分比
FROM V$TABLESPACE;

-- 查看数据文件详情
SELECT
  PATH AS 文件路径,
  CAST(TOTAL_SIZE AS DECIMAL)/1024 AS 总大小MB,
  CAST(FREE_SIZE AS DECIMAL)/1024 AS 可用大小MB,
  STATUS$ AS 状态,
  AUTO_EXTEND AS 是否自动扩展,
  MAX_SIZE AS 最大大小
FROM V$DATAFILE;

-- 查看表空间的空闲碎片
SELECT
  TABLESPACE_NAME,
  FILE_ID,
  BLOCK_ID,
  BLOCKS
FROM DBA_FREE_SPACE
WHERE TABLESPACE_NAME = 'TEST';

-- 查看特定用户使用的表空间
SELECT
  USERNAME AS 用户名,
  DEFAULT_TABLESPACE AS 默认表空间
FROM DBA_USERS;
```

#### 表空间使用率告警阈值

建议设置以下告警：
- **警告（Warning）：** 使用率 > 80%
- **严重（Critical）：** 使用率 > 90%
- **紧急（Emergency）：** 使用率 > 95%

### 页大小的影响

数据库初始化时确定的 `PAGE_SIZE` 直接影响表空间和字段定义的上限：

| 页大小 | VARCHAR 字段最大定义长度 | 每行非字段总长限制 | 数据文件最小/最大 |
|--------|------------------------|-------------------|-------------------|
| 4K | 1938 字节 | 2047 字节 | 16 MB / 8 TB |
| 8K | 3878 字节 | 4095 字节 | 32 MB / 16 TB |
| 16K | 8000 字节 | 8195 字节 | 64 MB / 32 TB |
| 32K | 8188 字节 | 16176 字节 | 128 MB / 64 TB |

> **注意：** `VARCHAR(1000)` 在 UTF-8 字符集下，一个中文字符占 3 字节，最多能存储约 333 个中文字符。务必根据实际业务计算字段长度需求。

---

## 用户管理

DM 数据库中有三类系统管理员：

| 管理员 | 角色 | 主要职责 |
|--------|------|---------|
| **SYSDBA** | 系统管理员 | 拥有所有系统管理权限，管理数据库对象 |
| **SYSAUDITOR** | 审计管理员 | 管理审计设置，查看审计记录 |
| **SYSSSO** | 安全管理员 | 管理安全策略（仅安全版） |

### 创建用户

#### 完整语法示例

```sql
CREATE USER "TEST"
  IDENTIFIED BY "<StrongPassword>"
  HASH WITH SHA512 SALT
  ENCRYPT BY "123456"
  DEFAULT TABLESPACE "TEST"
  DEFAULT INDEX TABLESPACE "TEST"
  LIMIT FAILED_LOGIN_ATTEMPS 3,
        PASSWORD_LOCK_TIME 1,
        PASSWORD_GRACE_TIME 10;
```

**参数详解：**

| 参数 | 说明 | 示例值 |
|------|------|--------|
| `IDENTIFIED BY` | 用户登录密码（需符合密码策略） | `"<StrongPassword>"` |
| `HASH WITH` | 密码哈希算法 | `SHA512 SALT`、`MD5 SALT` |
| `ENCRYPT BY` | 存储加密密钥 | `"<EncryptionKey>"` |
| `DEFAULT TABLESPACE` | 默认表空间（存放表数据） | `"TEST"` |
| `DEFAULT INDEX TABLESPACE` | 默认索引表空间 | `"TEST"` |
| `FAILED_LOGIN_ATTEMPS` | 允许的失败登录次数 | `3` |
| `PASSWORD_LOCK_TIME` | 超过失败次数后被锁定的天数 | `1` |
| `PASSWORD_GRACE_TIME` | 密码过期后的宽限期天数 | `10` |

#### 简化创建（推荐日常使用）

```sql
-- 最简创建方式
CREATE USER "APP_USER" IDENTIFIED BY "AppUser@2024";

-- 指定默认表空间
CREATE USER "BIZ_USER"
  IDENTIFIED BY "BizUser@2024"
  DEFAULT TABLESPACE "BIZ_DATA"
  DEFAULT INDEX TABLESPACE "BIZ_INDEX";
```

#### 创建只读用户

```sql
CREATE USER "READONLY_USER"
  IDENTIFIED BY "Readonly@2024"
  DEFAULT TABLESPACE "REPORT_DATA";

-- 授予只读权限
GRANT SELECT ANY TABLE TO "READONLY_USER";
```

### 用户授权

DM 数据库的角色遵循**最小权限原则**，不建议直接给业务用户授予 `DBA` 角色。

#### 系统预定义角色

| 角色 | 说明 | 包含的权限 |
|------|------|-----------|
| `DBA` | 数据库管理员 | 所有权限（**禁止授予普通用户！**） |
| `PUBLIC` | 公共角色 | 数据操纵（SELECT/INSERT/UPDATE/DELETE）及执行权限 |
| `RESOURCE` | 资源角色 | 创建表、视图、存储过程、函数、序列等 |
| `SOI` | 系统对象查询 | 查询静态系统表（如 SYSOBJECTS） |
| `SVI` | 系统视图查询 | 查询系统视图（基础信息视图） |
| `VTI` | 动态视图查询 | 查询动态性能视图（V$ 开头的视图） |

#### 批量授权（开发/测试环境）

```sql
-- 一次性授予常用角色（注意：RESOURCE 已包含 PUBLIC）
GRANT "RESOURCE", "SOI", "SVI", "VTI" TO "TEST";
```

#### 细粒度授权（生产环境推荐）

```sql
-- 1. 数据操作权限
GRANT "PUBLIC" TO "TEST";

-- 2. 对象创建权限（创建表、视图、过程、序列等）
GRANT "RESOURCE" TO "TEST";

-- 3. 系统动态性能视图查询（性能监控需要）
GRANT "VTI" TO "TEST";

-- 4. 系统表对象查询（排错和开发调试需要）
GRANT "SOI" TO "TEST";

-- 5. 系统视图查询
GRANT "SVI" TO "TEST";
```

#### 对象级授权（最严谨）

```sql
-- 只对特定表授权
GRANT SELECT, INSERT, UPDATE, DELETE ON "SCHEMA"."TABLE_NAME" TO "TEST";

-- 只授权查询
GRANT SELECT ON "SCHEMA"."TABLE_NAME" TO "TEST";

-- 授权执行存储过程
GRANT EXECUTE ON "SCHEMA"."PROCEDURE_NAME" TO "TEST";

-- 授权使用序列
GRANT SELECT ON "SCHEMA"."SEQ_NAME" TO "TEST";

-- 查看用户的系统权限
SELECT * FROM DBA_SYS_PRIVS WHERE GRANTEE = 'TEST';

-- 查看用户的角色
SELECT * FROM DBA_ROLE_PRIVS WHERE GRANTEE = 'TEST';

-- 查看用户的表级权限
SELECT * FROM DBA_TAB_PRIVS WHERE GRANTEE = 'TEST';
```

#### 回收权限

```sql
-- 回收角色
REVOKE "RESOURCE" FROM "TEST";

-- 回收表级权限
REVOKE INSERT, UPDATE, DELETE ON "SCHEMA"."TABLE_NAME" FROM "TEST";

-- 回收所有权限（慎用）
REVOKE ALL PRIVILEGES FROM "TEST";
```

### 密码策略

DM 通过 `PWD_POLICY` 系统参数控制密码复杂度。

#### PWD_POLICY 位掩码

| 位值 | 策略 | 说明 |
|------|------|------|
| **1** | 禁止用户名 | 密码不能与用户名相同 |
| **2** | 最小长度 | 密码长度不能小于 `PWD_MIN_LEN`（默认 9） |
| **4** | 大小写字母 | 至少包含一个大写字母和一个小写字母 |
| **8** | 数字 | 至少包含一个数字 |
| **16** | 标点符号 | 至少包含一个标点符号（如 `!@#$%^&*`） |

默认值 **15** = 1 + 2 + 4 + 8（即要求：不能与用户名相同 + 最小长度 + 大小写字母 + 数字）

#### 查询与修改密码策略

```sql
-- 查询当前密码策略
SELECT NAME, VALUE, DESCRIPTION
FROM V$PARAMETER
WHERE NAME LIKE '%PWD%';

-- 修改密码策略（增加标点符号要求：15+16=31）
ALTER SYSTEM SET 'PWD_POLICY' = 31 BOTH;

-- 修改最小密码长度
ALTER SYSTEM SET 'PWD_MIN_LEN' = 12 BOTH;

-- 修改密码有效期（天），0 表示永不过期
ALTER USER "TEST" LIMIT PASSWORD_LIFE_TIME 90;

-- 修改密码重用限制（多少次不能重复）
ALTER USER "TEST" LIMIT PASSWORD_REUSE_TIME 5;

-- 修改密码可重用最早时间（天）
ALTER USER "TEST" LIMIT PASSWORD_REUSE_MAX 10;
```

#### 密码过期处理

当用户密码过期后：

```sql
-- 管理员重置用户密码
ALTER USER "TEST" IDENTIFIED BY "NewPass@2024";

-- 用户在宽限期内登录后可自行修改密码
-- 在 disql 登录后会提示输入新密码
```

### 用户锁定与解锁

#### 锁定和解锁用户

```sql
-- 解锁被锁定的用户
ALTER USER "TEST" ACCOUNT UNLOCK;

-- 手动锁定用户
ALTER USER "TEST" ACCOUNT LOCK;

-- 查看用户锁定状态
SELECT USERNAME, ACCOUNT_STATUS, LOCK_DATE
FROM DBA_USERS
WHERE USERNAME = 'TEST';
```

**ACCOUNT_STATUS 取值：**

| 状态值 | 含义 |
|--------|------|
| `OPEN` | 正常 |
| `LOCKED` | 已锁定（登录失败超限） |
| `LOCKED(TIMED)` | 定时锁定 |
| `EXPIRED` | 密码已过期 |
| `EXPIRED(GRACE)` | 密码过期但处于宽限期 |
| `EXPIRED & LOCKED(TIMED)` | 过期且定时锁定 |

#### 修改用户锁定策略

```sql
-- 修改失败登录限制
ALTER USER "TEST" LIMIT
  FAILED_LOGIN_ATTEMPS 5,     -- 允许 5 次失败尝试
  PASSWORD_LOCK_TIME 3,       -- 超限后锁定 3 天
  PASSWORD_GRACE_TIME 10;     -- 密码过期后宽限 10 天

-- 查询用户资源限制信息
SELECT * FROM DBA_PROFILES
WHERE PROFILE = 'DEFAULT' AND RESOURCE_NAME LIKE 'FAILED%';

SELECT * FROM DBA_USERS
WHERE USERNAME = 'TEST';
```

#### 查看当前锁定的用户

```sql
SELECT
  USERNAME,
  ACCOUNT_STATUS,
  LOCK_DATE
FROM DBA_USERS
WHERE ACCOUNT_STATUS LIKE '%LOCKED%';
```

### 修改与删除用户

```sql
-- 修改用户默认表空间
ALTER USER "TEST" DEFAULT TABLESPACE "NEW_TBS";

-- 修改用户密码
ALTER USER "TEST" IDENTIFIED BY "NewPass@2024";

-- 删除用户
DROP USER "TEST";

-- 级联删除用户及其所有对象
DROP USER "TEST" CASCADE;

-- 如果用户已连接，强制删除
DROP USER "TEST" CASCADE FORCE;
```

> ⚠️ `DROP USER ... CASCADE` 是毁灭性操作，会删除该用户创建的所有表、视图、存储过程等，务必确认！

---

## 模式（Schema）

### 模式与用户的关系

在 DM 数据库中，**模式 = 用户**，这是与其他数据库最大的区别之一。

- 每个用户创建时会自动生成一个**同名模式**
- 一个用户**有且仅有一个**模式
- 模式名称 = 用户名称
- 用户创建的所有对象（表、视图、存储过程、函数、序列等）默认都在自己的模式下

```
用户 "TEST" 创建后：
  → 自动生成模式 "TEST"
    → CREATE TABLE "TEST"."TABLE1" (...)   ← 表在 TEST 模式下
    → CREATE TABLE "TABLE1" (...)           ← 等同上面（默认当前用户模式）
```

#### 跨模式访问

```sql
-- 查询其他模式下的表（需要被授权）
SELECT * FROM "OTHER_SCHEMA"."TABLE_NAME";

-- 当前用户在自己的模式下创建表
CREATE TABLE MY_TABLE (ID INT);

-- 等价于
CREATE TABLE "TEST"."MY_TABLE" (ID INT);
```

#### 设置当前模式

```sql
-- 查看当前模式
SELECT SYS_CONTEXT('USERENV', 'CURRENT_SCHEMA');

-- 切换当前模式（需要有该模式的权限）
SET SCHEMA "OTHER_SCHEMA";

-- 查看当前用户的所有模式（实际就是一个）
SELECT USERNAME FROM USER_USERS;
```

### MySQL 到 DM 的迁移映射

MySQL 和 DM 在数据库管理单元上有本质区别：

| 概念 | MySQL | 达梦 DM |
|------|-------|---------|
| 数据库 | 一个实例可包含多个独立数据库 | 一个实例 = 一个数据库 |
| 模式/用户 | `DATABASE` = 独立命名空间 | `USER/SCHEMA` = 独立命名空间 |
| 跨库访问 | `db.table` 方式 | `schema.table` 方式 |

**迁移策略：**

```
MySQL:  单实例 → 多数据库 → 每库有多张表
            ↓ 迁移映射 ↓
DM:     单实例 → 单数据库 → 多用户/模式 → 每用户代表一个"库"

即：
MySQL 的 database1        → DM 的 user "DB1" + 表空间 "DB1_DATA"
MySQL 的 database2        → DM 的 user "DB2" + 表空间 "DB2_DATA"
MySQL 的 db1.users 表     → DM 的 "DB1"."USERS" 表
MySQL 的 SELECT ... FROM db1.users  → DM 的 SELECT ... FROM "DB1"."USERS"
```

**迁移操作步骤：**

```sql
-- 1. 为每个 MySQL 数据库创建独立表空间
CREATE TABLESPACE "DB1_DATA"
  DATAFILE '/data/dmdata/DAMENG/DB1_DATA.DBF'
  SIZE 128 AUTOEXTEND ON NEXT 100 MAXSIZE 10240;

CREATE TABLESPACE "DB2_DATA"
  DATAFILE '/data/dmdata/DAMENG/DB2_DATA.DBF'
  SIZE 128 AUTOEXTEND ON NEXT 100 MAXSIZE 10240;

-- 2. 为每个 MySQL 数据库创建独立用户（模式）
CREATE USER "DB1" IDENTIFIED BY "Db1@2024"
  DEFAULT TABLESPACE "DB1_DATA";

CREATE USER "DB2" IDENTIFIED BY "Db2@2024"
  DEFAULT TABLESPACE "DB2_DATA";

-- 3. 授权
GRANT "RESOURCE", "SOI", "SVI", "VTI" TO "DB1";
GRANT "RESOURCE", "SOI", "SVI", "VTI" TO "DB2";

-- 4. 应用连接字符串中的 schema 需要指定为对应用户名
-- JDBC: jdbc:dm://host:5236?schema=DB1
```

> **关键提醒：** MySQL 的 `USE database_name;` 在 DM 中用 `SET SCHEMA "SCHEMA_NAME";` 替代。
