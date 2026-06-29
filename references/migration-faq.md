# 迁移常见问题 FAQ

## MySQL → DM（51条精选）

### 字符/编码类

**Q: 迁移后数据乱码**
A: DM 没有 MySQL `utf8mb4` 这个字符集名称。迁移时按目标库字符集（UTF-8/GB18030）、JDBC `characterEncoding` 和源库真实样本一起验证；如果业务包含 emoji 等 4 字节字符，先做样本写入/读取校验，不能只按名称自动映射。

**Q: char长度变为原来的3倍**
A: MySQL按字符计，DM按字节计。UTF-8编码下一个汉字=3字节。解决：DTS中设数据类型映射 varchar→varchar(N char)，建表时用 varchar(N char)。

**Q: VARCHAR字段内容截断或结尾乱码**
A: 同上。使用 varchar(N char) 或将字段长度扩大为原来的3倍(UTF-8)或2倍(GBK)。

**Q: unknown initial character 报错**
A: URL添加 `?useUnicode=true&characterEncoding=utf8`。

**Q: 列长度超出定义**
A: MySQL varchar以字符计→DM以字节计。扩大字段长度或使用 char 声明。

### 时间/日期类

**Q: TIMESTAMP DEFAULT '0000-00-00 00:00:00' 迁移报错**
A: DM不允许日期/月份为0。改为 DEFAULT SYSDATE 或改为有效日期如 '1900-01-01'。

**Q: 错误的日期时间类型格式**
A: 同上，'0000-00-00' 不合法。预处理为 '0001-01-01' 或改为varchar存储。

**Q: YEAR字段迁移后数据丢失**
A: DM不支持YEAR类型。MySQL端先 ALTER TABLE MODIFY COLUMN YEAR CHAR(4)，再迁移。

**Q: 时区值无法识别**
A: 自定义URL添加 `serverTimezone=GMT%2B8` 或 `serverTimezone=Asia/Shanghai`。

**Q: 时间日期类型数据溢出**
A: 使用 to_date() 对日期字段做类型转换。

**Q: ON UPDATE CURRENT_TIMESTAMP 不生效**
A: DM不支持此属性，用触发器替代：
```sql
CREATE OR REPLACE TRIGGER update_time
BEFORE UPDATE ON HR.TEST FOR EACH ROW
BEGIN
  :new.modifytime := sysdate;
END;
```

**Q: DTS连接MySQL报时区值无法识别**
A: URL加 `serverTimezone=GMT%2B8`

**Q: 迁移后两端时间数据不一致**
A: URL加 `&serverTimezone=Asia/Shanghai`

### 数据类型/结构类

**Q: 非法IDENTITY列类型**
A: IDENTITY只支持INT和BIGINT。SMALLINT→INT，DECIMAL(20,0)→BIGINT。

**Q: AUTO_INCREMENT报错**
A: MySQL AUTO_INCREMENT→DM使用IDENTITY(初始值,步长)。先查 `show variables like 'auto_inc%'` 获取起始值和步长。

**Q: 不支持该数据类型**
A: 时间列多了字符'T'，选择正确的驱动包和驱动类名。

**Q: USER表迁移后无法操作**
A: USER是DM关键字，需用双引号括起来：`"USER"`。自增列需 `SET IDENTITY_INSERT ON`。

**Q: 此列列表已索引**
A: DM不支持同一列建多个单列索引（MySQL允许）。可忽略此报错，或删除多余索引。

**Q: MySQL允许分母为0，DM报错**
A: 使用case when判断：
```sql
SELECT 2/CASE WHEN 分母=0 THEN NULL ELSE 分母 END;
```
或设 MY_STRICT_TABLES=0 最大容错。

**Q: char类型自动空格补齐**
A: 三种方案：①DTS映射char→varchar2 ②批量ALTER改为VARCHAR ③查询时RTRIM()。

**Q: ID列迁移后数据不一致**
A: 迁移时勾选"启动标志列插入"和"使用auto_increment自增列"。

### 函数改写类

| MySQL | DM等价写法 |
|-------|-----------|
| `FIND_IN_SET(str,list)` | 自定义函数或INSTR组合 |
| `MD5('abc')` | `TO_CHAR(MD5('abc'))`，或设MD5_TYPE=1 |
| `YEARWEEK('2019-07-11',1)` | `year(d)\|\|WEEK(d,1)` |
| `PERIOD_DIFF(202101,202001)` | `months_between(to_date(202101,'yyyymm'),to_date(202001,'yyyymm'))` |
| `(@age:=@age+age)` | `sum(age) OVER (ORDER BY id)` |
| `AES_ENCRYPT/DECRYPT` | DBMS_CRYPTO.ENCRYPT/DECRYPT, AES128/ECB/PKCS5 |
| `COLLATE utf8mb4_bin` | `NLSSORT(col,'NLS_SORT=BINARY')` + 函数索引 |
| SUM求和报错 cast string | `regexp_substr(字段,'\\d+')` 过滤数值 |

### 工具/连接类

**Q: UNSUPPORTED MAJOR.MINOR VERSION 52.0**
A: JDK版本不匹配。修改dts.ini中的JDK路径指向正确版本。

**Q: 界面不完整/无法勾选**
A: 调整显示器分辨率/缩放比例，或下载最新版DTS。

**Q: 刷新选不到数据库**
A: 使用自定义URL：
```
jdbc:mysql://localhost:3306/db?characterEncoding=utf8&useSSL=false&serverTimezone=UTC&rewriteBatchedStatements=true
```

**Q: streaming result set报错**
A: JDBC ResultSet未处理完又提交新query。调整读取行数或使用不同的连接。

**Q: Communications link failure**
A: 检查：驱动匹配、网络连通、端口开放、MySQL最大连接数、SSL设置。

**Q: Access denied for user**
A: MySQL权限问题。在my.ini加 `skip_grant_tables` 后重新授权。

### MyBatis/大小写类

**Q: MyBatis下对象需加双引号**
A: 两种方案：①设 CASE_SENSITIVE=0 ②DTS迁移时取消勾选"保持对象大小写"，对象名自动转大写。

**Q: 表名大小写和MySQL不一致**
A: 勾选DTS中"保持对象名大小写"。

---

## Oracle → DM（60+条精选）

### 核心兼容参数

```sql
COMPATIBLE_MODE=2       -- 部分兼容Oracle（静态参数，需重启）
FLOAT_MODE=1            -- 与COMPATIBLE_MODE=2同时开启
BLANK_PAD_MODE=1        -- 空格填充模式
CASE_SENSITIVE=Y        -- 大小写敏感

-- 修改方式
SP_SET_PARA_VALUE(2,'COMPATIBLE_MODE',2);
ALTER SYSTEM SET 'COMPATIBLE_MODE'=2 SPFILE;
-- 重启数据库生效
```

### 常见问题

**Q: varchar2(4000)迁移后变3900**
A: DM页大小限制。8KB页下最大3900。换16K/32K页可支持更大。

**Q: Oracle date类型**
A: 统一转换为TIMESTAMP。

**Q: 违反引用约束/唯一性约束**
A: 分三步迁移：先表定义→再数据→最后约束和索引。或选择"删除后再拷贝"策略。

**Q: 视图报"无效用户对象"**
A: 迁移顺序问题。先迁表→再迁视图/存储过程。

**Q: 精度必须大于标度**
A: Oracle允许number(m,n)中n>m，DM不允许。调整精度或改为其他类型。

**Q: Oracle raw→DM**
A: 使用VARBINARY替代。

**Q: Java Heap Space**
A: 修改dts.ini加 `-Xmx2048m`。

**Q: ORA-00942表不存在**
A: 需授予 `select any dictionary` 权限给迁移用户。

**Q: 序列最大值超出**
A: DM升序序列最大 9223372036854775807。

**Q: 精度超出范围**
A: 字符集差异(GBK/UTF-8)，调整数据类型映射中的字符长度。

**Q: blob/clob排序报错**
A: 设置 `ENABLE_BLOB_CMP_FLAG=1`。

**Q: COMPATIBLE_MODE=2后IS NULL失效**
A: 参数需在创建表之前设置。已存在的旧数据需update为null。

**Q: 数据缺失**
A: 指定与Oracle版本适配的JDBC驱动。使用Oracle安装目录下的ojdbc.jar。

**Q: 数据溢出**
A: Oracle int→number(38)，超过38位改为varchar存储。

**Q: 迁移卡住不动**
A: 大字段表每次读取行数改为1，关掉快速装载。

**Q: 唯一约束报错但数据无重复**
A: 末尾空格问题。初始化时加 `BLANK_PAD_MODE=1`。

**Q: forall报错**
A: DM的execute immediate不支持动态语句中的forall。改为静态SQL。

**Q: 更新只读视图**
A: 拆分复杂视图的修改语句为多个基表语句。

**Q: Oracle 12c+不识别SID**
A: 使用自定义URL：
```
jdbc:oracle:thin:@(DESCRIPTION=(ADDRESS_LIST=(ADDRESS=(PROTOCOL=TCP)(HOST=ip)(PORT=1521)))(CONNECT_DATA=(SERVICE_NAME=orcl)))
```

**Q: 列长度修改不生效**
A: 需在DTS数据类型映射中配置自定义映射关系。

**Q: Connection reset**
A: 注释/etc/resolv.conf中search openstacklocal，或添加 `-Djava.net.preferIPv4Stack=true`。

**Q: 记录超长**
A: `ALTER TABLE xxx ENABLE USING LONG ROW;`

**Q: 存储过程"对象定义被修改"**
A: DDL后DML用execute immediate动态执行。

**Q: ORA_LOGIN_USER无效**
A: 改用 `USER()` 函数。

**Q: 触发器跨模式报错**
A: 先迁移被调用的对象，手动恢复模式名。

**Q: ipv6连接问题**
A: dts.ini中 `-Djava.net.preferIPv4Stack` 改为false。

**Q: BM$/CTI$是什么表**
A: BM$=位图索引辅助表，CTI$=全文索引辅助表。

**Q: Oracle dblink→DM blob报错**
A: 用游标循环替代 insert select。

**Q: DTS命令行迁移**
A: 图形化配置→导出xml→`dts_cmd_run.sh` 执行。

**Q: 大数据量调研SQL**
```sql
SELECT a.username 用户,
  (SELECT count(1) FROM dba_tables b WHERE b.owner=a.username) 表数量,
  (SELECT count(1) FROM dba_indexes i WHERE UNIQUENESS='UNIQUE' AND owner=a.username) 唯一索引,
  (SELECT count(DISTINCT c.table_name) FROM dba_tab_partitions c WHERE c.table_owner=a.username) 分区表,
  (SELECT count(1) FROM dba_tab_cols d WHERE d.OWNER=a.username AND d.DATA_TYPE LIKE '%LOB%') 含LOB表,
  (SELECT sum(e.bytes)/1024/1024/1024 FROM dba_extents e WHERE EXISTS(SELECT 1 FROM dba_lobs f WHERE f.owner=a.username AND f.segment_name=e.segment_name)) LOB占用GB,
  (SELECT count(1) FROM dba_views g WHERE g.OWNER=a.username) 视图,
  (SELECT count(1) FROM dba_triggers h WHERE h.owner=a.username) 触发器,
  (SELECT count(DISTINCT i.NAME) FROM dba_source i WHERE i.OWNER=a.username AND i.TYPE='FUNCTION') 函数,
  (SELECT count(1) FROM dba_sequences j WHERE j.sequence_owner=a.username) 序列,
  (SELECT count(1) FROM dba_mviews k WHERE k.owner=a.username) 物化视图,
  (SELECT count(DISTINCT l.NAME) FROM dba_source l WHERE l.OWNER=a.username AND l.TYPE='PROCEDURE') 存储过程,
  (SELECT count(1) FROM dba_db_links m WHERE m.owner=a.username) DBLINK,
  (SELECT max(n.DATA_LENGTH) FROM dba_tab_cols n WHERE n.OWNER=a.username) 最大字段宽,
  (SELECT sum(o.DATA_LENGTH) FROM dba_tab_cols o WHERE o.OWNER=a.username AND o.DATA_TYPE NOT LIKE '%LOB%') 最大行宽
FROM dba_users a WHERE username IN ('用户');
```

---

## SQL Server → DM（15条精选）

### 兼容参数
```sql
COMPATIBLE_MODE=3       -- 兼容SQL Server
MS_PARSE_PERMIT=2       -- 支持MSSQL语法+@变量赋值

SP_SET_PARA_VALUE(2,'COMPATIBLE_MODE',3);
ALTER SYSTEM SET 'MS_PARSE_PERMIT'=2 SPFILE;
-- 重启生效
```

### 常见问题

**Q: 自增列已有数据如何保持不变**
A: 重命名原表→重建含IDENTITY(1,1)的新表→SET IDENTITY_INSERT ON→插入数据→SET IDENTITY_INSERT OFF。

**Q: varbinary(max)数据超范围**
A: 改为BLOB类型。

**Q: DATETIME迁移报无效日期**
A: DTS将DATETIME→TIMESTAMP，但可能转成了TIMESTAMP WITH TIME ZONE，需目标库字段类型匹配。

**Q: SQL Server 2019连接失败**
A: 开放防火墙端口1433 + 启用TCP/IP协议（SQL Server配置管理器）。

**Q: TLS10协议拒绝**
A: 修改java.security中jdk.tls.disabledAlgorithms，删除TLSv1/TLSv1.1。

**Q: convert函数改写**
A: SQL Server `convert(type,expr,style)`→DM `to_char(convert(type,expr),'yyyy-mm-dd hh24:MI:ss')`。

**Q: with(nolock)改写**
A: 改为 `WITH UR`（读未提交），在SELECT末尾加一次即可。

**Q: FOR XML PATH字符串拼接**
A: 两种方案：
```sql
-- 方案1: listagg → 返回varchar
SELECT listagg(hobby,',') WITHIN GROUP(ORDER BY hobby)||',' FROM t;

-- 方案2: xmlagg → 返回text
SELECT xmlagg(xmlparse(CONTENT hobby||',' WELLFORMED) ORDER BY hobby).getclobval() FROM t;
```

**Q: @@ROWCOUNT改写**
A: 在PL块中使用 `SQL%ROWCOUNT`。

**Q: 触发器改造**
A: 关键变化：
- `inserted`→`new.字段`, `deleted`→`old.字段`
- `select @status=status from inserted`→`status:=new.status`
- 变量不加@，赋值用`:=`
- 创建格式：`CREATE OR REPLACE TRIGGER ... AFTER INSERT ON ... FOR EACH ROW`

**Q: NEXT VALUE FOR**
A: 设置 `sp_set_session_parse_type('TSQL')` 兼容TSQL解析器。

**Q: 建表失败导致执行挂起**
A: dmhs.hs中添加 `<ddl_continue>1</ddl_continue>`。

---

## 通用移籍问题

**Q: DTS工具选择驱动**
A: 建议通过指定驱动方式连接。在DTS连接界面点击"指定驱动"，选择与源库版本匹配的JDBC驱动包。Oracle驱动可在Oracle官网获取对应版本。

**Q: Java Heap Space内存溢出**
A: 修改dts.ini（位于DTS安装目录），增加JVM内存：`-Xmx2048m`。

**Q: 迁移后数据量不一致**
A: 分别在源端和目的端创建辅助统计表，对比表数量、数据量、对象数量。详见migration.md校验章节。

**Q: 迁移后性能差**
A: 必须执行全库统计信息更新：
```sql
DBMS_STATS.GATHER_SCHEMA_STATS('模式名', 100, FALSE, 'FOR ALL COLUMNS SIZE AUTO');
```
然后开启SQL日志排查慢SQL。

**Q: 迁移顺序建议**
A: 序列→表结构→数据→视图→自定义类型→函数→存储过程→包→触发器→约束→索引。大表单独迁移。大字段多的表调小批量行数。
