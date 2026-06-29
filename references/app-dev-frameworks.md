# 其他开发框架适配

> 本章覆盖 `app-dev.md` 之外、达梦官方文档中支持的其他 ORM / 分库分表 / 跨语言驱动。所有驱动版本必须与 DM 服务器版本保持一致。

## 一、框架总览

| 语言 | 框架 | 达梦驱动 / 方言 | 关键依赖 | 文档 |
|------|------|-----------------|----------|------|
| Java | Hibernate 5.x/6.x | `org.hibernate.dialect.DmDialect` | `DmJdbcDriver8.jar` + `DmDialect-for-hibernate5.6.jar` | [java-hibernate-frame](https://eco.dameng.com/document/dm/zh-cn/app-dev/java-hibernate-frame.html) |
| Java | iBatis / MyBatis 旧版 | `dm.jdbc.driver.DmDriver`（数据源配置）| iBatis 2.x lib | [java-iBatis-frame](https://eco.dameng.com/document/dm/zh-cn/app-dev/java-iBatis-frame.html) |
| Java | MyCat 1.6 | JDBC 直连，URL 末尾加 `?comOra=true` | `DmJdbcDriver` + `mysql-connector-j` | [java-mycat_new](https://eco.dameng.com/document/dm/zh-cn/app-dev/java-mycat_new.html) |
| Java | Sharding-JDBC 5.x | `dm.jdbc.driver.DmDriver` | `sharding-jdbc-spring-boot-starter` + Druid | [JAVA_ShardingJDBC](https://eco.dameng.com/document/dm/zh-cn/app-dev/JAVA_ShardingJDBC.html) |
| Java | Querydsl 5.0 | `DmDialect`（Hibernate 5.6）+ APT 生成 Q 类 | `querydsl-jpa` + `apt-maven-plugin` | [Querydsl](https://eco.dameng.com/document/dm/zh-cn/app-dev/Querydsl.html) |
| Python | Django 3.1.x | `django_dmPython`（Backend Engine）| `Django_dmPython`（按 Django 版本匹配）+ `dmPython` | [python-django](https://eco.dameng.com/document/dm/zh-cn/app-dev/python-django.html) |
| Python | Flask + SQLAlchemy 2.0 | `dm+dmPython://` | `flask-sqlalchemy` + `sqlalchemy-dm` 2.0.0 | [python-flask](https://eco.dameng.com/document/dm/zh-cn/app-dev/python-flask.html) |
| Python | SQLAlchemy 1.3/1.4/2.0 | `sqlalchemy_dm` | `SQLAlchemy`（版本必须与 `sqlalchemy_dm` 对齐）+ `dmPython` | [python-SQLAlchemy](https://eco.dameng.com/document/dm/zh-cn/app-dev/python-SQLAlchemy.html) |
| .NET | Dapper | `Dm.Data.DmClient`（net45/net8）| `Dapper` + `DmProvider.dll` | [net-Dapper](https://eco.dameng.com/document/dm/zh-cn/app-dev/net-Dapper.html) |
| .NET | EF Core | ⚠️ 待官方手册确认 | — | — |
| Go | GORM V1 | `dmgorm1`（`gorm.Open("dm", ...)`）| `dmgorm1.zip` + GORM V1 | [go_gorm](https://eco.dameng.com/document/dm/zh-cn/app-dev/go_gorm.html) |
| Go | GORM V2 | `dmgorm2`（`dm.Open(...)`）| `dmgorm2.zip` + `gorm.io/gorm` | [go_gorm](https://eco.dameng.com/document/dm/zh-cn/app-dev/go_gorm.html) |
| Go | ODBC | `DM8 ODBC DRIVER` + `alexbrainman/odbc` | `unixODBC`（Linux）+ `libdodbc.so` | [go_odbc](https://eco.dameng.com/document/dm/zh-cn/app-dev/go_odbc.html) |
| Go | 原生 `go_dm` | ⚠️ 待官方手册确认（仅在 DM 安装目录 `drivers/go` 下提供）| — | [go_dm](https://eco.dameng.com/document/dm/zh-cn/app-dev/go_dm.html) |
| Node.js | dmdb | `dmdb`（npm 包）| Node.js ≥ 12.0.0 | [JavaScript_NodeJs](https://eco.dameng.com/document/dm/zh-cn/app-dev/JavaScript_NodeJs.html) |
| PHP | PDO_DMDB / 原生 `dm_` | `libphp74_dm.so` + `php74_pdo_dm.so` | PHP 7.2 / 7.3 / 7.4 / 8.x | [php_php_new](https://eco.dameng.com/document/dm/zh-cn/app-dev/php_php_new.html) |

## 二、Java 其他框架

### 2.1 Hibernate 5.x / 6.x

核心三件套：**Hibernate 方言 + DM JDBC 驱动 + 大字段配置**。`hibernate.cfg.xml` 关键属性如下（5.6 系列；6.x 方言类路径不变）：

```xml
<property name="hibernate.dialect">org.hibernate.dialect.DmDialect</property>
<property name="hibernate.connection.driver_class">dm.jdbc.driver.DmDriver</property>
<property name="hibernate.connection.url">jdbc:dm://localhost:5236</property>
<property name="hibernate.connection.SetBigStringTryClob">true</property>
```

序列绑定：`@GeneratedValue(strategy = GenerationType.SEQUENCE, generator = "seq")` + `@SequenceGenerator(name="seq", sequenceName="MY_SEQ")`；IDENTITY 自增列也支持。`@Generated(GenerationTime.INSERT)` 触发器生成列须使用 `@DynamicInsert`。

### 2.2 iBatis / MyBatis 旧版

iBatis 2.x 仅作为 SQL 映射框架，**不参与方言**，只换驱动：

```properties
# jdbc.properties
jdbc.driver=dm.jdbc.driver.DmDriver
jdbc.url=jdbc:dm://localhost:5236
jdbc.username=SYSDBA
jdbc.password=*****
```

XML 映射中**占位符使用 `#字段#`**（与 MyBatis 3.x 的 `#{field}` 不同），参数绑定用 `parameterClass`，结果映射用 `resultClass` / `resultMap`。

### 2.3 MyCat 1.6 分库分表

MyCat 端通过 `dbType="oracle"` + `dbDriver="jdbc"` 复用 Oracle 通道，URL 末尾加 `?comOra=true` 开启 DM 的 Oracle 兼容模式：

```xml
<dataHost name="localhost1" dbType="oracle" dbDriver="jdbc" switchType="1">
    <heartbeat>select user()</heartbeat>
    <writeHost host="hostM1"
               url="jdbc:dm://localhost:5236?comOra=true"
               user="APP_USER" password="*****"/>
</dataHost>
```

应用端用 **MySQL 驱动**（`com.mysql.cj.jdbc.Driver`）连 MyCat 8066 端口。需在 `schema.xml` 配 `checkSQLschema="true"` 自动剥离模式名。

### 2.4 Sharding-JDBC 5.x

`application.properties` 关键配置（按 `gid` 水平分表到 2 个达梦实例）：

```properties
spring.shardingsphere.datasource.names=g1,g2
spring.shardingsphere.datasource.g1.type=com.alibaba.druid.pool.DruidDataSource
spring.shardingsphere.datasource.g1.driver-class-name=dm.jdbc.driver.DmDriver
spring.shardingsphere.datasource.g1.url=jdbc:dm://localhost:5236
spring.shardingsphere.datasource.g1.username=SYSDBA
spring.shardingsphere.datasource.g1.password=*****
spring.shardingsphere.sharding.tables.goods.actual-data-nodes=g$->{1..2}.goods_$->{1..2}
spring.shardingsphere.sharding.tables.goods.key-generator.column=gid
spring.shardingsphere.sharding.tables.goods.key-generator.type=SNOWFLAKE
spring.shardingsphere.sharding.tables.goods.table-strategy.inline.sharding-column=gid
spring.shardingsphere.sharding.tables.goods.table-strategy.inline.algorithm-expression=goods_$->{gid % 2 + 1}
```

库内分表单实例即可，跨库分表用多 dataSource。`table-strategy.inline` 支持 Groovy 表达式，达梦做分片库时建议显式指定 `database-strategy.inline.sharding-column`。

### 2.5 Querydsl（类型安全 SQL）

Querydsl 5.0 + Hibernate 5.6，需 `apt-maven-plugin` 在编译期生成 Q 类；方言复用 `DmDialect`：

```xml
<!-- 关键依赖 -->
<dependency>
    <groupId>com.querydsl</groupId><artifactId>querydsl-jpa</artifactId><version>5.0.0</version>
</dependency>
<dependency>
    <groupId>com.querydsl</groupId><artifactId>querydsl-apt</artifactId>
    <version>5.0.0</version><classifier>jpa</classifier>
</dependency>
```

`persistence.xml` 配 `hibernate.dialect=org.hibernate.dialect.DmDialect`；`hbm2ddl.auto=update` 会按实体 `@Entity` 自动建表。查询写法：

```java
JPAQueryFactory qf = new JPAQueryFactory(em);
QUser u = QUser.user;
List<User> list = qf.selectFrom(u)
    .where(u.name.eq("Bob"))
    .orderBy(u.id.asc())
    .fetch();
```

## 三、Python 生态

### 3.1 Django + django_dmPython

`settings.py` 用 `django_dmPython` 作为 ENGINE（**不是 `django.db.backends.mysql`**）；版本号必须与 Django 主版本严格对齐（3.1.7 对应 `django317/django_dmPython`）：

```python
DATABASES = {
    'default': {
        'ENGINE': 'django_dmPython',         # 达梦专有 backend
        'NAME': 'DAMENG',                    # DM 实例名
        'USER': 'SYSDBA', 'PASSWORD': '*****',
        'HOST': '127.0.0.1', 'PORT': '5236',
        'OPTIONS': {'local_code': 1, 'connection_timeout': 5},
    }
}
TIME_ZONE = 'Asia/Shanghai'
USE_TZ = False
```

迁移命令 `python manage.py makemigrations dm` + `migrate` 正常生效。ORM 字段对应：`AutoField` → `IDENTITY`；`DecimalField(max_digits=10, decimal_places=4)` → `DEC(10,4)`；大字段 `TextField` 映射 `TEXT`。

### 3.2 Flask + Flask-SQLAlchemy

URI 前缀固定 `dm+dmPython://`，底层走 `sqlalchemy_dm` 方言包：

```python
app.config['SQLALCHEMY_DATABASE_URI'] = "dm+dmPython://SYSDBA:*****@127.0.0.1:5236"
app.config['SQLALCHEMY_ECHO'] = True
db = SQLAlchemy(app)

class Product(db.Model):
    __tablename__ = 'FLASK'
    PRODUCTID = db.Column(db.Integer, primary_key=True, autoincrement=True)
    NAME = db.Column(db.String(100))
    NOWPRICE = db.Column(db.Numeric(19, 4))
```

`sqlalchemy-dm` 与 `SQLAlchemy` 主版本强绑定（1.1.10↔1.3.x；1.4.39↔1.4.x；2.0.0↔2.0.x），混装会触发 `AttributeError: module 'sqlalchemy.util' has no attribute 'py2k'`。

### 3.3 SQLAlchemy 原生用法

不依赖 Flask，纯 SQLAlchemy Core / ORM：

```python
from sqlalchemy import create_engine
from sqlalchemy.orm import sessionmaker
engine = create_engine('dm+dmPython://SYSDBA:*****@127.0.0.1:5236')
Session = sessionmaker(bind=engine)
session = Session()
session.add(Product(NAME='水浒传', AUTHOR='施耐庵'))
session.commit()
```

大字段 `Text` → `TEXT`；`Numeric(19,4)` → `DEC(19,4)`；DM 的 `IDENTITY(1,1)` 对应 `Column(Integer, autoincrement=True, primary_key=True)`。

## 四、.NET 生态

### 4.1 Dapper

`Dapper` 通过扩展 `IDbConnection` 工作，达梦的 `DmConnection` 已实现 `IDbConnection`，所以**直接 new `DmConnection`**即可。NuGet 装 Dapper + 引用 `DmProvider.dll`（按 .NET Framework 版本选 net45 / net8 子目录）：

```csharp
using Dm;
using Dapper;

public class DmConnectionFactory {
    static string sqlConnString = "Server=127.0.0.1; UserId=SYSDBA; PWD=*****;";
    public static IDbConnection GetConn() => new DmConnection(sqlConnString);
}

// 调用：绑定参数用 :Name 形式
string sql = "select Id,Name,City from Person where Id=:Id";
var p = DmConnectionFactory.GetConn().QueryFirstOrDefault<Person>(sql, new { Id = 1 });
```

**注意参数占位符是 `:Name`（冒号前缀）**，与 SQL Server `@Name`、MySQL `?` 不同。

### 4.2 Entity Framework Core

EF Core 对达梦的支持**官方未明确**，社区方案是手写 `EntityFrameworkCore.Dm`（GitHub 第三方），DBA 视图与迁移可能与 DM 方言有差异。**生产项目建议走 Dapper + 手写 SQL** 或 Hibernate 替代。

## 五、Go 生态

### 5.1 GORM V1（`gorm.Open("dm", ...)`）

驱动包 `dmgorm1`（来自 `gorm_v1_dialect.zip`），需置于 `GOPATH/src` 并 `import _ "dmgorm1"`。**使用 V1 时数据库 INI 参数必须 `CASE_SENSITIVE=N`**，否则大小写策略冲突：

```go
import (
    _ "dmgorm1"
    "github.com/jinzhu/gorm"
)
type Product struct { gorm.Model; Code string; Price uint }

db, _ := gorm.Open("dm", "dm://SYSDBA:*****@localhost:5236")
db.AutoMigrate(&Product{})
db.Create(&Product{Code: "L1212", Price: 1000})
db.First(&product, "\"code\" = ?", "L1212") // 字段名需带双引号
```

### 5.2 GORM V2（`gorm.Open(dm.Open(...))`）

`dmgorm2` 与 GORM V2 互相不兼容，必须配套使用：

```go
import (
    "dmgorm2"
    "gorm.io/gorm"
    "gorm.io/gorm/logger"
)
db, _ := gorm.Open(dm.Open("dm://SYSDBA:*****@localhost:5236"),
    &gorm.Config{Logger: logger.Default.LogMode(logger.Info)})
db.AutoMigrate(&Product{})
db.Create(&Product{Code: "D42", Price: 100})
db.First(&product, "\"code\" = ?", "D42")
```

V2 已无大小写约束限制，标识符默认带双引号。

### 5.3 Go ODBC（`alexbrainman/odbc` + `unixODBC`）

Linux 需先编译 `unixODBC`（`./configure --enable-gui=no && make && make install`），再配置 `/etc/odbcinst.ini` + `/etc/odbc.ini`：

```ini
# /etc/odbcinst.ini
[DM8 ODBC DRIVER]
Driver = /home/dmdba/dmdbms/bin/libdodbc.so

# /etc/odbc.ini
[dm8]
Driver = DM8 ODBC DRIVER
SERVER = localhost; UID = SYSDBA; PWD = *****; TCP_PORT = 5236
```

Go 代码（搭配 `xorm` 或 `database/sql`）：

```go
import (
    _ "github.com/alexbrainman/odbc"
    "github.com/go-xorm/xorm"
)
engine, _ := xorm.NewEngine("odbc",
    "driver={DM8 ODBC DRIVER};server=127.0.0.1:5236;database=DM8;uid=SYSDBA;pwd=*****;charset=utf8")
engine.Ping()
```

### 5.4 Go 原生驱动

`go_dm` 在 DM 安装目录 `drivers/go` 下提供源码包，使用方式类似 `dmgorm` 但**不带 GORM 封装**，直接做 `database/sql` driver。**官方单独文档较少，建议优先用 GORM V2 + `dmgorm2`**。

## 六、Node.js（dmdb 驱动）

达梦官方发布到 npm 的包名为 `dmdb`，**Node.js 版本必须 ≥ 12.0.0**。安装后通过 `dm.createPool` / `conn.execute` 操作：

```js
const db = require('dmdb');
const fs = require('fs');

async function example() {
    const pool = await db.createPool({
        connectString: "dm://SYSDBA:*****@localhost:5236?autoCommit=false",
        poolMax: 10, poolMin: 1
    });
    const conn = await pool.getConnection();
    // 占位符用 :1, :2, ...（与 JDBC 类似）
    await conn.execute(
        "INSERT INTO production.product(name,author,photo) VALUES(:1,:2,:3)",
        [{val:"三国演义"}, {val:"罗贯中"},
         {val: fs.createReadStream("/data/sgy.jpg")}]
    );
    await conn.execute("commit;");
}
example();
```

LOB 用流式对象（`fs.createReadStream`），查询后通过 `lob.on('data')` 读出。无 Sequelize 官方适配，需 ORM 自行封装。

## 七、PHP（PDO_DMDB + 原生 dm_ 函数）

PHP 必须选**与版本严格匹配**的 `.so` 驱动（`libphp74ts_dm.so` + `php74ts_pdo_dm.so`，TS/NTS 也分两种）。`php.ini` 增加：

```ini
[PHP_DM]
extension_dir = "/opt/dmdbms/drivers/php_pdo"
extension = libphp74_dm.so
extension = php74_pdo_dm.so

[dm]
dm.port = 5236
dm.default_host = localhost
dm.default_user = SYSDBA
dm.default_pw = *****
```

两种调用风格——**PDO 风格**：

```php
$pdo = new PDO("dm:host=localhost;port=5236", "SYSDBA", "*****");
$stmt = $pdo->prepare("INSERT INTO PRODUCTION.PRODUCT_CATEGORY(name) VALUES(?)");
$stmt->execute(['物理']);
```

**原生 `dm_` 风格**（不用 PDO）：

```php
$link = dm_connect("localhost:5236", "SYSDBA", "*****");
$result = dm_exec($link, "insert into PRODUCTION.PRODUCT_CATEGORY(NAME) values('物理')");
dm_close($link);
```

Windows 还需把 `dmdpi.dll / dmcalc.dll / dmelog.dll / dmos.dll` 等 16 个 dll 拷到 `C:\Windows\SysWOW64` 和 `System32`（不替换已有文件）。

## 八、参考

1. [Hibernate 框架](https://eco.dameng.com/document/dm/zh-cn/app-dev/java-hibernate-frame.html)
2. [iBatis 框架](https://eco.dameng.com/document/dm/zh-cn/app-dev/java-iBatis-frame.html)
3. [MyCat 框架](https://eco.dameng.com/document/dm/zh-cn/app-dev/java-mycat_new.html)
4. [Sharding-JDBC 框架](https://eco.dameng.com/document/dm/zh-cn/app-dev/JAVA_ShardingJDBC.html)
5. [Querydsl 连接达梦](https://eco.dameng.com/document/dm/zh-cn/app-dev/Querydsl.html)
6. [Django 框架](https://eco.dameng.com/document/dm/zh-cn/app-dev/python-django.html)
7. [Flask 框架](https://eco.dameng.com/document/dm/zh-cn/app-dev/python-flask.html)
8. [SQLAlchemy 框架](https://eco.dameng.com/document/dm/zh-cn/app-dev/python-SQLAlchemy.html)
9. [Dapper 框架](https://eco.dameng.com/document/dm/zh-cn/app-dev/net-Dapper.html)
10. [GORM 框架](https://eco.dameng.com/document/dm/zh-cn/app-dev/go_gorm.html)
11. [Go ODBC 接口](https://eco.dameng.com/document/dm/zh-cn/app-dev/go_odbc.html)
12. [Node.js 框架](https://eco.dameng.com/document/dm/zh-cn/app-dev/JavaScript_NodeJs.html)
13. [PHP 数据库接口](https://eco.dameng.com/document/dm/zh-cn/app-dev/php_php_new.html)

## 九、方言与字段类型对照（跨语言速查）

| DM 原生类型 | Hibernate | Django | SQLAlchemy | EFCore（社区） | GORM V2 |
|-------------|-----------|--------|------------|-----------------|---------|
| `INT IDENTITY` | `Integer` + `@GeneratedValue(IDENTITY)` | `AutoField` / `IntegerField` | `Integer, autoincrement=True` | `int` + `ValueGeneratedOnAdd` | `int` + `autoIncrement` |
| `BIGINT` | `Long` | `BigIntegerField` | `BigInteger` | `long` | `int64` |
| `VARCHAR(n)` | `String(n)` | `CharField(max_length=n)` | `String(n)` | `[MaxLength(n)] string` | `string` + `size:n` |
| `TEXT` / `CLOB` | `String` / `@Lob` | `TextField` | `Text` | `string` (unbounded) | `string` |
| `BLOB` / `IMAGE` | `byte[]` + `@Lob` | `BinaryField` | `LargeBinary` | `byte[]` | `[]byte` |
| `DEC(p,s)` | `BigDecimal` | `DecimalField(max_digits=p, decimal_places=s)` | `Numeric(p, s)` | `decimal(p,s)` | `float64` / `decimal` |
| `DATE` | `Date` | `DateField` | `Date` | `DateTime`（只取日期） | `time.Time`（truncate） |
| `DATETIME` / `TIMESTAMP` | `Timestamp` | `DateTimeField` | `DateTime` | `DateTime` | `time.Time` |
| `ROWID` 伪列 | 无标准映射，建议业务主键 | 同左 | 同左 | 同左 | 同左 |

> 关键点：DM 的 `IDENTITY(1,1)` 自增列在 Hibernate 中是 `GenerationType.IDENTITY`、在 Django 是 `AutoField`、在 SQLAlchemy 是 `autoincrement=True`、在 EFCore 是 `ValueGeneratedOnAdd`、在 GORM 是 `autoIncrementTag`。**不要混用 SEQUENCE + IDENTITY**。

## 十、JDBC URL 写法速查（所有语言通用）

| 场景 | URL / 连接串 | 备注 |
|------|--------------|------|
| 默认（IP+端口） | `jdbc:dm://localhost:5236` | 默认实例 `DAMENG` |
| 指定实例名 | `jdbc:dm://localhost:5236/DAMENG` | 服务名（INSTANCE_NAME）|
| Oracle 兼容模式 | `jdbc:dm://localhost:5236?comOra=true` | MyCat 分片必备 |
| MySQL 兼容模式 | `jdbc:dm://localhost:5236?comMysql=true` | 让 MyBatis-Plus 用 MySQL 方言 |
| SSL 加密 | `jdbc:dm://localhost:5236?ssl=true&sslTrustStore=...` | 国产化部署要求 |
| 服务名连接 | `jdbc:dm://localhost?svcName=dm_service` | 需 `/etc/dm_svc.conf` 配置 |
| Python / Go / PHP URI | `dm+dmPython://user:pwd@host:5236` / `dm://user:pwd@host:port` | 各种 SDK 略不同 |

驱动类统一为 `dm.jdbc.driver.DmDriver`；Python 走 `dmPython` / `sqlalchemy_dm`；Go 用 `dmgorm1/2` 或 `libdodbc`；Node.js 是 `dmdb`；PHP 是 `libphpXX_dm.so` + `phpXX_pdo_dm.so`。

## 十一、常见踩坑（按出现频次）

1. **驱动版本不匹配**：报错 `NoClassDefFoundError` 或 `cannot find symbol`。**所有驱动版本必须与 DM 服务器版本保持一致**（DM8.1 配 DM8.1 的 jdbc、dmgorm、dmdb、PHP so 套件）。
2. **大小写策略冲突**：GORM V1 + DM 默认 `CASE_SENSITIVE=Y` 会导致 `"PRODUCT_CATEGORY"` 与 `product_category` 找不到。**方案：DB 配 `CASE_SENSITIVE=N`，或 V1 全程手动双引号。**V2 已修复。
3. **Dapper 占位符是 `:Name` 不是 `@Name`**：从 SQL Server 迁移过来的最常踩。批量插入也用 `:Name` + `DynamicParameters.Add`。
4. **PHP 没把 `libphp74_dm.so` + `php74_pdo_dm.so` 同时加载**：缺一会报 `Unable to start DM module`。Windows 还要把 16 个 bin 下 dll 拷到 `SysWOW64` 和 `System32`。
5. **sqlalchemy_dm 与 SQLAlchemy 主版本错配**：报错 `AttributeError: module 'sqlalchemy.util' has no attribute 'py2k'`。三者对应关系：`1.1.10 ↔ 1.3.x`、`1.4.39 ↔ 1.4.x`、`2.0.0 ↔ 2.0.x`。
6. **Django `django_dmPython` 源码包与 Django 主版本错配**：必须下与 Django 主版本号一致的子目录（3.1.x 用 `django317/django_dmPython`、2.2.x 用 `django22/django_dmPython`）。
7. **Sharding-JDBC + 达梦分片时主键策略**：达梦自增列不能跨实例全局唯一，建议用 `SNOWFLAKE` 算法（`key-generator.type=SNOWFLAKE`）。
8. **MyCat + 达梦时模式名错乱**：在 `schema.xml` 把 `checkSQLschema="true"`，SQL 不要带模式名前缀。
9. **Querydsl 编译期没生成 Q 类**：检查 `apt-maven-plugin` 的 `outputDirectory` + `processor=com.querydsl.apt.jpa.JPAAnnotationProcessor`，且 `mvn compile` 前 clean 一次。
10. **Node.js `dmdb` 装不上**：Node.js 版本 < 12.0.0 会失败；离线环境用 `node_modules_linux.zip` 解压到 node 安装目录。

## 十二、性能与稳定性建议

- **连接池大小**：Java 侧 Druid `initialSize=5, minIdle=10, maxActive=50`；Python 侧 `pool_size=10, max_overflow=20`；Go 侧 `SetMaxOpenConns(5)` + `SetMaxIdleConns(5)`（DM 单机不要超过 CPU 核数 × 4）。
- **大事务**：DM 默认锁等待 60s，跨多个 DM 实例的事务考虑关掉强一致性改 `READ UNCOMMITTED`（业务允许前提下）。
- **大字段（BLOB/CLOB）**：JDBC 加 `SetBigStringTryClob=true`（Hibernate）、Python 用 `LargeBinary` 单独插入（不要走 `INSERT ... VALUES` 一次大对象）。
- **分页**：DM `LIMIT n OFFSET m` 与 MySQL 一致；老版本只支持 `ROWNUM <= n` 写法。
- **连接复用**：所有语言都建议启用连接池；Node.js 的 `dm.createPool` 是必须。
- **方言与 SQL 兼容**：业务里能不用触发器就不用、能不用 IDENTITY 就用 SEQUENCE（迁移到 Oracle/MySQL 更顺）。

## 十三、跨语言迁移清单

如果项目从其它数据库迁到 DM8，下表是对各框架的最小改动：

| 框架 | 改 Driver | 改 Dialect | 改 URL | 改 SQL |
|------|-----------|------------|--------|--------|
| MyBatis / MyBatis-Plus | ✅ 仅 DmDriver | 无 | `jdbc:dm://...` | 分页 / `INSERT ... ON DUPLICATE` |
| Spring Data JPA | ✅ DmDriver | `DmDialect` | 同上 | 同上 |
| Hibernate | ✅ DmDriver | `DmDialect` | 同上 | `LIMIT` / 序列 |
| Django | ✅ `django_dmPython` | 字段类型核对 | URI | 少量 manager API |
| Flask-SQLAlchemy | ✅ `sqlalchemy-dm` | URI 前缀改 `dm+dmPython` | 同上 | 极少量 |
| Dapper | ✅ `DmProvider` | 占位符 `:Name` | `Server=...;` | `?` 改 `:N` |
| GORM V1/V2 | ✅ `dmgorm1/2` | V1 配 `CASE_SENSITIVE=N` | URI | 字段名加双引号 |
| xorm | ✅ odbc/原生 | 无 | driver=... | 无 |
| dmdb (Node.js) | ✅ `npm i dmdb` | 占位符 `:1, :2` | connectString | 无 |
| PDO (PHP) | ✅ `pdo_dm` | 无 | `dm:host=...` | `?` 即可 |
| 原生 `dm_` PHP | ✅ `libphpXX_dm.so` | 无 | `dm_connect` | 无 |

> 凡是原来用 MySQL 方言特性（如 `INSERT IGNORE`、`GROUP_CONCAT`）的，需要替换为 DM 等价物（`MERGE INTO`、`WM_CONCAT`）。`AUTO_INCREMENT` → `IDENTITY(1,1)`，`ENGINE=InnoDB` 段在迁移脚本中删除。

## 十四、按业务场景的选型建议

- **企业后台管理系统（CRUD 为主，少复杂查询）**：Spring Boot + **MyBatis-Plus** + `DmDriver` 是最少踩坑的方案；如历史是 Hibernate 5 栈，可保持 Hibernate 不动。
- **金融核心 / 复杂事务**：Hibernate 5.6 + **DmDialect** + Druid 连接池。Hibernate 的 HQL 易于审计和分库分表扩展。
- **分库分表 / 读写分离**：Sharding-JDBC 5.x（应用内嵌）或 MyCat 1.6（独立代理）。两者都可与达梦组成完整分布式方案。
- **数据分析 / 报表**：保持 SQL 手写，**直接用 JDBC + Druid**；ORM 反而会限制复杂 SQL。
- **数据中台 / ETL**：Python + **SQLAlchemy 1.4/2.0** + `dm+dmPython`，搭配 pandas 做数据加工。
- **快速原型 / Web 小应用**：Flask + Flask-SQLAlchemy + `dm+dmPython`；10 行代码就跑通。
- **Django 全栈**：`django_dmPython` backend；Admin 后台、ORM 迁移、表单校验全部开箱即用。
- **.NET 业务系统**：Dapper + 手写 SQL；不要硬上 EF Core（DM 方言差异大）。
- **Go 微服务**：GORM V2 + `dmgorm2`；V1 仅遗留系统使用。
- **Node.js 实时服务 / BFF**：`dmdb` 驱动 + `dm.createPool`；不适合高并发写入场景。
- **PHP 传统网站**：Nginx + PHP-FPM + **PDO_DMDB**；传统 `.php` 老系统用原生 `dm_` 函数。

## 十五、国产化迁移 Checklist

把 MySQL/Oracle/SQL Server 项目迁到 DM8 时，按以下顺序调整可减少 80% 故障：

1. **环境准备**：DM8 服务器安装 → 模式/用户/表空间 → 用 DTS 工具迁移数据 → 数据校验。
2. **驱动替换**：把所有第三方驱动换成 DmDriver（Java）、`sqlalchemy_dm`（Python）、`dmgorm2`（Go）、`DmProvider`（.NET）、`dmdb`（Node.js）、`pdo_dm`（PHP）。
3. **方言调整**：Hibernate 设 `DmDialect`、MyBatis 无需改、GORM V1 配 `CASE_SENSITIVE=N`、SQLAlchemy 改 URI 前缀。
4. **DDL 适配**：删除 `ENGINE=InnoDB` / `CHARSET=utf8mb4` / `DEFAULT CHARSET` 等 MySQL 专属段；`AUTO_INCREMENT` → `IDENTITY(1,1)`；`ON DUPLICATE KEY UPDATE` → `MERGE INTO`。
5. **DML 适配**：`GROUP_CONCAT` → `WM_CONCAT`；`IFNULL` 仍是 `NVL`；`LIMIT m, n` → `LIMIT n OFFSET m`；分页用 `ROWNUM` 时改成 `LIMIT`。
6. **PL/SQL 适配**：DM 支持大部分 Oracle PL/SQL 语法，但 `DBMS_OUTPUT.PUT_LINE` 行为略有不同；用 `disql` 工具调试。
7. **事务/锁**：DM 默认隔离级别 `READ COMMITTED`；如业务依赖 `REPEATABLE READ`，显式设置。
8. **连接池**：原 Druid/HikariCP 继续使用，只需把 driver 改 DmDriver。
9. **测试回归**：单元测试 + 集成测试 + 性能压测；重点关注 `LIMIT` 分页、`ON DUPLICATE KEY UPDATE`、外键约束、序列一致性。
10. **上线观察**：上线后 1 周内重点监控慢 SQL（`SELECT * FROM V$SQL WHERE TIME_USED > 1000`）、锁等待（`V$LOCK`）、表空间使用率。

## 十六、与其他国产数据库对比（应用层角度）

DM8、KingbaseES（人大金仓）、GaussDB（华为）、OceanBase 在应用层接入的差异：

| 维度 | DM8 | KingbaseES | GaussDB | OceanBase |
|------|-----|------------|---------|-----------|
| JDBC 驱动 | `DmDriver`（自研）| `kingbase8-8.x.jar` | `gsjdbc4.jar` | `oceanbase-client.jar`（兼容 MySQL/Oracle）|
| MyBatis 兼容 | ✅ 直接可用 | ✅ 需用 PG 方言 | ⚠️ 部分函数差异 | ✅ 走 MySQL 方言 |
| Hibernate Dialect | `DmDialect` | `Kingbase8Dialect` | `GaussDBDialect` | `OceanBaseMySQLDialect` |
| Python SQLAlchemy | `sqlalchemy_dm` | `sqlalchemy-kingbase` | ⚠️ 需自定义 | `mysql+pymysql`（兼容）|
| Go GORM | `dmgorm1/2`（自研）| `gorm-kingbase`（社区）| ⚠️ 自定义 | `gorm.io/driver/mysql` |
| 分库分表中间件 | MyCat / Sharding-JDBC | 自带 + Sharding-JDBC | 自带分布式 | 自身即分布式 |
| 应用层改造量 | 中 | 中 | 中-大 | 小（MySQL 模式）|

> DM 在应用层的特点是：**自研方言多，迁移工具完善，但生态比 OceanBase/KingbaseES 小**。建议：如果项目是从 Oracle 迁移、且需要国产化认证，DM8 是首选；如果是从 MySQL 迁移且要求零改造，OceanBase 兼容模式更省事。

## 十七、本章定位说明

本章**不重复** `app-dev.md` 中已展开的 JDBC / ODBC / DPI / Spring / MyBatis / MyBatis-Plus 等主流用法。**仅覆盖官方文档中独立成节的辅助框架**。如需主流 Java 框架细节，参见 `app-dev.md`；如需 DBA / 集群 / 备份等运维细节，参见 `ops.md` / `clusters.md` / `monitoring.md`。

## 十八、版本与生命周期

- 截至 2026-06，本章内容基于 DM8.x 各版本官方文档整理。
- DM 方言包版本（如 `DmDialect-for-hibernate5.6`、`sqlalchemy-dm 2.0.0`）需要**与上游 ORM 框架主版本号严格对齐**；升级 ORM 主版本前先查 DM 驱动兼容矩阵。
- 官方下载入口：[达梦云适配中心](https://eco.dameng.com/download/)。
