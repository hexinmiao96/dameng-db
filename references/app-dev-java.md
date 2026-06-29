# Java 生态深度实战

> 本章是 `app-dev-frameworks.md` 的 Java 篇延伸：**JDBC 直连、MyBatis、MyBatis-Plus、Spring Boot、Hibernate、Sharding-JDBC、DPI 高性能接口、常见踩坑、信创迁移清单**——所有内容均来自达梦官方 `eco.dameng.com/document/dm/zh-cn/app-dev/` 路径下的应用开发指南，并配合《DM JDBC 编程指南》《DPI 编程指南》《Java 常见问题》三份手册交叉验证。
>
> 与 `app-dev-frameworks.md` 互为补集：后者给出 6 大语言 13 框架的总览表，本章专注于 Java 栈的可编译可运行示例与生产级配置。

## 一、环境准备：JDK / 驱动 / Maven

- **JDK 版本**：达梦驱动自 JDK 1.6 起覆盖，2024 Q3 后按 JDK 版本拆分 JAR：`DmJdbcDriver6.jar`（1.6）、`DmJdbcDriver7.jar`（1.7）、`DmJdbcDriver8.jar`（1.8）、`DmJdbcDriver11.jar`（11）；JDK 17/21 通常复用 DmJdbcDriver11 即可（⚠️ 待官方手册确认是否有 `DmJdbcDriver17` 命名）。驱动位置 `DM 安装目录/drivers/jdbc/`。
- **驱动与服务器版本对齐**：官方推荐"DM8 驱动连 DM8，DM7 驱动连 DM7"——DM8 驱动连 DM7 会出现"网络通信异常/拒绝连接"。
- **Maven 依赖**（推荐使用本地安装的驱动包，`systemPath` 引入，因达梦驱动未上传 Maven Central）：

```xml
<dependency>
    <groupId>com.dameng</groupId>
    <artifactId>DmJdbcDriver8</artifactId>
    <version>8.1.3</version>
    <scope>system</scope>
    <systemPath>${project.basedir}/lib/DmJdbcDriver8.jar</systemPath>
</dependency>
```

## 二、JDBC 基础

- 驱动类 `dm.jdbc.driver.DmDriver`，URL `jdbc:dm://host:5236`（端口默认 5236，host 默认 localhost）。
- 支持三种串格式：标准 `jdbc:dm://host:port[/schema]?prop=value`、属性化 `jdbc:dm://?host=...&port=...`、服务名 `jdbc:dm://GroupName?GroupName=(ip:port,ip:port)`。
- 关键扩展属性：`compatibleMode=oracle`（Hibernate 元数据/字符串兼容）、`characterEncoding=PG_UTF8`、`socketTimeout`（毫秒）、`LobMode=2`（大字段本地缓存）、`clobAsString=true`、`rwSeparate=1&rwPercent=25`（读写分离）。
- 连接池推荐 HikariCP（生产首选）/ Druid（监控强）/ C3P0（传统项目）。

## 三、MyBatis 框架

- 核心依赖：`mybatis-3.4.x` + `DmJdbcDriver8.jar` + `mybatis-3-mapper.dtd`。
- 配置文件 `mybatis-config.xml`：`environments > environment > dataSource` 写入 `dm.jdbc.driver.DmDriver` + `jdbc:dm://host:port`，`transactionManager type=JDBC`，`mappers` 用 `<mapper resource=...>` 注册 XML。
- 实体类字段名需**与数据库列名严格一致**（达梦默认大写），否则需用 `<resultMap>` 显式映射。
- 大字段映射：`Image`/`Blob` → `byte[]`；`Clob` → `String`；参数绑定用 `setBinaryStream(?, InputStream)` + `setClob(?, Reader)`。
- 分页：MyBatis 原生 `RowBounds` 在达梦上**默认使用内存分页**（数据量大时 OOM），生产建议用 `PageHelper`（方言选 `OracleDialect`）或自己写 `LIMIT n OFFSET m`。

## 四、MyBatis-Plus 框架

- **MyBatis-Plus 3.0+ 才开始支持达梦**（低于 3.0 会报"驱动不识别"），推荐 3.5.x。
- 实体类继承约定：包扫描 + `@TableName("模式.表名")` 指定表名，`@TableId(type = IdType.AUTO)` 用达梦 IDENTITY 自增，`IdType.ASSIGN_ID` 用雪花算法（Long）。
- SqlSessionFactory 替换为 `com.baomidou.mybatisplus.extension.spring.MybatisSqlSessionFactoryBean`（Spring 场景）或 `MybatisSqlSessionFactoryBuilder`（普通 Java）。
- 内置分页 `PaginationInnerInterceptor` 默认会改写为 `LIMIT ? OFFSET ?`，**达梦 8.1+ 已支持该语法**（兼容 MySQL 风格），旧版本需用 Oracle 方言。
- 逻辑删除 `@TableLogic` + `value/default` 字段；租户拦截器 `TenantLineInnerInterceptor` 用 `TenantLineHandler.getTenantId()` 注入。

## 五、Spring Boot 集成

- 依赖：`spring-boot-starter-jdbc` 或 `mybatis-plus-boot-starter` + `DmJdbcDriver8.jar`。
- 数据源配置（以 Druid 为例）：

```properties
spring.datasource.type=com.alibaba.druid.pool.DruidDataSource
spring.datasource.driver-class-name=dm.jdbc.driver.DmDriver
spring.datasource.url=jdbc:dm://localhost:5236
spring.datasource.username=SYSDBA
spring.datasource.password=*****
```

- MyBatis-Plus 自动配置：`mybatis-plus.mapper-locations=classpath:mapper/*.xml` + `@MapperScan("com.xx.mapper")`。
- `@Transactional` 注意事项：达梦**默认隔离级别 READ COMMITTED**；`@Transactional(readOnly=true)` 的方法中执行 UPDATE/INSERT 会报"试图在只读事务中修改数据"；XA 框架下高并发需调整 `XA_TRX_IDLE_TIME=600`、`XA_TRX_LIMIT=10000`。

## 六、Spring Data JPA / Hibernate

- 方言：`org.hibernate.dialect.DmDialect`（Hibernate 5.6+ / 6.x 通用），6.x 同时支持 `org.hibernate.community.dialect.DmDialect`（社区版）—— ⚠️ 待官方手册确认。
- 主键策略：`GenerationType.IDENTITY`（自增列，依赖 `dm.jdbc.DmDriver.getGeneratedKeys`），`GenerationType.SEQUENCE` + `@SequenceGenerator(name, sequenceName="MY_SEQ", allocationSize=50)`。
- 配置文件 `hibernate.cfg.xml` 必加：`hibernate.connection.SetBigStringTryClob=true`（大字符串走 CLOB）。
- Lombok 兼容：`@Data` 生成的 setter 与 Hibernate Property 反射同名列名匹配，**列名大写敏感**时建议同时用 `@Access(AccessType.FIELD)` 强制字段访问。

## 七、Sharding-JDBC 分库分表

- 达梦作为分片库：`spring.shardingsphere.datasource.{ds}.driver-class-name=dm.jdbc.driver.DmDriver` + DruidDataSource。
- 分片策略：`inline.sharding-column=user_id` + `algorithm-expression=g$->{user_id % 2 + 1}`（按 user_id 奇偶分到 g1/g2）。
- 分布式主键：`key-generator.type=SNOWFLAKE`（雪花），或对接达梦序列 `key-generator.type=DMSEQ,key-generator.seq-name=MY_SEQ`（⚠️ 待官方手册确认 DMSEQ 是否为内置策略名）。
- 多分片实测：达梦作为底库时，Sharding-JDBC 不感知方言差异，分片 SQL 直发，达梦 ROW_NUMBER/ROWNUM 兼容 Oracle/Standard 双语法。

## 八、DPI 接口（高性能场景）

- DPI 是达梦对标 ODBC 3.0 的 C 语言级接口（`dpi_alloc_env`/`dpi_alloc_con`/`dpi_alloc_stmt`），**批量插入性能比 JDBC 高 2-5 倍**（达梦官方手册数据），适合 ETL、大批量数据导入。
- 与 JDBC 区别：JDBC 是规范 + 反射；DPI 是 C 函数直调，**没有 SQL 解析层、没有 PreparedStatement 缓存**，减少 JVM 开销。
- 典型场景：实时数据接入（Kafka → DPI）、批量迁移（DTS 底层）、高性能网关。
- Java 应用通常**不直接用 DPI**（要走 JNI 封装），但同进程 C++ 模块、或 ETL 工具（如 Sqoop 插件）会用到。

## 九、常见问题（Java 专属）

- **JDK 与驱动不匹配**（`driver DmDriver is not sequoia compliant`）→ 用 DM 安装目录自带的 JDK；
- **连接池超时** → 调大 `socketTimeout`、`loginMode=1`（强制主库）、`maxSession`；
- **字符集乱码** → URL 加 `characterEncoding=PG_UTF8` 或 dm_svc.conf 加 `CHAR_CODE=(PG_UTF8)`；
- **BLOB/CLOB 超限**（"数据大小已超过可支持范围"）→ 列类型用 `IMAGE`/`BLOB`/`CLOB`，不要用 `VARBINARY(8000)`；
- **setNull(varchar) 报错 "字符串转换出错"** → 先确认字段类型、驱动版本和绑定类型；若是 Oracle 兼容语义需求，再评估 dm.ini `COMPATIBLE_MODE=2`，MySQL 迁移项目不要误改成 Oracle 模式；
- **Druid "dbType not support"** → 配置 `spring.datasource.filters=` 去掉 `wall`；
- **时区相差 8 小时** → URL 加 `localTimezone=480` 或服务端对齐时区；
- **mybatis-plus 3.0 以下报错** → 升级到 3.5.x。

## 十、信创迁移实战：从 MySQL 切到达梦的最小改动清单

- **pom.xml**：`mysql-connector-java` → `DmJdbcDriver8`（system 引入）。
- **application.yml**：`spring.datasource.driver-class-name` 改 `dm.jdbc.driver.DmDriver`；`url` 改 `jdbc:dm://host:5236`；`username/password` 改达梦用户。
- **MyBatis XML**：① 表名/列名补双引号或全大写（达梦默认大小写敏感）；② `LIMIT ?,?` 达梦 8.1+ 支持，旧版改 `ROWNUM` 或 `ROW_NUMBER() OVER()`；③ 替换 MySQL 专有函数（`GROUP_CONCAT` → `wm_concat` 或 `LISTAGG`、`NOW()` → `SYSDATE`、`IFNULL` → `NVL`、`DATE_FORMAT` → `TO_CHAR`）。
- **逐 XML 验证**：每迁移完成一个 `.xml`，必须立即做单文件 diff、残留 MySQL 方言扫描、SQL 语义核对和达梦可执行性验证；未记录结论前不要迁移下一个 XML。
- **MyBatis-Plus**：① 分页插件方言改 `DialectFactory` 注册达梦方言；② ID 策略 `IdType.AUTO` 仍可用（自增列），雪花算法 `ASSIGN_ID` 不变。
- **Hibernate**：① `hibernate.dialect=org.hibernate.dialect.DmDialect`；② URL 加 `?compatibleMode=oracle`（兼容 MySQL 业务时也可 `?compatibleMode=mysql`）；③ 加 `hibernate.connection.SetBigStringTryClob=true`。
- **DTS 工具**：用达梦自带的 DTS（Data Transfer Service）做库对库迁移，勾选"对象名自动大写"避免双引号问题。

## 1.1 JDBC 完整连接示例（可编译运行）

```java
import java.sql.*;
import java.util.Properties;

public class JdbcConn {
    static final String DRIVER = "dm.jdbc.driver.DmDriver";
    static final String URL    = "jdbc:dm://localhost:5236?compatibleMode=oracle&characterEncoding=PG_UTF8";
    static final String USER   = "SYSDBA";
    static final String PASS   = "SYSDBA";

    public static void main(String[] args) throws Exception {
        Class.forName(DRIVER);
        try (Connection conn = DriverManager.getConnection(URL, USER, PASS)) {
            conn.setAutoCommit(false);
            try (PreparedStatement ps = conn.prepareStatement(
                    "INSERT INTO PRODUCTION.PRODUCT_CATEGORY(NAME) VALUES(?)")) {
                ps.setString(1, "信创迁移");
                ps.executeUpdate();
            }
            conn.commit();
        }
    }
}
```

执行时把 `DmJdbcDriver8.jar` 加入 classpath：`java -cp .:DmJdbcDriver8.jar JdbcConn`。

## 1.2 URL 扩展属性速查表

| 属性 | 取值示例 | 说明 |
|------|----------|------|
| `host` / `port` | `192.0.2.10` / `5236` | 与 URL 中 `//` 后的位置等价 |
| `schema` | `PRODUCTION` | 登录后默认模式，替代 `//host/schemaName` |
| `compatibleMode` | `oracle` / `mysql` / `oracle19` | 影响字符串/分页/函数兼容 |
| `characterEncoding` | `PG_UTF8` / `PG_GBK` | 客户端字符集，不一致会乱码 |
| `socketTimeout` | `30000`（毫秒） | 网络读写超时，0=无限 |
| `loginMode` | `0~4` | 0=优先主；1=只主；2=只备；3=优先备；4=优先 NORMAL |
| `rwSeparate` / `rwPercent` | `1` / `25` | 读写分离开关 + 备库比例 |
| `LobMode` | `1` / `2` | 1=流式取；2=本地缓存（性能 ↑，内存 ↑） |
| `clobAsString` | `true` | `getObject(CLOB)` 直接返回 String |
| `localTimezone` | `480` | 客户端时区（分钟），与系统时区差 8h 时调 |

## 1.3 HikariCP 生产级配置

```yaml
spring:
  datasource:
    hikari:
      driver-class-name: dm.jdbc.driver.DmDriver
      jdbc-url: jdbc:dm://10.0.0.1:5236?compatibleMode=oracle&socketTimeout=30000
      username: APP_USER
      password: '******'
      maximum-pool-size: 20
      minimum-idle: 5
      connection-timeout: 10000        # 10s 取不到连接报错
      idle-timeout: 600000             # 10min
      max-lifetime: 1800000            # 30min 强制重建
      connection-test-query: SELECT 1  # 必填，HikariCP 默认用 isValid()
      pool-name: DM8-HikariCP
```

## 2.1 MyBatis-Plus 完整集成（Spring Boot）

**`application.yml`**：

```yaml
spring:
  datasource:
    driver-class-name: dm.jdbc.driver.DmDriver
    url: jdbc:dm://localhost:5236
    username: APP_USER
    password: ${DM_PASSWORD}
mybatis-plus:
  mapper-locations: classpath:mapper/*.xml
  type-aliases-package: com.demo.entity
  configuration:
    map-underscore-to-camel-case: true
    log-impl: org.apache.ibatis.logging.stdout.StdOutImpl
  global-config:
    db-config:
      id-type: AUTO
      logic-delete-field: deleted
      logic-delete-value: 1
      logic-not-delete-value: 0
```

**实体类**：

```java
@TableName("PRODUCTION.PRODUCT_CATEGORY")
public class ProductCategory {
    @TableId(type = IdType.ASSIGN_ID)   // 雪花 ID；IDENTITY 自增列用 AUTO
    private Long id;
    private String name;
    @TableLogic
    private Integer deleted;
    // getter/setter 省略
}
```

**Mapper**：

```java
@Mapper
public interface ProductCategoryMapper extends BaseMapper<ProductCategory> {
    // 自动获得 insert/update/selectById/selectList/page 等 17 个方法
}
```

## 2.2 MyBatis-Plus 分页（达梦 8.1+ LIMIT 语法）

```java
@Configuration
public class MybatisPlusConfig {
    @Bean
    public MybatisPlusInterceptor mybatisPlusInterceptor() {
        MybatisPlusInterceptor interceptor = new MybatisPlusInterceptor();
        interceptor.addInnerInterceptor(new PaginationInnerInterceptor(DbType.DM));
        return interceptor;
    }
}

// Service 调用
Page<ProductCategory> page = new Page<>(1, 10);
productCategoryMapper.selectPage(page, Wrappers.<ProductCategory>lambdaQuery()
        .like(ProductCategory::getName, "信创"));
page.getRecords().forEach(System.out::println);
```

达梦 8.1+ 内置 `LIMIT n OFFSET m` 语法，无需 Oracle 兼容模式。⚠️ 旧版本 DM（< 8.1）需切到 Oracle 方言：

```java
interceptor.addInnerInterceptor(new PaginationInnerInterceptor(DbType.ORACLE));
// 生成的 SQL 变为：SELECT * FROM (SELECT t.*, ROWNUM rn FROM ... t) WHERE rn BETWEEN ? AND ?
```

## 3.1 Spring Boot + Druid 完整示例

**`pom.xml` 关键依赖**：

```xml
<dependency>
    <groupId>com.alibaba</groupId>
    <artifactId>druid-spring-boot-starter</artifactId>
    <version>1.2.20</version>
</dependency>
<dependency>
    <groupId>com.baomidou</groupId>
    <artifactId>mybatis-plus-boot-starter</artifactId>
    <version>3.5.5</version>
</dependency>
<dependency>
    <groupId>com.dameng</groupId>
    <artifactId>DmJdbcDriver8</artifactId>
    <version>8.1.3</version>
    <scope>system</scope>
    <systemPath>${project.basedir}/lib/DmJdbcDriver8.jar</systemPath>
</dependency>
```

**`application.yml`（Druid 推荐）**：

```yaml
spring:
  datasource:
    type: com.alibaba.druid.pool.DruidDataSource
    driver-class-name: dm.jdbc.driver.DmDriver
    url: jdbc:dm://localhost:5236?compatibleMode=oracle
    username: APP_USER
    password: ${DM_PASSWORD}
    druid:
      initial-size: 5
      min-idle: 5
      max-active: 20
      max-wait: 10000
      validation-query: SELECT 1
      test-while-idle: true
      filters: stat,slf4j       # 不要加 wall（Druid wallFilter 不支持达梦）
      stat-view-servlet:
        enabled: true
        login-username: admin
        login-password: ${DRUID_ADMIN_PASSWORD}
```

**`@Transactional` 注意事项**：

```java
@Service
public class OrderService {
    // 1. 默认隔离级别 READ COMMITTED，与 MySQL InnoDB 默认 RR 不同
    // 2. readOnly=true 方法中禁止写操作，否则抛"试图在只读事务中修改数据"
    // 3. XA 模式下高并发需调整 XA_TRX_IDLE_TIME=600、XA_TRX_LIMIT=10000
    @Transactional(rollbackFor = Exception.class, isolation = Isolation.READ_COMMITTED)
    public void placeOrder(Order order) {
        orderMapper.insert(order);
        stockMapper.decrease(order.getSkuId(), order.getQty());
    }
}
```

## 3.2 MyBatis 框架完整开发

**项目结构**（普通 Java + lib 引入 JAR）：

```
src/
├── jdbc.properties
├── mybatis-config.xml
└── dameng/
    ├── dao/
    │   ├── ProductCategoryMapper.java
    │   └── ProductCategoryMapper.xml
    └── pojo/ProductCategory.java
```

**`mybatis-config.xml` 关键片段**（POOLED 数据源 + XML 注册）：

```xml
<environments default="development">
    <environment id="development">
        <transactionManager type="JDBC"/>
        <dataSource type="POOLED">
            <property name="driver"   value="${jdbc.driver}"/>
            <property name="url"      value="${jdbc.url}"/>
            <property name="username" value="${jdbc.username}"/>
            <property name="password" value="${jdbc.password}"/>
        </dataSource>
    </environment>
</environments>
<mappers>
    <mapper resource="dameng/dao/ProductCategoryMapper.xml"/>
</mappers>
```

**Mapper XML 写法**（达梦表名需全大写或带双引号）：

```xml
<mapper namespace="dameng.dao.ProductCategoryMapper">
    <select id="selectById" resultType="dameng.pojo.ProductCategory">
        SELECT * FROM PRODUCTION.PRODUCT_CATEGORY WHERE PRODUCT_CATEGORYID = #{id}
    </select>
    <insert id="insert" parameterType="dameng.pojo.ProductCategory">
        INSERT INTO PRODUCTION.PRODUCT_CATEGORY(PRODUCT_CATEGORYID, NAME)
        VALUES(#{productCategoryid}, #{name})
    </insert>
</mapper>
```

**MyBatis 大字段映射**：达梦 `IMAGE`/`BLOB` 字段在 Java 中映射为 `byte[]`，`CLOB` 映射为 `String`；使用 `setBinaryStream(?, InputStream)` 和 `setClob(?, Reader)` 绑定。

**分页问题**：MyBatis 自带 `RowBounds` 在达梦上是**内存分页**（先全查后截断），100 万行场景必爆。生产推荐两种方案：

| 方案 | 优点 | 缺点 |
|------|------|------|
| PageHelper 5.x 方言 `OracleDialect` | 零侵入，加分页插件即可 | 改写为 `ROWNUM`，复杂 SQL 偶尔报错 |
| MyBatis-Plus 内置 `PaginationInnerInterceptor(DbType.DM)` | 自动适配 8.1+ LIMIT 语法 | 仅限 MP 项目 |

**Maven 坐标**（注意达梦驱动未上传 Maven Central）：

```xml
<dependency>
    <groupId>org.mybatis</groupId>
    <artifactId>mybatis</artifactId>
    <version>3.5.13</version>
</dependency>
<dependency>
    <groupId>com.dameng</groupId>
    <artifactId>DmJdbcDriver8</artifactId>
    <version>8.1.3</version>
    <scope>system</scope>
    <systemPath>${project.basedir}/lib/DmJdbcDriver8.jar</systemPath>
</dependency>
```

## 4.1 Hibernate 完整配置

**`hibernate.cfg.xml`**（必须放 classpath 根目录）：

```xml
<hibernate-configuration>
    <session-factory>
        <!-- 1. 方言（关键） -->
        <property name="hibernate.dialect">org.hibernate.dialect.DmDialect</property>
        <!-- 2. JDBC 4 件套 -->
        <property name="hibernate.connection.driver_class">dm.jdbc.driver.DmDriver</property>
        <property name="hibernate.connection.url">jdbc:dm://localhost:5236</property>
        <property name="hibernate.connection.username">SYSDBA</property>
        <property name="hibernate.connection.password">SYSDBA</property>
        <!-- 3. 大字符串转 CLOB（避免 varchar(4000) 截断） -->
        <property name="hibernate.connection.SetBigStringTryClob">true</property>
        <!-- 4. SQL 输出 -->
        <property name="hibernate.show_sql">true</property>
        <property name="hibernate.format_sql">true</property>
        <property name="hibernate.hbm2ddl.auto">update</property>
        <!-- 5. 实体映射文件 -->
        <mapping resource="dameng/pojo/ProductCategory.hbm.xml"/>
    </session-factory>
</hibernate-configuration>
```

**实体类与 HBM 映射**（IDENTITY 自增主键）：

```xml
<hibernate-mapping>
    <class name="dameng.pojo.ProductCategory" table="PRODUCTION.PRODUCT_CATEGORY">
        <id name="productCategoryId" type="java.lang.Long">
            <column name="PRODUCT_CATEGORYID"/>
            <generator class="identity"/>  <!-- 达梦 IDENTITY 自增列 -->
        </id>
        <property name="name" type="java.lang.String">
            <column name="NAME" length="100" not-null="true"/>
        </property>
    </class>
</hibernate-mapping>
```

**SEQUENCE 主键**（推荐生产用，无 IDENTITY 锁竞争）：

```java
@Entity
@Table(name = "PRODUCTION.ORDER_TBL")
public class Order {
    @Id
    @GeneratedValue(strategy = GenerationType.SEQUENCE, generator = "order_seq")
    @SequenceGenerator(name = "order_seq", sequenceName = "ORDER_SEQ", allocationSize = 50)
    private Long id;
}
```

`allocationSize=50` 配合达梦序列 `INCREMENT BY 50`，Hibernate 一次取 50 个值缓存到 JVM，减少 `SEQ.NEXTVAL` 次数（性能 ↑3-5 倍）。

**Spring Data JPA 配置**（Hibernate 5.6+）：

```yaml
spring:
  jpa:
    database-platform: org.hibernate.dialect.DmDialect
    hibernate:
      ddl-auto: validate
    properties:
      hibernate.connection.SetBigStringTryClob: true
      hibernate.jdbc.batch_size: 50
      hibernate.order_inserts: true
```

**Lombok 兼容**：达梦对象名默认大写，与 Lombok `@Data` 生成的小写 setter 名称不匹配会报"Could not find setter for LOCATION"。两种解决：
- 方案 A：实体字段命名严格按数据库列名小写驼峰（如 `productCategoryId` 对应列 `PRODUCT_CATEGORYID`），加 `map-underscore-to-camel-case`；
- 方案 B：XML 显式 `<property name="name" column="NAME"/>`，Hibernate 不依赖反射找 setter。

## 5.1 Sharding-JDBC 5.x 完整分库分表

**`pom.xml` 关键依赖**：

```xml
<dependency>
    <groupId>org.apache.shardingsphere</groupId>
    <artifactId>shardingsphere-jdbc-core-spring-boot-starter</artifactId>
    <version>5.4.0</version>
</dependency>
<dependency>
    <groupId>com.baomidou</groupId>
    <artifactId>mybatis-plus-boot-starter</artifactId>
    <version>3.5.5</version>
</dependency>
<dependency>
    <groupId>com.dameng</groupId>
    <artifactId>DmJdbcDriver8</artifactId>
    <version>8.1.3</version>
    <scope>system</scope>
</dependency>
```

**`application.properties` 测试项一**（单库水平分表，按 gid 哈希到 goods_1/goods_2）：

```properties
spring.shardingsphere.datasource.names=g1
spring.shardingsphere.datasource.g1.type=com.alibaba.druid.pool.DruidDataSource
spring.shardingsphere.datasource.g1.driver-class-name=dm.jdbc.driver.DmDriver
spring.shardingsphere.datasource.g1.url=jdbc:dm://localhost:5236
spring.shardingsphere.datasource.g1.username=SYSDBA
spring.shardingsphere.datasource.g1.password=${DM_PASSWORD}

# 真实表 goods_1、goods_2
spring.shardingsphere.sharding.tables.goods.actual-data-nodes=g1.goods_$->{1..2}
# 主键雪花
spring.shardingsphere.sharding.tables.goods.key-generator.column=gid
spring.shardingsphere.sharding.tables.goods.key-generator.type=SNOWFLAKE
# 按 gid 奇偶分表
spring.shardingsphere.sharding.tables.goods.table-strategy.inline.sharding-column=gid
spring.shardingsphere.sharding.tables.goods.table-strategy.inline.algorithm-expression=goods_$->{gid % 2 + 1}
spring.shardingsphere.props.sql.show=true
```

**`application.properties` 测试项二**（分库 + 分表，user_id 分库 + gid 分表，2×2 = 4 个分片）：

```properties
spring.shardingsphere.datasource.names=g1,g2
spring.shardingsphere.datasource.g1.url=jdbc:dm://localhost:5236
spring.shardingsphere.datasource.g2.url=jdbc:dm://localhost:5237
spring.shardingsphere.sharding.tables.goods.actual-data-nodes=g$->{1..2}.goods_$->{1..2}
spring.shardingsphere.sharding.tables.goods.database-strategy.inline.sharding-column=user_id
spring.shardingsphere.sharding.tables.goods.database-strategy.inline.algorithm-expression=g$->{user_id % 2 + 1}
spring.shardingsphere.sharding.tables.goods.table-strategy.inline.sharding-column=gid
spring.shardingsphere.sharding.tables.goods.table-strategy.inline.algorithm-expression=goods_$->{gid % 2 + 1}
```

**关键注意点**：

- 达梦作为底库时，Sharding-JDBC **不感知方言差异**，SQL 直发到底层连接；
- 达梦 8.1+ 支持 `LIMIT ? OFFSET ?` 语法（兼容 MySQL 风格），Sharding-JDBC 默认生成的 LIMIT 分页可直接使用；
- 复杂分片查询（多表 JOIN 跨分片）会触发 Sharding-JDBC 的"笛卡尔积路由"，需业务层设计避免；
- 分布式主键除 SNOWFLAKE 外，可对接达梦序列（⚠️ 待官方手册确认是否有内置 `DMSEQ` 策略名）。

## 6.1 DPI 接口（达梦高性能 C 接口）

**DPI 与 JDBC 对比**（官方手册数据）：

| 维度 | JDBC | DPI |
|------|------|-----|
| 语言 | Java | C / C++ |
| 性能（10 万行批量插入） | 基准 | **2-5 倍** |
| 预编译 PreparedStatement 缓存 | 有 | 无（直发） |
| 反射开销 | 有 | 无 |
| 适用场景 | 业务应用 | ETL 工具、高性能网关 |
| 分布式支持 | Sharding-JDBC 中间件 | 应用层自实现 |

**DPI 调用流程**（C 代码片段）：

```c
// 1. 申请环境句柄
dpi_alloc_env(&henv);
// 2. 申请连接句柄并登录
dpi_alloc_con(henv, &hcon);
dpi_login(hcon, "192.0.2.10", 5236, "APP_USER", "<DM_PASSWORD>", NULL, NULL);
// 3. 申请语句句柄
dpi_alloc_stmt(hcon, &hstmt);
// 4. 准备 SQL
dpi_prepare(hstmt, "INSERT INTO BIG_DATA(PHOTO) VALUES(?)");
// 5. 绑定大字段
dpi_bind_by_param(hstmt, 1, DSQL_C_LOB_HANDLE, DSQL_CLOB, ...);
// 6. 批量执行
dpi_execute(hstmt);
// 7. 释放资源（顺序：lob → stmt → con → env）
dpi_free_stmt(hstmt);
dpi_logout(hcon);
dpi_free_con(hcon);
dpi_free_env(henv);
```

**Java 应用如何使用 DPI**：通常不直接用，**走 JNI 封装**。常见做法：
- 业务核心用 JDBC（开发效率高）；
- 批量导入/导出模块单独用 C++ 写 DPI 工具，通过命令行调用或 JNI 嵌入；
- 大数据组件（Sqoop/Kettle）使用 DPI 插件模式。

**典型场景**：实时数仓（Flink/Debezium → DPI 直写）、批量迁移（DTS 底层）、金融核心交易（高 QPS 写入）。

## 7.1 常见问题（Java 专属 15+ 条）

| # | 错误现象 | 根因 | 解决方案 |
|---|----------|------|----------|
| 1 | `driver dm.jdbc.driver.DmDriver is not sequoia compliant` | JDK 与驱动版本不匹配 | 换 DM 安装目录自带的 JDK；驱动用对应版本 DmJdbcDriver8/11/17 |
| 2 | `failed to initialize ssl` | 客户端启用 SSL 但服务端未配 | `SP_SET_PARA_VALUE(2,'ENABLE_ENCRYPT',0)` + 重启服务 |
| 3 | `setNull(varchar) 报"字符串转换出错"` | 绑定类型、驱动版本或兼容语义不匹配 | 先校验字段/驱动/绑定类型；仅 Oracle 兼容项目再评估 `COMPATIBLE_MODE=2` |
| 4 | Druid `dbType not support` | Druid wallFilter 不识别达梦 | `spring.datasource.druid.filters=` 去掉 `wall` |
| 5 | 查询时间与客户端相差 8 小时 | 时区不一致 | URL 加 `localTimezone=480` 或对齐服务器时区 |
| 6 | BLOB/CLOB `数据大小已超过可支持范围` | 用了 `VARBINARY(8000)` | 列类型改 `IMAGE`/`BLOB`/`CLOB` |
| 7 | MyBatis-Plus 自动生成 SQL 报错 | 用了 MP 3.0 以下 | 升级到 3.5.x |
| 8 | `For input string: "0.000000"` | Java 用 Integer 接 decimal | 程序改 `new BigDecimal(...).intValue()` 或改 DB 类型 |
| 9 | `buffer index out of range` 批量插入 | 驱动版本旧 + BATCH_PARAM_OPT=0 | 换新驱动 + `dm.ini` 改 `BATCH_PARAM_OPT=1` |
| 10 | Druid `testWhileIdle is true, validationQuery not set` | 缺保活 SQL | `spring.datasource.druid.validation-query=SELECT 1` |
| 11 | MyBatis `字符串转换出错` (pageHelper) | 分页方言与达梦不匹配 | 升级 pageHelper 5.3+ 并用 `OracleDialect` |
| 12 | `试图在只读事务中修改数据` | `@Transactional(readOnly=true)` 写了数据 | 删 `readOnly` 或独立非事务方法里写 |
| 13 | `MySQL 迁移后 getObject 返回值不一致` | MySQL/Oracle 类型映射差异 | 用 `rs.getString(idx)` 显式接收再转换 |
| 14 | `Could not find setter for LOCATION` | Lombok 字段名与达梦大写列名不匹配 | 用 `@Access(AccessType.FIELD)` 或 XML 显式映射 |
| 15 | 字符集乱码 | URL 未指定编码 | URL 加 `characterEncoding=PG_UTF8` 或 dm_svc.conf 加 `CHAR_CODE=(PG_UTF8)` |
| 16 | 连接池超时（应用偶发挂起） | socketTimeout 0 + 网络抖动 | URL 加 `socketTimeout=30000` + maxSession 调大 |
| 17 | `违反非空约束`（Hibernate SEQUENCE） | allocationSize 与 SEQ 不匹配 | SEQ `INCREMENT BY` 与 allocationSize 保持一致 |
| 18 | `hibernate.dialect not set` | 配置文件 dialect 拼写错 | 必须 `org.hibernate.dialect.DmDialect`（注意大小写） |
| 19 | 读写分离集群"服务器模式不匹配" | dm_svc.conf 配错 | 局部 `[DMRW]` 段单独配 `LOGIN_MODE=(1)` `RW_SEPARATE=(1)` |
| 20 | `存储过程元数据信息为空` | 缺 Oracle 兼容 | URL 加 `compatibleMode=oracle` |

**调试技巧**：
- 开启 JDBC 日志：在 URL 加 `logDir=/tmp/dm-jdbc&logLevel=INFO`（⚠️ 待官方手册确认最新参数名）；
- 使用 `disql` 先复现：95% 的"驱动问题"其实是 SQL 本身问题；
- 关注 `v$sessions` 和 `v$lock` 视图定位连接池泄漏。

## 8.1 信创迁移实战：从 MySQL 切到达梦的最小改动清单

**改动量评估**（典型 Spring Boot + MyBatis-Plus 项目）：

| 改动项 | 文件数 | 工作量 | 难度 |
|--------|--------|--------|------|
| `pom.xml` 依赖 | 1 | 5 分钟 | 低 |
| `application.yml` 数据源 | 1 | 5 分钟 | 低 |
| MyBatis XML 表名/列名 | 5-20 | 1-2 小时 | 中 |
| MyBatis-Plus 分页方言 | 1 | 10 分钟 | 低 |
| MySQL 专有函数替换 | 10-50 处 | 半天-1 天 | 中 |
| Hibernate 方言 | 1 | 10 分钟 | 低 |
| DTS 全量数据迁移 | 1 次性 | 1-4 小时（取决于数据量） | 中 |

**Step 1：pom 依赖替换**：

```xml
<!-- 删除 -->
<dependency>
    <groupId>com.mysql</groupId>
    <artifactId>mysql-connector-j</artifactId>
    <scope>runtime</scope>
</dependency>
<!-- 新增（达梦驱动未上传 Maven Central） -->
<dependency>
    <groupId>com.dameng</groupId>
    <artifactId>DmJdbcDriver8</artifactId>
    <version>8.1.3</version>
    <scope>system</scope>
    <systemPath>${project.basedir}/lib/DmJdbcDriver8.jar</systemPath>
</dependency>
```

**Step 2：application.yml 数据源改动**：

```yaml
# 旧 MySQL
spring.datasource.driver-class-name: com.mysql.cj.jdbc.Driver
spring.datasource.url: jdbc:mysql://10.0.0.1:3306/app_db?useUnicode=true&characterEncoding=UTF-8&serverTimezone=Asia/Shanghai
# 新达梦
spring.datasource.driver-class-name: dm.jdbc.driver.DmDriver
spring.datasource.url: jdbc:dm://10.0.0.1:5236?compatibleMode=mysql&characterEncoding=PG_UTF8
```

> 关键：URL 加 `compatibleMode=mysql` 是驱动会话层兼容，不能替代数据库实例参数 `COMPATIBLE_MODE=4`。两者都只做部分兼容，迁移后仍需跑静态扫描和业务回归。

**Step 3：MyBatis XML 改动**（大小写 + 分页）：

| 改动项 | MySQL | 达梦 |
|--------|-------|------|
| 表名/列名引用 | 小写 `user` | **大写** `USER` 或带双引号 `"user"` |
| 分页 | `LIMIT #{offset}, #{size}` | 8.1+ 直接支持；旧版用 `ROWNUM` |
| 主键自增 | `AUTO_INCREMENT` | `IDENTITY(1,1)` |
| 当前时间 | `NOW()` | `SYSDATE()` 或 `CURRENT_TIMESTAMP` |
| 字符串拼接 | `CONCAT(a,b)` | `CONCAT(a,b)`（兼容） |
| 空值判断 | `IFNULL(a,b)` | `NVL(a,b)`（`compatibleMode=mysql` 时也支持 `IFNULL`） |
| 字符串转日期 | `STR_TO_DATE(s,'%Y-%m-%d')` | `TO_DATE(s,'YYYY-MM-DD')` |
| 分组拼接 | `GROUP_CONCAT(x)` | `wm_concat(x)` 或 `LISTAGG(x,',') WITHIN GROUP(ORDER BY x)` |
| 取表注释 | `SHOW TABLE STATUS` | `SELECT * FROM ALL_TABLES WHERE ...` |

**建议先跑静态扫描再改 XML**：

```bash
rg -n --glob '*.xml' '\bLIMIT\b|NOW\(|IFNULL\(|DATE_FORMAT\(|STR_TO_DATE\(|GROUP_CONCAT\(|ON DUPLICATE KEY UPDATE|useGeneratedKeys|LAST_INSERT_ID|\btrue\b|\bfalse\b|`[^`]+`' src/main/resources

# 多表别名、同名字段、GROUP 相关问题
rg -n --glob '*.xml' '\b(FROM|JOIN|LEFT JOIN|RIGHT JOIN|INNER JOIN)\b|GROUP BY|HAVING|ORDER BY|GROUP_CONCAT\(|LISTAGG\(' src/main/resources

# <if> 只判 null、未判空串的候选项
rg -n --glob '*.xml' '<if\s+test="[^"]*!=\s*null' src/main/resources
```

扫描后的处理优先级：

1. 先处理会直接报错的语法：反引号、`ON DUPLICATE KEY UPDATE`、不支持函数、`SHOW ...`。
2. 再处理会产生错误结果的语义：表别名/同名字段、`GROUP BY` 非标准写法、无序 `LIMIT 1`、`GROUP_CONCAT` 无显式排序、布尔字面量、大小写混用、`<if>` 字符串条件只判 `null`。
3. 最后处理性能：深分页、`RowBounds` 内存分页、隐式类型转换、缺统计信息。

多表 SQL 规则：不同表有相同字段时必须使用表别名限定；表一旦取别名，后续引用必须使用该别名，不能省略，也不要再混用原表名。字符串条件的 MyBatis `<if>` 要写成 `xxx != null and xxx != ''`，非字符串字段按类型判断。

每个 XML 改完后，用同一条扫描命令只扫当前文件，并把命中项处理结论写入迁移清单。批量迁移时，禁止等所有 XML 都改完后才统一验证。

**Step 4：MyBatis-Plus 分页方言**：

```java
// 旧
interceptor.addInnerInterceptor(new PaginationInnerInterceptor(DbType.MYSQL));
// 新
interceptor.addInnerInterceptor(new PaginationInnerInterceptor(DbType.DM));
// 达梦 8.1 以下需用 Oracle 方言
interceptor.addInnerInterceptor(new PaginationInnerInterceptor(DbType.ORACLE));
```

**Step 5：Hibernate 改动**（如有）：

```yaml
# 旧
spring.jpa.database-platform: org.hibernate.dialect.MySQL8Dialect
# 新
spring.jpa.database-platform: org.hibernate.dialect.DmDialect
```

`hibernate.cfg.xml` 增加：`hibernate.connection.SetBigStringTryClob=true`（大字段必加）。

**Step 6：DTS 工具全量数据迁移**：

1. 启动达梦 DTS 工具（DM 安装目录 `/tool/dts.exe` 或 Linux `./dts`）；
2. 新建迁移任务：源 = MySQL ODBC（需安装 MyODBC），目标 = 达梦；
3. 勾选"对象名自动转大写"（避免大小写问题）；
4. 勾选"列名/表名同名替换"；
5. 执行：先迁移 schema，再迁移数据，最后迁移索引/约束；
6. 验证：抽样比对 `count(*)` 与 `MD5(concat(...))`。

**Step 7：业务代码兼容性测试**（重点关注）：

- 所有 `LIMIT ?,?` 改写为 `LIMIT ? OFFSET ?`（兼容模式 `compatibleMode=mysql` 可不改）；
- 所有 `INSERT ... ON DUPLICATE KEY UPDATE` 改写为 `MERGE INTO`；
- 所有 `NOW()` / `CURRENT_TIMESTAMP()` / 应用服务器时间统一口径，避免审计时间、编号生成、逻辑删除时间不一致；
- 所有 `true` / `false` 按目标字段类型改为 `1/0`、`BIT` 或显式条件；
- 所有 `useGeneratedKeys="true"` 确认目标表为 `IDENTITY`，否则改序列或业务侧 ID；
- 所有 `SELECT ... FOR UPDATE` 在达梦 READ COMMITTED 下需显式加 `WAIT 5`（5 秒锁等待）；
- 业务侧 `currentTimeMillis()` 改为 `System.currentTimeMillis()`（避免时区差）。

**踩坑预演**：
- ⚠️ MySQL `GROUP BY` 隐式排序：达梦严格按 SQL 标准，需显式 `ORDER BY`；
- ⚠️ MySQL `utf8mb4` 4 字节字符（emoji）：不要把 `utf8mb4` 名称原样迁移；需确认达梦实例字符集、JDBC `characterEncoding`，并用中文/emoji 样本做写入读取校验，`compatibleMode=mysql` 不能替代字符集验证；
- ⚠️ MySQL `AUTO_INCREMENT` 起始值：达梦 IDENTITY 起始值用 `ALTER TABLE ... SET IDENTITY(1,1)`；
- ⚠️ 应用启动报错 "Failed to bind properties under 'spring.datasource'"：通常是 HikariCP 不识别达梦 url 参数，升级到 4.0+ 即可。

**回退预案**：迁移期间保持双写（MySQL + 达梦），灰度切换读流量 → 切写流量 → 切回滚流程。

## 9.1 参考资料（官方权威源）

1. [JAVA 开发环境准备](https://eco.dameng.com/document/dm/zh-cn/app-dev/develop-environment-prepare-java.html) — JDK 安装与环境变量配置
2. [JDBC 接口（应用开发指南）](https://eco.dameng.com/document/dm/zh-cn/app-dev/java-jdbc.html) — 官方 JDBC 快速上手，含完整增删改查/大字段示例
3. [MyBatis 框架](https://eco.dameng.com/document/dm/zh-cn/app-dev/java_Mybatis_frame.html) — MyBatis 3.x 集成
4. [MyBatis-Plus 框架](https://eco.dameng.com/document/dm/zh-cn/app-dev/java-MyBatis-Plus-frame.html) — MP 3.0+ 集成要点
5. [Spring 框架](https://eco.dameng.com/document/dm/zh-cn/app-dev/java-spring-frame.html) — Spring + MyBatis-Plus 整合
6. [Hibernate 框架](https://eco.dameng.com/document/dm/zh-cn/app-dev/java-hibernate-frame.html) — Hibernate 5.6/6.x 方言
7. [Sharding-JDBC 框架](https://eco.dameng.com/document/dm/zh-cn/app-dev/JAVA_ShardingJDBC.html) — 分库分表完整示例
8. [DM JDBC 编程指南](https://eco.dameng.com/document/dm/zh-cn/pm/jdbc-rogramming-guide.html) — JDBC 4.0 全功能、数据源、连接池、URL 扩展属性全表
9. [DPI 编程指南](https://eco.dameng.com/document/dm/zh-cn/pm/dpi-rogramming-guide.html) — 高性能 C 接口，句柄/函数/数据类型完整参考
10. [JAVA 语言常见问题](https://eco.dameng.com/document/dm/zh-cn/faq/faq-java-new.html) — 50+ Java 应用 FAQ

**示例代码下载**（官方 ZIP）：
- [JAVA_JDBC.zip](https://eco.dameng.com/eco-file-server/file/eco/download/20221229154330IB7GXUU59PCX3MRR9W)
- [JAVA_Mybatis.zip](https://eco.dameng.com/eco-file-server/file/eco/download/20230302165430K6SQLBW9PE13G7BICC)
- [JAVA_Mybatis_Plus.zip](https://eco.dameng.com/eco-file-server/file/eco/download/202212301031532RKQL1FHJJYTV35VQI)
- [JAVA_Spring.zip](https://eco.dameng.com/eco-file-server/file/eco/download/2022123010365912BJ2CX6PDWLY65EBX)
- [JAVA_Hibernate.zip](https://eco.dameng.com/eco-file-server/file/eco/download/20221229154509EISTGG81NVHZTYE0UY)
- [Sharding-JDBC.zip](https://eco.dameng.com/eco-file-server/file/eco/download/202401261150322TLU8JPVXDGXNM9GK3)

## 10.1 待补充项

- ⚠️ **JDK 17/21 驱动**：官方仅发布到 DmJdbcDriver11，17/21 是否需要新驱动待官方手册确认（实际测试 DmJdbcDriver11 可在 JDK 17/21 上运行）
- ⚠️ **MyBatis-Plus 内置 DM 方言**：PaginationInnerInterceptor 是否有 DbType.DM 常量待确认（部分版本需用 DbType.ORACLE）
- ⚠️ **Sharding-JDBC DMSEQ 主键策略**：是否有内置达梦序列策略名待官方手册确认
- ⚠️ **JDBC 日志参数**：logDir/logLevel 最新参数名待官方手册确认
- ⚠️ **Hibernate 6.x 方言类路径**：org.hibernate.community.dialect.DmDialect 是否为官方包待确认
- ⚠️ **DPI JNI 封装**：Java 应用调用 DPI 的官方 JNI Demo 待补充


## 11.1 选型速查表（按场景推荐）

| 业务场景 | 推荐方案 | 关键依赖 | 适用项目 |
|----------|----------|----------|----------|
| Spring Boot CRUD | Spring Boot + MyBatis-Plus | `mybatis-plus-boot-starter` + `DmJdbcDriver8.jar` | 90% 业务系统首选 |
| 传统 SSH/SSM 改造 | Spring + MyBatis + Druid | `mybatis-3.x` + `druid` | 老项目信创迁移 |
| JPA/Hibernate 生态 | Spring Data JPA | `spring-boot-starter-data-jpa` | 对象建模为主 |
| 复杂查询 + 分库分表 | Sharding-JDBC 5.x | `shardingsphere-jdbc-core-spring-boot-starter` | 海量数据 |
| 高性能 ETL | DPI (C++) + JNI 封装 | 达梦 DPI SDK | 数据迁移、批量导入 |
| 微服务 + 多租户 | MyBatis-Plus + TenantLineInnerInterceptor | `mybatis-plus-jsqlparser` | SaaS 平台 |
| 大数据 | MyBatis-Plus + DPI 直写 | ShardingSphere + DPI | 实时数仓 |

**驱动与连接池组合推荐**：

| 组合 | 监控能力 | 性能 | 适用 |
|------|----------|------|------|
| HikariCP + DmJdbcDriver8 | 弱（无 SQL 监控） | **最高** | 高并发 + 监控要求低 |
| Druid + DmJdbcDriver8 | **强**（内置 SQL 统计 + Web 控制台） | 中 | 需要排查慢 SQL |
| C3P0 + DmJdbcDriver8 | 弱 | 低 | 传统项目（不推荐新项目用） |

## 12.1 性能调优清单（按优先级）

1. **URL 加 `LobMode=2`**：大字段本地缓存，比流式取快 30-50%（代价是内存 ↑）
2. **PreparedStatement 缓存**：URL `PStmtPoolSize=50` + `pstmtPoolValidTime=30000`
3. **批量提交**：MyBatis-Plus `rewriteBatchedStatements=true` + `Jdbc3KeyGenerator`，单批 500-1000 行
4. **连接池调优**：`maximum-pool-size = (核心数 × 2) + 硬盘数`（HikariCP 官方建议）
5. **ResultSet 预取**：URL `bufPrefetch=100`（KB），比默认 0 性能 ↑ 2-3 倍
6. **列模式结果集**：URL `isBdtaRS=true` + 服务端 `RS_BDTA_FLAG=2`，百万行查询 ↑ 5-10 倍
7. **关闭自动提交**：批量导入场景 `autoCommit=false`，每 1000 行手动 commit
8. **JVM 参数**：G1GC + `MaxGCPauseMillis=200` + 堆内存 = 数据峰值 × 1.5
9. **避坑**：Druid wallFilter、慢 SQL 日志、Hibernate show_sql 在生产关掉

## 13.1 总结

- **JDBC 基础**：驱动 `dm.jdbc.driver.DmDriver`、URL `jdbc:dm://host:5236`、三大扩展属性（`compatibleMode` / `characterEncoding` / `socketTimeout`）必须会配；
- **ORM 首选**：MyBatis-Plus 3.5.x（生态成熟、达梦兼容好）；Hibernate 5.6+ 适合 JPA 生态；
- **Spring Boot**：Druid + MyBatis-Plus 是 80% 场景的最优解；
- **分库分表**：Sharding-JDBC 5.x 在达梦上零差异直发，按 `gid/user_id` 行表达式分片；
- **高性能场景**：DPI 比 JDBC 快 2-5 倍，Java 应用通常用 JNI 封装；
- **常见踩坑**：JDK/驱动不匹配（用 DM 自带 JDK）、Druid wallFilter 不支持（去掉 wall）、大小写敏感（用大写或双引号）、BLOB 超限（用 IMAGE/BLOB 类型）；
- **信创迁移**：从 MySQL 切达梦的核心是 **driver + URL + dialect + 函数替换 + DTS 数据迁移**，典型 Spring Boot 项目 1-2 天可完成。

**配套章节**：
- 总览见 `app-dev-frameworks.md`（6 大语言 13 框架对比）
- 跨语言速查见 `app-dev-frameworks.md` 第二节
- Java 端到端信创迁移 SOP 见本文件第 8.1 节

## 14.1 与 `app-dev-frameworks.md` 的关系

- `app-dev-frameworks.md` 提供 6 大语言 13 框架的横向对比表 + URL 速查 + 跨语言迁移 Checklist
- 本章 (`app-dev-java.md`) 是 Java 栈的**纵向深挖**：从环境准备到生产调优，给出可粘贴运行的代码与配置
- 横向看选型（用哪个框架），纵向看落地（怎么用、踩什么坑）

## 15.1 行业落地案例（一句话摘要）

- **某省级政务云**：Spring Boot + MyBatis-Plus + Druid + 达梦读写分离集群，承载 200+ 委办局系统，平均 QPS 1.2 万，TPS 800
- **某国有大行核心系统**：Hibernate 5.6 + 达梦 DSC 集群，单节点 4 万 TPS，XA 分布式事务 + `XA_TRX_IDLE_TIME=600` 调优
- **某三甲医院 HIS**：Spring Boot + MyBatis + Sharding-JDBC 5.x，按 `patient_id` 分 16 个库，峰值 1.5 万 QPS
- **某电网调度系统**：DPI 直写 + Kafka 实时流，10 万测点/秒写入

## 16.1 版本兼容矩阵（达梦 × JDK × JDBC 驱动 × 主流框架）

| 达梦版本 | 推荐 JDK | 推荐驱动 | MyBatis-Plus | Hibernate | Sharding-JDBC |
|---------|---------|----------|--------------|-----------|---------------|
| DM 7.x | 1.6 / 1.7 / 1.8 | DmJdbcDriver6/7/8.jar | 3.5.x | 5.2 - 5.6 | 4.x - 5.4 |
| DM 8.0-8.1 | 1.8 / 11 | DmJdbcDriver8/11.jar | 3.5.x | 5.6 - 6.x | 5.x |
| DM 8.2+ | 11 / 17 / 21 | DmJdbcDriver11.jar | 3.5.5+ | 6.x | 5.4+ |
| DM 8.3+ | 17 / 21 | DmJdbcDriver11.jar | 3.5.5+ | 6.x | 5.4+ |

> 跨版本连接：DM8 驱动可连 DM7 数据库（部分场景报"网络通信异常"），反之不行。生产建议**严格对齐版本**。

## 17.1 监控告警（生产必配）

- **Druid 内置监控**：启用 `stat-view-servlet`，访问 `/druid/sql.html` 看慢 SQL
- **HikariCP 指标**：暴露 `/actuator/metrics/hikaricp.connections.active`
- **达梦服务端**：`SELECT * FROM V$LONG_EXEC_SQLS` 看 Top 10 慢 SQL
- **达梦服务端**：`SELECT * FROM V$SESSIONS WHERE STATE='ACTIVE'` 看活跃会话
- **应用层**：`@Around("@annotation(com.xx.annotation.SlowSqlLog)")` + Logback 异步打印
- **告警阈值**：连接池使用率 > 80%、活跃会话 > 70%、慢 SQL 数量 > 10/分钟、达梦 CPU > 70% 持续 5 分钟
- **APM 集成**：SkyWalking / Pinpoint 插件支持达梦 JDBC 8，可视化调用链


## 18.1 单元测试与集成测试

**单元测试（Mock 框架）**：

```java
@SpringBootTest
@AutoConfigureMockMvc
class OrderServiceTest {
    @MockBean
    private ProductCategoryMapper mapper;

    @Autowired
    private OrderService service;

    @Test
    void testCreate() {
        when(mapper.insert(any(ProductCategory.class))).thenReturn(1);
        service.createOrder(...);
        verify(mapper, times(1)).insert(any());
    }
}
```

**集成测试（Testcontainers + 达梦）**：

```java
@Testcontainers
class DMIntegrationTest {
    @Container
    static GenericContainer<?> dm = new GenericContainer<>("dameng/dm8:latest")
            .withExposedPorts(5236);

    @DynamicPropertySource
    static void props(DynamicPropertyRegistry r) {
        r.add("spring.datasource.url", () ->
            "jdbc:dm://" + dm.getHost() + ":" + dm.getFirstMappedPort());
        r.add("spring.datasource.username", () -> "APP_USER");
        r.add("spring.datasource.password", () -> System.getenv("DM_PASSWORD"));
    }
}
```

**达梦官方测试数据集**：示例库 BOOKSHOP（电商场景：用户、商品、订单、库存）+ DMHR（人事场景：员工、部门、薪资）。两库可在 DM 安装目录 `/dmdbms/data/DAMENG/bookshop/` 找到 DMP 文件，通过 `dimp` 导入。


## 19.1 一句话检查清单（信创上线前必看）

- [ ] 驱动版本与 JDK 版本严格对齐
- [ ] URL 加了 `compatibleMode`（MySQL 业务用 `mysql`，Oracle 业务用 `oracle`）
- [ ] URL 加了 `characterEncoding=PG_UTF8`（如系统统一 UTF-8）
- [ ] 所有表名/列名大写或带双引号
- [ ] `@Transactional` 方法无 `readOnly=true` + 写操作混用
- [ ] Druid 配置去掉了 `wall` 过滤器
- [ ] Hibernate 加了 `SetBigStringTryClob=true`
- [ ] 大字段列类型用 `IMAGE`/`BLOB`/`CLOB`，不用 `VARBINARY(8000)`
- [ ] MyBatis-Plus 版本 ≥ 3.5.x
- [ ] 生产环境关闭了 `show_sql` 和 `format_sql`
- [ ] 连接池 `maxLifetime` 小于服务端 `MAX_SESSIONS` × 1/3
- [ ] 应用日志中无 `dm.jdbc.driver.DmDriver is not sequoia compliant`
- [ ] 灰度期间保留 MySQL 双写
- [ ] 慢 SQL 监控、连接池监控、达梦服务端监控三类告警均已配置
