# 应用开发接口

> **重要提示：所有驱动版本必须与数据库服务器版本保持一致！**

---

## Python (dmPython)

dmPython 是达梦官方提供的 Python 数据库接口，遵循 Python DB-API 2.0 规范。

### 安装前提

| 前提条件 | 说明 |
|----------|------|
| `DM_HOME` | 环境变量，指向达梦数据库安装目录 |
| `LD_LIBRARY_PATH` | 需包含 dpi 动态库所在路径（`$DM_HOME/bin`） |
| `gcc` | C 编译器，用于编译 dmPython 扩展 |
| `python-devel` | Python 开发头文件 |

**安装步骤：**

```bash
# 1. 设置环境变量
export DM_HOME=/dm8
export LD_LIBRARY_PATH=$DM_HOME/bin:$LD_LIBRARY_PATH

# 2. 安装依赖
yum install -y gcc python3-devel

# 3. 进入 dmPython 源码目录编译安装
cd $DM_HOME/drivers/python/dmPython/
python3 setup.py install
```

### 基础操作

#### 连接数据库

```python
import dmPython

# 建立连接
conn = dmPython.connect(
    user='SYSDBA',
    password='*****',
    server='localhost',
    port=5236
)
cursor = conn.cursor()
```

#### 增删改查

```python
import dmPython

conn = dmPython.connect(user='SYSDBA', password='*****', server='localhost', port=5236)
cursor = conn.cursor()

# -------------------- 插入数据 --------------------
cursor.execute(
    "INSERT INTO PRODUCTION.PRODUCT_CATEGORY(NAME) VALUES('语文'), ('数学'), ('英语')"
)
conn.commit()
print(f"插入行数: {cursor.rowcount}")

# -------------------- 删除数据 --------------------
cursor.execute("DELETE FROM PRODUCTION.PRODUCT_CATEGORY WHERE NAME='数学'")
conn.commit()
print(f"删除行数: {cursor.rowcount}")

# -------------------- 更新数据 --------------------
cursor.execute(
    "UPDATE PRODUCTION.PRODUCT_CATEGORY SET NAME='英语-新课标' WHERE NAME='英语'"
)
conn.commit()
print(f"更新行数: {cursor.rowcount}")

# -------------------- 查询数据 --------------------
cursor.execute("SELECT NAME FROM PRODUCTION.PRODUCT_CATEGORY")
res = cursor.fetchall()
for row in res:
    print(row)

# 获取单行
cursor.execute("SELECT NAME FROM PRODUCTION.PRODUCT_CATEGORY WHERE ROWNUM=1")
row = cursor.fetchone()
print(row)
```

#### 绑定变量（参数化查询）

dmPython 使用 `?` 作为占位符：

```python
import dmPython

conn = dmPython.connect(user='SYSDBA', password='*****', server='localhost', port=5236)
cursor = conn.cursor()

# 单参数绑定
cursor.execute(
    "INSERT INTO PRODUCTION.PRODUCT_CATEGORY(NAME) VALUES(?)",
    ('物理',)
)

# 多参数绑定
cursor.execute(
    "INSERT INTO PRODUCTION.PRODUCT_CATEGORY(ID, NAME) VALUES(?, ?)",
    (100, '化学')
)

# 查询绑定
cursor.execute(
    "SELECT * FROM PRODUCTION.PRODUCT_CATEGORY WHERE NAME LIKE ?",
    ('%物%',)
)
results = cursor.fetchall()

conn.commit()
conn.close()
```

#### 大字段操作（BLOB / CLOB）

```python
import dmPython

conn = dmPython.connect(user='SYSDBA', password='*****', server='localhost', port=5236)
cursor = conn.cursor()

# -------------------- 写入 BLOB/CLOB --------------------
with open('/path/to/image.jpg', 'rb') as f:
    bvalue = f.read()
cvalue = '这是一段很长的文本内容...' * 1000

cursor.execute(
    'INSERT INTO PRODUCTION.BIG_DATA VALUES(?, ?)',
    (bvalue, cvalue)
)
conn.commit()

# -------------------- 读取 BLOB/CLOB --------------------
cursor.execute('SELECT * FROM PRODUCTION.BIG_DATA')
row = cursor.fetchone()
(blob_data, clob_data) = row

print(f"BLOB 大小: {len(blob_data)} bytes")
print(f"CLOB 内容前100字符: {clob_data[:100]}")

conn.close()
```

#### 事务管理

```python
import dmPython

conn = dmPython.connect(user='SYSDBA', password='*****', server='localhost', port=5236)
conn.autocommit(False)  # 关闭自动提交

try:
    cursor = conn.cursor()
    cursor.execute("INSERT INTO T1 VALUES(1)")
    cursor.execute("INSERT INTO T2 VALUES(2)")
    conn.commit()
    print("事务提交成功")
except Exception as e:
    conn.rollback()
    print(f"事务回滚: {e}")
finally:
    conn.close()
```

### 多线程使用

每个线程必须创建独立的连接对象：

```python
import dmPython
import _thread
import time

def worker(name, delay):
    """每个线程独立连接"""
    conn = dmPython.connect(
        user='SYSDBA',
        password='*****',
        server='localhost',
        port=5236
    )
    try:
        cursor = conn.cursor()
        for i in range(3):
            cursor.execute(
                "INSERT INTO PRODUCTION.PRODUCT_CATEGORY(NAME) VALUES(?)",
                (f"{name}-{i}",)
            )
            conn.commit()
            time.sleep(delay)
        print(f"{name} 执行完毕")
    finally:
        conn.close()

# 启动多线程
for i in range(5):
    _thread.start_new_thread(worker, (f"Thread-{i}", 1))

time.sleep(10)  # 等待子线程完成
```

### Trace 跟踪

用于调试和问题排查：

**配置方法：** 在 `dm_svc.conf` 文件中设置：

```ini
# dm_svc.conf
DPI_TRACE=(1)
```

**生成的日志文件：**
- `dmPython_trace.log` — dmPython 层面的调用跟踪
- `dpi_trace.log` — DPI 接口层的详细跟踪

**日志级别配置：**

```ini
# 可选跟踪级别
DPI_TRACE=(0)   # 关闭跟踪
DPI_TRACE=(1)   # 基本跟踪
DPI_TRACE=(2)   # 详细跟踪（含参数值）
DPI_TRACE=(3)   # 最详细跟踪（含内存分配）
```

### 常见问题排查

| 错误信息 | 原因 | 解决方案 |
|----------|------|----------|
| `permission denied` | 安装权限不足 | 使用 `root` 用户安装 |
| `cannot locate Dameng` | 环境变量缺失 | 设置 `DM_HOME` 和 `LD_LIBRARY_PATH` |
| `unable to execute gcc` | gcc 未安装 | `yum install gcc*` |
| `Python.h: No such file` | python-devel 缺失 | `yum install python3-devel` |
| Windows 导入 `dmPython` 失败 | DLL 未找到 | 复制 `dpi` 相关 dll 到 Python 安装目录或 `C:\Windows\System32` |
| `DPI-10002: 连接失败` | 服务未启动或端口错误 | 检查 DM 服务状态和端口号 |
| 字符集乱码 | 字符集不匹配 | 连接时指定 `unicode=True` 参数 |

---

## Java (JDBC)

### 驱动获取

驱动 JAR 包位于达梦安装目录：`$DM_HOME/drivers/jdbc/`

| 文件 | 说明 |
|------|------|
| `DmJdbcDriver18.jar` | JDK 1.8 版本 |
| `DmJdbcDriverX.jar` | 其他 JDK 版本 |

### 连接数据库

**基础 URL 格式：**

```
jdbc:dm://host:port
```

**兼容模式 URL：**

```
jdbc:dm://host:5236?compatibleMode=oracle
```

**完整连接示例：**

```java
import java.sql.*;

public class DmJdbcExample {
    public static void main(String[] args) {
        // JDBC 驱动类（可省略，自动注册）
        String driver = "dm.jdbc.driver.DmDriver";
        String url = "jdbc:dm://localhost:5236";
        String user = "SYSDBA";
        String password = "*****";

        Connection conn = null;
        Statement stmt = null;
        ResultSet rs = null;

        try {
            // 1. 加载驱动
            Class.forName(driver);

            // 2. 建立连接
            conn = DriverManager.getConnection(url, user, password);

            // 3. 执行查询
            stmt = conn.createStatement();
            rs = stmt.executeQuery("SELECT NAME FROM PRODUCTION.PRODUCT_CATEGORY");

            while (rs.next()) {
                System.out.println(rs.getString("NAME"));
            }
        } catch (Exception e) {
            e.printStackTrace();
        } finally {
            // 4. 释放资源
            try { if (rs != null) rs.close(); } catch (Exception e) {}
            try { if (stmt != null) stmt.close(); } catch (Exception e) {}
            try { if (conn != null) conn.close(); } catch (Exception e) {}
        }
    }
}
```

### PreparedStatement（预编译）

```java
// 防止 SQL 注入，提高执行效率
String sql = "INSERT INTO PRODUCTION.PRODUCT_CATEGORY(NAME) VALUES(?)";
PreparedStatement pstmt = conn.prepareStatement(sql);
pstmt.setString(1, "物理");
pstmt.executeUpdate();

// 批量操作
pstmt.setString(1, "化学");
pstmt.addBatch();
pstmt.setString(1, "生物");
pstmt.addBatch();
int[] results = pstmt.executeBatch();
```

### 连接池配置（HikariCP 示例）

```java
// Maven: com.zaxxer:HikariCP
import com.zaxxer.hikari.HikariConfig;
import com.zaxxer.hikari.HikariDataSource;

HikariConfig config = new HikariConfig();
config.setJdbcUrl("jdbc:dm://localhost:5236");
config.setUsername("SYSDBA");
config.setPassword("*****");
config.setDriverClassName("dm.jdbc.driver.DmDriver");
config.setMaximumPoolSize(20);
config.setMinimumIdle(5);

HikariDataSource dataSource = new HikariDataSource(config);
```

### 框架支持

达梦 JDBC 驱动兼容以下主流 Java 框架：

| 框架 | 说明 |
|------|------|
| **Hibernate** | 配置 `hibernate.dialect` 为 `org.hibernate.dialect.DmDialect` |
| **MyBatis** | 直接使用 JDBC 驱动，无需额外方言 |
| **MyBatis-Plus** | 同上，注意大小写兼容 |
| **Spring / Spring Boot** | 配置 DataSource 即可 |
| **iBatis** | 老版本 ORM，兼容 |
| **MyCat** | 数据库中间件，支持达梦节点 |
| **Sharding-JDBC** | 分库分表中间件 |
| **Querydsl** | 类型安全的查询框架 |

**MyBatis 大小写处理：**

```xml
<!-- 方式一：数据库层面设置大小写不敏感 -->
<!-- DM 参数：CASE_SENSITIVE=0，重启生效 -->

<!-- 方式二：迁移时取消"保持大小写"勾选 -->
<!-- 在 DTS 工具中配置 -->
```

**Spring Boot 配置示例：**

```yaml
spring:
  datasource:
    driver-class-name: dm.jdbc.driver.DmDriver
    url: jdbc:dm://localhost:5236
    username: SYSDBA
    password: *****
```

---

## C/C++ (ODBC)

### Linux ODBC 配置

#### odbcinst.ini — 驱动注册

```ini
# /etc/odbcinst.ini
[DM8 ODBC DRIVER]
Description = DM8 ODBC Driver
Driver      = /dm8/bin/libdodbc.so
# 64位系统注意使用 libdodbc.so
```

#### odbc.ini — 数据源配置

```ini
# /etc/odbc.ini
[DM8]
DRIVER   = DM8 ODBC DRIVER
SERVER   = localhost
UID      = SYSDBA
PWD      = *****
TCP_PORT = 5236
DATABASE = DM8
```

#### 验证配置

```bash
# 测试 ODBC 连接
isql -v DM8 SYSDBA *****
```

### 核心 API 流程

```
SQLAllocHandle(ENV)          — 分配环境句柄
    ↓
SQLSetEnvAttr               — 设置 ODBC 版本
    ↓
SQLAllocHandle(DBC)          — 分配连接句柄
    ↓
SQLConnect / SQLDriverConnect — 建立连接
    ↓
SQLAllocHandle(STMT)         — 分配语句句柄
    ↓
┌─────────────────────────┐
│ 直接执行：               │
│   SQLExecDirect          │
│                          │
│ 预处理执行：              │
│   SQLPrepare             │
│   SQLBindParameter       │
│   SQLExecute             │
└─────────────────────────┘
    ↓
SQLBindCol + SQLFetch       — 绑定列并获取结果
    ↓
SQLPutData / SQLGetData     — 大字段读写（80KB/块）
    ↓
SQLFreeHandle(STMT)         — 释放语句句柄
    ↓
SQLDisconnect               — 断开连接
    ↓
SQLFreeHandle(DBC)          — 释放连接句柄
    ↓
SQLFreeHandle(ENV)          — 释放环境句柄
```

### C 语言完整示例

```c
#include <stdio.h>
#include <sql.h>
#include <sqlext.h>

int main() {
    SQLHENV henv;
    SQLHDBC hdbc;
    SQLHSTMT hstmt;
    SQLRETURN ret;
    SQLCHAR connStr[] = "DSN=DM8;UID=SYSDBA;PWD=*****";

    // 1. 分配环境句柄
    SQLAllocHandle(SQL_HANDLE_ENV, SQL_NULL_HANDLE, &henv);
    SQLSetEnvAttr(henv, SQL_ATTR_ODBC_VERSION, (SQLPOINTER)SQL_OV_ODBC3, 0);

    // 2. 分配连接句柄
    SQLAllocHandle(SQL_HANDLE_DBC, henv, &hdbc);

    // 3. 连接数据库
    ret = SQLDriverConnect(hdbc, NULL, connStr, SQL_NTS, NULL, 0, NULL, SQL_DRIVER_COMPLETE);
    if (ret != SQL_SUCCESS && ret != SQL_SUCCESS_WITH_INFO) {
        printf("连接失败\n");
        return 1;
    }

    // 4. 分配语句句柄
    SQLAllocHandle(SQL_HANDLE_STMT, hdbc, &hstmt);

    // 5. 执行查询
    SQLExecDirect(hstmt, (SQLCHAR*)"SELECT NAME FROM PRODUCTION.PRODUCT_CATEGORY", SQL_NTS);

    // 6. 绑定列并获取数据
    SQLCHAR name[256];
    SQLLEN indicator;
    SQLBindCol(hstmt, 1, SQL_C_CHAR, name, sizeof(name), &indicator);

    while (SQLFetch(hstmt) == SQL_SUCCESS) {
        printf("NAME: %s\n", name);
    }

    // 7. 清理资源
    SQLFreeHandle(SQL_HANDLE_STMT, hstmt);
    SQLDisconnect(hdbc);
    SQLFreeHandle(SQL_HANDLE_DBC, hdbc);
    SQLFreeHandle(SQL_HANDLE_ENV, henv);

    return 0;
}
```

### 预处理参数绑定

```c
// 预处理插入
SQLPrepare(hstmt, (SQLCHAR*)"INSERT INTO T VALUES(?, ?)", SQL_NTS);

SQLINTEGER id = 100;
SQLCHAR name[64] = "test";
SQLLEN id_ind = 0, name_ind = SQL_NTS;

SQLBindParameter(hstmt, 1, SQL_PARAM_INPUT, SQL_C_LONG, SQL_INTEGER, 0, 0, &id, 0, &id_ind);
SQLBindParameter(hstmt, 2, SQL_PARAM_INPUT, SQL_C_CHAR, SQL_VARCHAR, 0, 0, name, sizeof(name), &name_ind);

SQLExecute(hstmt);
```

### 大字段处理

```c
// BLOB 写入（SQLPutData，每次最多 80KB）
SQLPrepare(hstmt, (SQLCHAR*)"INSERT INTO BIG_DATA VALUES(?)", SQL_NTS);
SQLBindParameter(hstmt, 1, SQL_PARAM_INPUT, SQL_C_BINARY, SQL_LONGVARBINARY,
                 0, 0, NULL, 0, NULL);
SQLExecute(hstmt);

// 分块写入
char buffer[81920];  // 80KB
while (fread(buffer, 1, sizeof(buffer), fp) > 0) {
    SQLPutData(hstmt, buffer, bytesRead);
}

// BLOB 读取（SQLGetData，每次最多 80KB）
SQLExecDirect(hstmt, (SQLCHAR*)"SELECT DATA FROM BIG_DATA", SQL_NTS);
SQLFetch(hstmt);

char buf[81920];
SQLLEN len;
while (SQLGetData(hstmt, 1, SQL_C_BINARY, buf, sizeof(buf), &len) == SQL_SUCCESS) {
    fwrite(buf, 1, len, outfp);
}
```

---

## PHP

### 版本选择

| PHP 版本类型 | 适用场景 | SAPI 模式 |
|-------------|----------|-----------|
| **TS（线程安全）** | Apache + mod_php | ISAPI |
| **NTS（非线程安全）** | Nginx / IIS + PHP-FPM | FAST-CGI |

**判断方法：** 查看 `phpinfo()` 输出中的 `Thread Safety` 字段：
- `enabled` → TS 版本
- `disabled` → NTS 版本

### 驱动安装

#### Linux 安装

```bash
# 1. 找到 PHP 扩展目录
php -i | grep extension_dir
# 输出示例：/usr/lib64/php/modules/

# 2. 从 DM 安装目录复制驱动
cp $DM_HOME/drivers/php/libphp74_dm.so /usr/lib64/php/modules/
cp $DM_HOME/drivers/php/php74_pdo_dm.so /usr/lib64/php/modules/

# 3. 编辑 php.ini 添加扩展
# extension=libphp74_dm.so
# extension=php74_pdo_dm.so

echo "extension=libphp74_dm.so" >> /etc/php.ini
echo "extension=php74_pdo_dm.so" >> /etc/php.ini

# 4. 重启 PHP-FPM 或 Web 服务
systemctl restart php-fpm
# 或
systemctl restart httpd

# 5. 验证
php -m | grep dm
```

#### Windows 安装

```cmd
REM 1. 复制驱动 DLL
copy D:\dmdbms\drivers\php\php74_dm.dll C:\php\ext\
copy D:\dmdbms\drivers\php\php74_pdo_dm.dll C:\php\ext\

REM 2. 复制 bin 目录下的 16 个依赖 DLL
copy D:\dmdbms\bin\*.dll C:\Windows\System32\
REM 64位系统额外复制到 SysWOW64
copy D:\dmdbms\bin\*.dll C:\Windows\SysWOW64\

REM 3. 修改 php.ini 添加
; extension=php74_dm.dll
; extension=php74_pdo_dm.dll

REM 4. 重启 IIS 或 Apache
```

### 核心 API

#### 连接与基本操作

```php
<?php
// 建立连接
$link = dm_connect("localhost:5236", "SYSDBA", "*****");
if (!$link) {
    die("连接失败: " . dm_error());
}

// 设置模式
dm_exec($link, "SET SCHEMA PRODUCTION");

// 执行查询
$result = dm_exec($link, "SELECT * FROM PRODUCT_CATEGORY");
while ($row = dm_fetch_array($result)) {
    print_r($row);
}

// 释放结果集
dm_free_result($result);

// 关闭连接
dm_close($link);
?>
```

#### 预处理语句（参数绑定）

```php
<?php
$link = dm_connect("localhost:5236", "SYSDBA", "*****");

// 准备语句（使用命名参数 :v1, :v2）
$stmt = dm_prepare($link, "INSERT INTO PRODUCT_CATEGORY VALUES(:v1, :v2)");

// 执行（传入参数数组）
dm_execute($stmt, ["value1", "value2"]);
dm_execute($stmt, ["value3", "value4"]);

dm_close($link);
?>
```

#### PDO 方式

```php
<?php
try {
    $pdo = new PDO("dm:host=localhost;port=5236", "SYSDBA", "*****");
    $pdo->setAttribute(PDO::ATTR_ERRMODE, PDO::ERRMODE_EXCEPTION);

    // 查询
    $stmt = $pdo->query("SELECT * FROM PRODUCTION.PRODUCT_CATEGORY");
    while ($row = $stmt->fetch(PDO::FETCH_ASSOC)) {
        print_r($row);
    }

    // 预处理
    $stmt = $pdo->prepare("INSERT INTO PRODUCTION.PRODUCT_CATEGORY(NAME) VALUES(?)");
    $stmt->execute(['物理']);

} catch (PDOException $e) {
    echo "错误: " . $e->getMessage();
}
?>
```

#### 事务处理

```php
<?php
$link = dm_connect("localhost:5236", "SYSDBA", "*****");

dm_autocommit($link, false);  // 关闭自动提交

try {
    dm_exec($link, "INSERT INTO T1 VALUES(1)");
    dm_exec($link, "INSERT INTO T2 VALUES(2)");
    dm_commit($link);
    echo "事务提交";
} catch (Exception $e) {
    dm_rollback($link);
    echo "事务回滚: " . $e->getMessage();
}

dm_close($link);
?>
```

---

## Go

### 方式一：ODBC 方式

**依赖包：**

```bash
go get github.com/alexbrainman/odbc
go get github.com/go-xorm/xorm
```

**连接串格式：**

```
driver={DM8 ODBC DRIVER};server=192.0.2.100:5236;database=DM8;uid=SYSDBA;pwd=***;charset=utf8
```

**示例代码：**

```go
package main

import (
    "database/sql"
    "fmt"
    _ "github.com/alexbrainman/odbc"
)

func main() {
    connStr := "driver={DM8 ODBC DRIVER};server=localhost:5236;database=DM8;uid=SYSDBA;pwd=***;charset=utf8"

    db, err := sql.Open("odbc", connStr)
    if err != nil {
        panic(err)
    }
    defer db.Close()

    rows, err := db.Query("SELECT NAME FROM PRODUCTION.PRODUCT_CATEGORY")
    if err != nil {
        panic(err)
    }
    defer rows.Close()

    for rows.Next() {
        var name string
        rows.Scan(&name)
        fmt.Println(name)
    }
}
```

### 方式二：GORM 方言

驱动位于达梦安装目录 `drivers/go/` 下。

#### GORM V1 方言

```go
// 解压 gorm_v1_dialect.zip 到项目
// 注意：仅支持 CASE_SENSITIVE=N

import (
    "github.com/jinzhu/gorm"
    _ "dameng/db"                        // 达梦驱动
    _ "your_project/dialects/dm"         // GORM V1 方言
)

db, err := gorm.Open("dm", "dm://SYSDBA:***@localhost:5236")
```

#### GORM V2 方言

```go
// 解压 gorm_v2_dialect.zip 到项目

import (
    "gorm.io/gorm"
    "gorm.io/driver/dm"                  // GORM V2 方言
)

db, err := gorm.Open(dm.Open("dm://SYSDBA:***@localhost:5236"), &gorm.Config{})
```

**GORM V2 完整示例：**

```go
package main

import (
    "gorm.io/gorm"
    "gorm.io/driver/dm"
)

type ProductCategory struct {
    ID   int    `gorm:"primaryKey"`
    Name string `gorm:"column:NAME"`
}

func main() {
    db, err := gorm.Open(dm.Open("dm://SYSDBA:***@localhost:5236"), &gorm.Config{})
    if err != nil {
        panic(err)
    }

    // 自动迁移
    db.AutoMigrate(&ProductCategory{})

    // CRUD
    db.Create(&ProductCategory{Name: "物理"})

    var categories []ProductCategory
    db.Find(&categories)
}
```

### 方式三：原生驱动

**依赖：**

```
golang.org/x/text
github.com/golang/snappy
```

**代码示例：**

```go
package main

import (
    "database/sql"
    "fmt"
    _ "dameng/db"        // 达梦原生驱动
)

func main() {
    // 连接数据库
    db, err := sql.Open("dm", "dm://SYSDBA:***@localhost:5236")
    if err != nil {
        panic(err)
    }
    defer db.Close()

    // ---------- 基本查询 ----------
    rows, err := db.Query("SELECT NAME FROM PRODUCTION.PRODUCT_CATEGORY")
    if err != nil {
        panic(err)
    }
    defer rows.Close()

    for rows.Next() {
        var name string
        rows.Scan(&name)
        fmt.Println(name)
    }

    // ---------- 参数绑定（使用 :1, :2 占位符） ----------
    _, err = db.Exec(
        "INSERT INTO PRODUCTION.PRODUCT_CATEGORY(NAME) VALUES(:1)",
        "物理",
    )
    if err != nil {
        panic(err)
    }

    // ---------- 大字段操作 ----------
    // CLOB 使用 dm.DmClob，BLOB 使用 dm.DmBlob
    // 具体类型参见 dm 包文档
}
```

**三种方式对比：**

| 方式 | 优点 | 缺点 |
|------|------|------|
| ODBC | 通用标准接口 | 需配置 ODBC 驱动，性能略低 |
| GORM 方言 | ORM 便利，代码简洁 | 需额外方言包 |
| 原生驱动 | 性能最优，纯 Go 实现 | 直接写 SQL |

---

## .NET

### .Net Data Provider

在项目中引用达梦 .NET 驱动 `DmProvider.dll`：

```csharp
using System;
using System.Data;
using Dm;

class Program
{
    static void Main()
    {
        string connStr = "Server=localhost;Port=5236;User Id=SYSDBA;PWD=*****;Database=DM8";

        using (DmConnection conn = new DmConnection(connStr))
        {
            conn.Open();

            // 查询
            using (DmCommand cmd = new DmCommand("SELECT NAME FROM PRODUCTION.PRODUCT_CATEGORY", conn))
            {
                using (DmDataReader reader = cmd.ExecuteReader())
                {
                    while (reader.Read())
                    {
                        Console.WriteLine(reader["NAME"]);
                    }
                }
            }

            // 参数化插入
            using (DmCommand cmd = new DmCommand(
                "INSERT INTO PRODUCTION.PRODUCT_CATEGORY(NAME) VALUES(:name)", conn))
            {
                cmd.Parameters.Add(new DmParameter(":name", "物理"));
                cmd.ExecuteNonQuery();
            }
        }
    }
}
```

### Dapper 框架

```csharp
using Dapper;
using Dm;
using System.Data;

// 建立连接
using (IDbConnection conn = new DmConnection(
    "Server=localhost;Port=5236;User Id=SYSDBA;PWD=*****;Database=DM8"))
{
    // 查询
    var categories = conn.Query<ProductCategory>("SELECT * FROM PRODUCTION.PRODUCT_CATEGORY");

    // 参数化
    var result = conn.Execute(
        "INSERT INTO PRODUCTION.PRODUCT_CATEGORY(NAME) VALUES(:name)",
        new { name = "物理" }
    );

    // 存储过程
    var data = conn.Query("CALL GET_CATEGORIES(:id)", new { id = 1 });
}
```

---

## 附录：各语言驱动对照速查

| 语言 | 驱动/接口 | 占位符 | 安装方式 |
|------|----------|--------|----------|
| Python | dmPython | `?` | `python setup.py install` |
| Java | JDBC | `?` | classpath 引入 JAR |
| C/C++ | ODBC | `?` | 系统 odbcinst.ini 配置 |
| PHP | dm 扩展 | `:name` | 复制 .so/.dll + php.ini |
| PHP PDO | PDO_DM | `?` | 同上 |
| Go (原生) | dameng/db | `:1,:2` | `go get` + 依赖 |
| Go (ODBC) | odbc | `?` | `go get` |
| Go (GORM) | gorm dm driver | `?` | 解压方言 zip |
| .NET | DmProvider.dll | `:name` | 项目引用 DLL |

---

## 补充：SQLark 跨平台开发管理工具

SQLark 是达梦公司在 2024 年正式推出的**跨平台数据库统一管理工具**，定位是"一个客户端管理多种数据库"，目标是降低信创项目里"多库并存"的运维复杂度。

### 支持的数据库

- 达梦 DM / DMHS
- Oracle、MySQL、PostgreSQL、SQL Server
- KingbaseES（人大金仓）
- 后续会扩展到更多国产数据库

### 核心特性

| 特性 | 说明 |
|------|------|
| **跨库统一管理** | 一个客户端连接 DM、Oracle、MySQL、PG 等，无需切换工具 |
| **测试数据生成** | 内置数据生成向导，可按表结构批量造数（性能压测必备） |
| **数据迁移向导** | 图形化 DTS 操作，适合非 DBA 角色完成异构迁移 |
| **SQL 智能提示** | 上下文感知的关键字、表名、列名补全 |
| **执行计划可视化** | 图形化展示 DM 优化器选择的执行路径 |
| **多端登录** | Windows / macOS / Linux 三大平台同步发布 |

### 适用场景

- 多数据库并存的项目（信创替代 + 老系统并存期）
- 数据库选型 PoC 阶段需要快速试用多款产品
- 性能压测前的批量测试数据准备
- 非 DBA 角色（开发、测试）日常查询

### 与传统 DM 工具的对比

| 工具 | 定位 | 适用人群 |
|------|------|----------|
| **DM Manager** | DM 官方桌面管理工具 | DBA |
| **DTS** | 达梦数据迁移工具 | 迁移工程师 |
| **DISQL** | 达梦命令行客户端 | 高级 DBA / 脚本化运维 |
| **SQLark** | 跨库统一管理 + 一站式开发 | 全员（开发/测试/DBA） |

### 下载

前往达梦官网（[eco.dameng.com](https://eco.dameng.com)）下载 SQLark 安装包，安装后注册达梦社区账号即可使用。

---

## 补充：Django + 达梦

达梦官方为 Django 框架提供了**专门的 dmDjango 驱动**，该驱动在 dmPython 之上做了 Django ORM 适配（DB-API 2.0 接口 + Django backend 协议），可以像使用 MySQL/PostgreSQL 一样使用达梦。

### 安装

```bash
# 1. 先确保 dmPython 已安装（参考本文 Python 章节）
export DM_HOME=/dm8
export LD_LIBRARY_PATH=$DM_HOME/bin:$LD_LIBRARY_PATH
python3 setup.py install  # 在 $DM_HOME/drivers/python/dmPython/ 下

# 2. 安装 dmDjango（实际包名以官方为准，社区常见名为 django-dm）
pip install django-dm
```

### settings.py 配置

```python
DATABASES = {
    'default': {
        'ENGINE': 'django_dm.backends.dm',   # 达梦后端
        'NAME': 'BOOKSHOP',                  # 数据库名
        'USER': 'SYSDBA',
        'PASSWORD': '<StrongPassword>',
        'HOST': '127.0.0.1',
        'PORT': '5236',
        'OPTIONS': {'charset': 'UTF-8'},     # 字符集
    }
}
```

### 关键注意点

- **大小写敏感**：`CASE_SENSITIVE` 参数决定对象名大小写是否敏感，迁移到生产前需要与 DBA 确认
- **自增主键**：达梦支持 `IDENTITY` 列 + 序列两种方式，Django 默认使用 `IDENTITY`
- **分页 SQL**：Django ORM 生成的 `LIMIT/OFFSET` 在达梦上工作正常，DM 也支持 `ROWNUM` 写法
- **字符集**：库字符集需与 `OPTIONS.charset` 一致，否则中文会出现乱码

### 常用命令

```bash
# 迁移
python manage.py makemigrations
python manage.py migrate

# 创建超级用户
python manage.py createsuperuser
```

---

## 补充：Flask + 达梦

Flask 框架没有内置 ORM，搭配达梦最简洁的方式是**直接使用 dmPython**，或者通过 **SQLAlchemy + dmPython 适配**。

### 方式一：直接使用 dmPython（轻量级）

```python
from flask import Flask, jsonify
import dmPython

app = Flask(__name__)

def get_conn():
    return dmPython.connect(
        user='SYSDBA',
        password='*****',
        server='127.0.0.1',
        port=5236
    )

@app.route('/users')
def users():
    conn = get_conn()
    try:
        cursor = conn.cursor()
        cursor.execute('SELECT id, name FROM sysuser')
        rows = cursor.fetchall()
        # 序列化为 JSON
        return jsonify([{'id': r[0], 'name': r[1]} for r in rows])
    finally:
        conn.close()

if __name__ == '__main__':
    app.run(debug=True)
```

### 方式二：SQLAlchemy 集成（推荐生产用）

```bash
pip install sqlalchemy flask-sqlalchemy dmPython
```

```python
from flask import Flask
from flask_sqlalchemy import SQLAlchemy
import dmPython

app = Flask(__name__)
app.config['SQLALCHEMY_DATABASE_URI'] = 'dm+dmPython://SYSDBA:<StrongPassword>@127.0.0.1:5236/BOOKSHOP'
app.config['SQLALCHEMY_TRACK_MODIFICATIONS'] = False

db = SQLAlchemy(app)

class User(db.Model):
    __tablename__ = 'sysuser'
    id = db.Column(db.Integer, primary_key=True)
    name = db.Column(db.String(50))

@app.route('/users')
def users():
    return {'users': [{'id': u.id, 'name': u.name} for u in User.query.all()]}
```

### 关键注意点

- **连接池**：生产环境务必使用连接池（推荐 `sqlalchemy.pool.QueuePool`），避免每次请求都新建连接
- **大对象 BLOB/CLOB**：dmPython 通过 `dmPython.CLOB` / `dmPython.BLOB` 类型处理，需要在 cursor 显式声明
- **占位符**：dmPython 使用 `?` 作为占位符，SQLAlchemy ORM 会自动处理
- **服务名 vs SID**：达梦默认使用全局数据库名（类似 Oracle Service Name），单实例可省略
- **超时与断开**：长连接建议开启 `pool_pre_ping=True` 防止数据库端超时断开

---

## 选型建议

- **简单脚本 / 内部工具** → dmPython 直接调用
- **Web 后台 / API 服务** → Flask + SQLAlchemy + dmPython
- **CMS / 表单驱动业务** → Django + django-dm
- **企业级管理后台** → Django + Admin + django-dm（最快出活）
- **多库统一开发** → SQLark 作为日常查询 + 建模工具

## 版本与踩坑提示

| 场景 | 坑点 | 解决方式 |
|------|------|----------|
| Django 4.x + DM8 | 4.x 移除了部分 2.0 老 API | 使用 3.2 LTS 版本 |
| Flask + SQLAlchemy 2.x | 新版 style 用 `db.session.execute(select(...))` | ORM 模型写法不变，raw SQL 改用 `text()` 包装 |
| SQLark 登录 | 默认端口 5236，连不上 | 确认 `dm.ini` 里的 `PORT_NUM` 参数 |
| Django migration | 自增 ID 冲突 | 用 `BigAutoField` 主键 |
| Flask 部署 | gunicorn 多 worker 重复建连 | 启用 `gunicorn --preload` 共享连接池 |
