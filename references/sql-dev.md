# SQL 开发技巧与实践（达梦 DM8）

> 本章节围绕达梦 DM8 在日常开发中常用的 SQL写法展开，覆盖从单表查询、聚合、连接、数据操纵，到字符串/数字/日期处理、范围查询、闪回、物化视图、DBLINK、视图、分区表、层次查询，以及读写分离集群下的开发原则。所有示例基于达梦官方示例库 `dmhr`（employee / department / job / city / location 等表）。

## 一、单表查询

最基础也最常用。DM8 支持 `SELECT *`、部分列、`WHERE` 行过滤、`GROUP BY`聚合、`HAVING`过滤分组、`ORDER BY`排序、`LIMIT/OFFSET` 与 `ROWNUM` 双语法限制返回行数。需要注意：`NULL` 不参与算术与比较运算，所有结果为 NULL；可用 `NVL(commission_pct,0)`转换为有意义值。

```sql
--限制返回行数（两种写法等价）
SELECT * FROM dmhr.employee LIMIT10 OFFSET0;
SELECT * FROM (SELECT ROWNUM rn, t.* FROM dmhr.employee t WHERE ROWNUM <=10) WHERE rn BETWEEN1 AND10;

-- 按职位统计薪资
SELECT job_id,
 AVG(salary) 平均值,
 MIN(salary)最小值,
 MAX(salary) 最大值,
 SUM(salary)工资合计,
 COUNT(*) 总行数
 FROM dmhr.employee
 GROUP BY job_id
HAVING AVG(salary) >8000;
```

## 二、查询结果排序

`ORDER BY` 支持列名、列序号、`ASC/DESC`。注意：多列排序只有当前列重复时后续列才生效；`NULL` 在 `ASC` 时排最后、`DESC` 时排最前，可使用 `NULLS FIRST/LAST`显式控制。`TRANSLATE(expr, from, to)` 是按字符做"一对一替换"的利器，常用于清洗数字字母混合串。`LISTAGG ... WITHIN GROUP` 可将排序结果拼成字符串，常用于"同部门同事名字横向展示"场景。

```sql
-- 按手机号尾号升序（子串排序）
SELECT employee_name姓名, SUBSTR(phone_num, -4)尾号
 FROM dmhr.employee
 WHERE ROWNUM <=5
 ORDER BY2;

-- 按指定优先级排序：6000~8000 的排在最前
SELECT job_title职务,
 CASE WHEN min_salary BETWEEN6000 AND8000 THEN1 ELSE2 END级别,
 min_salary工资
 FROM dmhr.job
 WHERE ROWNUM <5
 ORDER BY2,3;
```

## 三、多表联合检索

DM8 支持 `INNER/LEFT/RIGHT/FULL JOIN`，以及 `ON` 与 `USING` 两套连接语法。`USING (col)` 是 `ON a.col = b.col` 的简写（但会自动合并重复列）。`UNION ALL` 仅合并不去重（比 `OR` 执行计划更高效），`UNION` 自动去重，`EXCEPT` 用于差集（类似减法），`INTERSECT` 用于交集。**关键陷阱**：`NOT IN` 子查询若含 `NULL`永远返回空（逻辑 `AND NULL`），应改用 `NOT EXISTS`。连接顺序建议：小表驱动大表（LEFT JOIN 时左表作为驱动表）；笛卡尔积务必先评估行数，必要时用 `WHERE` 加过滤条件。

```sql
-- 内连接（两种写法）
SELECT je.employee_name, d.department_name
 FROM dmhr.join_emp je, dmhr.department d
 WHERE je.department_id = d.department_id;

-- 左外连接保留左表所有记录
SELECT je.employee_name, d.department_name
 FROM dmhr.join_emp je LEFT JOIN dmhr.department d
 ON je.department_id = d.department_id;

--差集：查出 dept 中不存在于 employee 的部门号
SELECT department_id FROM dmhr.dept
EXCEPT
SELECT department_id FROM dmhr.employee;
```

## 四、数据操纵（INSERT / UPDATE / DELETE / MERGE）

DM8 单条 `INSERT` 支持完整字段或按列插入（其余取默认值或 NULL）。批量插入多行使用逗号分隔的 VALUES列表，比单行多次提交快5~10 倍。`MERGE INTO` 是"存在则 update、不存在则 insert"的幂等写法，避免"重复键冲突"。多表 `UPDATE` 必须加 `WHERE EXISTS`过滤，否则未匹配行会被 NULL覆盖（典型坑）。删除重复记录的标准写法：`ROWID NOT IN (SELECT MAX(ROWID) ... GROUP BY重复列)`。`INSERT ALL` 可一次向多表写入，常见于日志分发、ETL 中转场景。`DELETE`关联删除推荐写法：先 `DELETE FROM 子表 t WHERE NOT EXISTS (SELECT1 FROM父表 p WHERE p.id = t.pid)`，再 `DELETE FROM父表` 以保证外键不被破坏。

```sql
-- MERGE INTO：调薪同步
MERGE INTO dmhr.dup_emp t
 USING dmhr.emp_salary s
 ON (t.employee_id = s.employee_id)
WHEN MATCHED THEN
 UPDATE SET t.salary = s.new_salary
WHEN NOT MATCHED THEN
 INSERT VALUES (s.employee_id, 'dm2024', '410107197103257999', s.new_salary,102);

-- 多表 UPDATE 加 WHERE EXISTS（安全写法）
UPDATE dmhr.test ot
 SET (salary, department_id) =
 (SELECT nt.salary, nt.department_id FROM dmhr.test_new nt
 WHERE ot.employee_id = nt.employee_id)
 WHERE EXISTS (SELECT1 FROM dmhr.test_new nt
 WHERE ot.employee_id = nt.employee_id);
```

## 五、字符串处理

DM8字符串函数兼容 Oracle 与 PostgreSQL两种风格，常用：`SUBSTR/INSTR/LENGTH/CONCAT/REPLACE/TRIM/UPPER/LOWER/TRANSLATE`，正则三剑客 `REGEXP_LIKE/REGEXP_REPLACE/REGEXP_SUBSTR` 与 `REGEXP_COUNT`。批量拼接字符串用 `LISTAGG ... WITHIN GROUP`。常用技巧：(1) `SUBSTR(str, -n)` 取末尾 n 个字符；(2) `INSTR(str, sub)`找子串位置（找不到返回0）；(3) `TRANSLATE(str, 'aeiou', '12345')`字符级映射（to 为空则返回空）；(4) `||` 或 `CONCAT()`拼接，注意 `CONCAT` 仅两个参数串联，多个需要嵌套。`q'[ ...]'`替代引号转义，写脚本生成器特别方便。

```sql
--拆分 IP 地址（正则取每段）
SELECT REGEXP_SUBSTR(v.ip,'[^.]+',1,1) a,
 REGEXP_SUBSTR(v.ip,'[^.]+',1,2) b,
 REGEXP_SUBSTR(v.ip,'[^.]+',1,3) c,
 REGEXP_SUBSTR(v.ip,'[^.]+',1,4) d
 FROM (SELECT '192.168.1.111' ip FROM DUAL) v;

-- LISTAGG汇总
SELECT deptno,
 LISTAGG(name, ',') WITHIN GROUP (ORDER BY name) AS total_name
 FROM v
 GROUP BY deptno;
```

## 六、数字处理

`ROUND/CEIL/FLOOR/MOD/TRUNC/ABS` 是基础数值函数；`NULLIF(a, b)` 在 `a = b` 时返回 NULL（用于替换异常值）；`GREATEST/LEAST` 取最大最小（注意 NULL 会污染结果，需要 `COALESCE`兜底）。`SUM(...) OVER (ORDER BY ...)` 是累计和的分析函数写法，比自关联性能高一个数量级。

```sql
--累计成本：员工工资按编号累加
SELECT employee_id编号, employee_name姓名, salary人工成本,
 SUM(salary) OVER (ORDER BY employee_id)成本累计
 FROM dmhr.employee
 WHERE job_id =11;

-- 把异常值（NULLIF 等于某值时置空）转0
SELECT employee_id, salary, COALESCE(NULLIF(salary,0), -1)修正后
 FROM dmhr.employee;
```

## 七、日期运算

DM8 与 Oracle兼容：`ADD_MONTHS`、`MONTHS_BETWEEN`、`LAST_DAY`、`NEXT_DAY`、`EXTRACT`。同时支持 `DATEADD/DATEDIFF/DATE_PART`（更接近 SQL Server风格）。时/分/秒加减用浮点：`1/24 =1 小时`、`1/24/60 =1 分钟`。两个 DATE 相减得到天数（带小数），乘以24 得小时。常用模式：(1) 月末计算 `LAST_DAY(date)`；(2) 当年第几周 `TO_CHAR(date, 'IW')`；(3) 周几（数字）`TO_CHAR(date, 'D')`；(4)季度 `TO_NUMBER(TO_CHAR(date, 'Q'))`。月末、月初、下月初、下月末四个时点是做"月度报表"必算量：`TRUNC(hd, 'mm')`月初、`LAST_DAY(hd)`月末、`ADD_MONTHS(TRUNC(hd,'mm'),1)`下月初、`ADD_MONTHS(TRUNC(hd,'mm'),-1)`上月初。

```sql
-- 入职前后日期推算
SELECT hire_date 入职,
 ADD_MONTHS(hire_date, -6) 半年前,
 ADD_MONTHS(hire_date,6)半年后,
 ADD_MONTHS(hire_date, -60) 五年前
 FROM dmhr.employee WHERE ROWNUM =1;

-- 与当前记录的下一条间隔天数（LEAD 分析函数）
SELECT employee_name, hire_date,
 LEAD(hire_date) OVER (ORDER BY hire_date)下一记录日期,
 LEAD(hire_date) OVER (ORDER BY hire_date) - hire_date间隔天
 FROM dmhr.employee WHERE job_id =11;
```

## 八、日期操作

`TRUNC(date, 'mm' / 'yy' / 'dd' / 'day')` 取月初/年初/一天之始/周初；`EXTRACT` 取年/月/日/时/分/秒（返回 NUMBER）；`TO_CHAR` 可格式化输出；`LAST_DAY` 取月末；`NEXT_DAY(date, n)` 取下一个周 n（1=周日,2=周一）。判断闰年：`LAST_DAY(ADD_MONTHS(TRUNC(date, 'yy'),1))` 是29 号即为闰年。

```sql
-- 月度日历：行转列显示本周一到周日
WITH x1 AS (SELECT TO_DATE('2020-11-01','yyyy-mm-dd') cur FROM DUAL),
 x2 AS (SELECT TRUNC(cur,'mm') 月初, ADD_MONTHS(TRUNC(cur,'mm'),1) 下月初 FROM x1),
 x3 AS (SELECT 月初+(LEVEL-1) 日 FROM x2 CONNECT BY LEVEL <= 下月初-月初),
 x4 AS (SELECT TO_CHAR(日,'iw') 周, TO_CHAR(日,'dd') 日期, TO_NUMBER(TO_CHAR(日,'d')) 周几 FROM x3)
SELECT MAX(CASE 周几 WHEN2 THEN 日期 END) 周一,
 MAX(CASE 周几 WHEN3 THEN 日期 END) 周二,
 MAX(CASE 周几 WHEN4 THEN 日期 END) 周三,
 MAX(CASE 周几 WHEN5 THEN 日期 END) 周四,
 MAX(CASE 周几 WHEN6 THEN 日期 END) 周五,
 MAX(CASE 周几 WHEN7 THEN 日期 END) 周六,
 MAX(CASE 周几 WHEN1 THEN 日期 END) 周日
 FROM x4 GROUP BY 周 ORDER BY 周;
```

## 九、范围处理

DM8 提供4套"范围"工具：`BETWEEN ... AND`（闭区间含两端）、`IN`（离散集合，注意 NULL陷阱）、`EXISTS`/`NOT EXISTS`（半连接，NULL 安全）、`ROW_NUMBER()/RANK/DENSE_RANK OVER`（窗口函数实现 Top-N 与分组去重）。`LEAD/LAG ... OVER` 是相邻行比较的标配，常用于连续区间检测与登录间隔计算。**经验法则**：(1) 主表大、子表小，优先用 `EXISTS`；(2) 子表需要返回列，用 `IN`；(3)找 Top-N 用 `ROW_NUMBER`；(4) 同分组内找最大值/最小值用 `FIRST_VALUE/LAST_VALUE`；(5) 大数据量分页用 `OFFSET n ROWS FETCH NEXT m ROWS ONLY`，避免深翻页性能衰减。

```sql
--找连续值范围：下一行起始日期 == 当前行结束日期
SELECT 工程号, 开始日期,结束日期
 FROM (SELECT pro_id 工程号, pro_start 开始日期, pro_end结束日期,
 LEAD(pro_start) OVER (ORDER BY pro_id)下一工程开始
 FROM v)
 WHERE下一工程开始 =结束日期;

-- 每部门薪资 Top3
SELECT *
 FROM (SELECT department_id, employee_name, salary,
 DENSE_RANK() OVER (PARTITION BY department_id ORDER BY salary DESC) rk
 FROM dmhr.employee)
 WHERE rk <=3;
```

## 十、闪回查询

DM8闪回默认关闭。开启：`ALTER SYSTEM SET 'enable_flashback'=1 BOTH;`，并延长 `undo_retention=1200`（秒）。按时间闪回：`SELECT ... WHEN TIMESTAMP '2024-01-0110:00:00'`；按事务闪回：`VERSIONS BETWEEN TIMESTAMP ... AND SYSDATE`，配合伪列 `VERSIONS_ENDTRXID`、`VERSIONS_STARTTIME`。注意：闪回只支持普通表/水平分区表/堆表，不支持临时表、列存储表。

```sql
-- 查看2024-01-0116:00之前 city 表的数据
SELECT * FROM dmhr.city WHEN TIMESTAMP '2024-01-0116:00:00' WHERE city_id='CD';

-- 查看 job_id=22 在16:00之后的全部历史版本
SELECT VERSIONS_ENDTRXID, *
 FROM dmhr.job VERSIONS BETWEEN TIMESTAMP '2024-01-0116:06:00' AND SYSDATE
 WHERE job_id =22;
```

##十一、物化视图

DM8物化视图是"占存储空间的查询快照"，支持 `COMPLETE/FAST/FORCE` 三种刷新模式，`ON COMMIT / ON DEMAND / START WITH...NEXT / NEVER` 四种刷新时机。基于主键的快速刷新要求基表必须含主键；基于 ROWID快速刷新则必须把 ROWID 取别名列入 SELECT 列。聚合型物化视图要求 `SUM/COUNT(*)` 成对出现。`BUILD IMMEDIATE` 创建时立即填充数据；`BUILD DEFERRED`延迟填充，第一次需要全量刷新。物化视图日志表 `MLOG$_<base_table>`记录基表的所有 DML变化，是 FAST刷新的依据。一张基表只能建一个物化视图日志。FAST刷新限制：不能含 UNION/子查询/HAVING/ANY/ALL/NOT EXISTS/层次查询/分析函数；同一表最多127 个支持 FAST刷新的物化视图。

```sql
-- 创建主键型物化视图 + 自动 ON COMMIT刷新
CREATE MATERIALIZED VIEW mv_emp REFRESH WITH PRIMARY KEY ON COMMIT AS
SELECT * FROM dmhr.employee;

-- 创建物化视图日志后切换到 FAST刷新
CREATE MATERIALIZED VIEW LOG ON dmhr.employee WITH PRIMARY KEY;
ALTER MATERIALIZED VIEW mv_emp REFRESH FAST;

--手动强制刷新
REFRESH MATERIALIZED VIEW mv_emp FORCE;
```

##十二、DBLINK

DM8 DBLINK 支持同构（DM-DM）与异构（DM-Oracle、DM-ODBC）。使用语法：`表名@链接名`。DM-DM 同构链接**不支持 MPP 环境**；异构链接支持 MPP。LOB 类型不支持增删改查，但常量赋值可以。

```sql
-- 创建到远程 DM 的 DBLINK
CREATE PUBLIC LINK link_dm01 CONNECT WITH SYSDBA IDENTIFIED BY ***** USING '192.0.2.100/5236';

--跨库 JOIN
SELECT a.employee_id, a.employee_name, b.department_name
 FROM dmhr.employee a, dmhr.department@link_dm01 b
 WHERE a.department_id = b.department_id;

--跨库数据迁移
INSERT INTO dmhr.employee SELECT * FROM dmhr.employee@link_dm01;
COMMIT;
```

##十三、视图与同义词

普通视图是逻辑表，不占存储；复杂视图（含聚合/分组/连接/UNION）默认只读。带 `WITH CHECK OPTION` 的视图会自动按定义条件校验 UPDATE/INSERT。可更新视图要求基表一对一、不含 DISTINCT/聚合/集合运算。同义词（Synonym）是对象的别名，常用于跨模式访问、版本切换。`WITH READ ONLY`显式标注只读视图。视图基表结构变更后，可用 `ALTER VIEW ... COMPILE`重新编译，DM 会检查视图定义的合法性。公有同义词（`PUBLIC SYNONYM`）所有用户可见，私有同义词仅创建者可用。同义词链（A → B → C）在 DM 中允许，但生产建议避免多层嵌套以减少调用复杂度。`ENABLE_PL_SYNONYM=0`时，禁止通过全局同义词执行非系统用户创建的包。

```sql
-- 可更新视图（带 CHECK OPTION 防越界）
CREATE VIEW purchasing.vendor_excellent AS
SELECT * FROM purchasing.vendor WHERE credit =1 WITH CHECK OPTION;

-- 同义词
CREATE SYNONYM dmhr.emp_syn FOR dmhr.employee;
SELECT * FROM dmhr.emp_syn WHERE department_id =100;
```

##十四、分区表

DM8 支持范围（RANGE）、列表（LIST）、哈希（HASH）、多级组合（最多8 层）、间隔（INTERVAL，自动扩展）5 类分区。**关键限制**：索引组织表上分区，主键必须包含分区键；堆表分区不能跨表空间。间隔分区是范围分区的扩展，按 `INTERVAL (NUMTOYMINTERVAL(1, 'year'))` 自动建新分区。**分区裁剪**（Partition Pruning）是分区表性能的核心：WHERE条件必须包含分区键，否则会全分区扫描。维护操作：`ADD/DROP/TRUNCATE/MERGE/SPLIT PARTITION`。已存在 MAXVALUE 分区时，新增分区需要先 DROP MAXVALUE → ADD 新分区 →重建 MAXVALUE 三步走。全局索引（`GLOBAL`）跨分区但维护成本高，局部分区索引（默认）与表分区一一对应，是 OLTP场景首选。

```sql
-- 按入职年分区（间隔分区自动扩展）
CREATE TABLE dmhr.emp_part (
 employee_id INT PRIMARY KEY,
 hire_date DATE NOT NULL,
 salary INT
)
PARTITION BY RANGE (hire_date)
 INTERVAL (NUMTOYMINTERVAL(1, 'year'))
 (PARTITION p_before_2007 VALUES LESS THAN (DATE '2007-01-01'));

--拆分已合并的分区
ALTER TABLE dmhr.rp_hiredt_emp
 SPLIT PARTITION p3_4 AT ('2009-01-01')
 INTO (PARTITION p3, PARTITION p4);
```

##十五、层次查询

DM8完整兼容 Oracle层次查询语法：`START WITH` 定根节点，`CONNECT BY PRIOR` 定义父子方向（上→下：`PRIOR父键 = 子键`；下→上：`PRIOR 子键 =父键`），`LEVEL`伪列标识层级，`ORDER SIBLINGS BY` 保兄弟节点有序，`SYS_CONNECT_BY_PATH(col, sep)` 输出路径，`CONNECT_BY_ISLEAF` 判断是否叶子，`CONNECT_BY_ISCYCLE` + `NOCYCLE` 检测/规避环。**注意 WHERE是在层次遍历结束后再过滤**，所以不影响 LEVEL编号。常用于组织架构图、BOM物料清单、家族族谱、菜单树、目录树等。`CONNECT_BY_ROOT col` 返回当前节点的根节点列值（适合做"直属上级是 X 的所有下属"）。

```sql
-- 从顶向下：列出100 号员工下所有下属
SELECT employee_id, employee_name, job_title, manager_id, LEVEL
 FROM dmhr.emp
 START WITH employee_id =100
 CONNECT BY PRIOR employee_id = manager_id
 ORDER SIBLINGS BY employee_id;

-- 输出根到当前节点的路径
SELECT employee_id, employee_name, LEVEL,
 SYS_CONNECT_BY_PATH(employee_name, '/')路径
 FROM dmhr.emp
 START WITH employee_id =100
 CONNECT BY NOCYCLE PRIOR employee_id = manager_id;
```

##十六、读写分离集群开发原则

DM读写分离集群（最多8 个备机）通过 JDBC 自动分发：只读事务在备机执行，写事务自动转到主机。事务一致性等价于读提交隔离级。**关键原则**：(1)事务尽量规划为"纯读"+"纯写"，避免无效的备库试错；(2)读操作放在写操作之前；(3)涉及"读后写"的逻辑不要走高性能模式（ARCH_WAIT_APPLY=0），否则读到的可能是备机还没重演的值。锁语句（LOCK TABLE / SELECT FOR UPDATE）、包操作、临时表、动态视图、@@IDENTITY 等全局变量访问**强制走主库**。`rwPercent` 参数控制主备机读事务比例，建议主库10~30%、备机70~90%，结合主备机硬件配置灵活调整。Hint强制主备：`/*+ PRIMARY */`强制走主库，`/*+ STANDBY */`强制走备库。归档故障时主库会切换到 `SUSPEND`状态，对应备库上的影子会话被强制断开，避免数据不一致。

```sql
-- 推荐的事务模式（读在写之前）
SELECT * FROM orders WHERE user_id = :uid; --备机
UPDATE orders SET status = 'PAID' WHERE id = :oid; -- 自动转主
COMMIT;

-- 不推荐的事务模式（依赖同一事务内的读结果）
INSERT INTO tx VALUES (1);
COMMIT;
SELECT TOP1 c1 INTO :v FROM tx WHERE ...; -- 高性能模式下备机可能未重演 →查不到
UPDATE tx SET c1 = :v +1 WHERE c1 = :v; -- 更新不到数据
```

---

##实战：报表查询优化7 连击

针对一张千万级订单表 `orders(order_id, user_id, create_date, amount, status)`，按以下顺序组合技巧可获得最佳性能：

1. **范围裁剪**：`WHERE create_date BETWEEN DATE '2024-01-01' AND DATE '2024-01-31'`（配合分区表裁剪90% 数据）
2. **半连接替代 NOT IN**：`WHERE user_id NOT IN (SELECT user_id FROM vip_blacklist)` →改为 `NOT EXISTS`
3. **先聚合再连接**：`SELECT u.region, SUM(o.amount) FROM ... GROUP BY u.region`，子查询先按 user_id聚合，避免笛卡尔放大
4. **分析函数替代自关联**：累计金额 `SUM(amount) OVER (PARTITION BY user_id ORDER BY create_date)`
5. **物化视图缓存汇总**：日报表建 `ON COMMIT`物化视图，秒级响应
6. **层次查询生成组织树汇总**：组织架构汇总用 `SYS_CONNECT_BY_PATH` + `CONNECT_BY_ROOT`，避免递归 CTE
7. **读写分离路由**：报表类大查询走备机（JDBC 自动），事务内"读后写"强制 `/*+ PRIMARY */` Hint走主库

**逐项收益分析**：(1)分区裁剪可将扫描行数从1000 万降到100 万（按月分区）；(2)NOT EXISTS走 HASH ANTI JOIN，比 NOT IN + 子查询快3~5 倍；(3)子查询先聚合使连接基数从百万降到千级；(4)分析函数单次扫描即可拿到累计值，避免对同一表的自关联重复扫描；(5)物化视图让日报表从分钟级降到毫秒级；(6)层次查询一句话完成组织树汇总，省去应用层多次查询；(7)读写分离让报表与事务互不干扰，主库专注 TP、备库承担 AP。

```sql
-- 综合：1 月各区域每日销售（按 create_date范围裁剪 + 半连接 + 先聚合）
SELECT o.day, u.region,
 SUM(o.amount)销售额,
 RANK() OVER (PARTITION BY o.day ORDER BY SUM(o.amount) DESC) 当日排名
 FROM (SELECT order_id, user_id, amount,
 TRUNC(create_date, 'dd') day
 FROM orders
 WHERE create_date >= DATE '2024-01-01'
 AND create_date < DATE '2024-02-01'
 AND status = 'PAID') o,
 users u
 WHERE o.user_id = u.user_id
 AND NOT EXISTS (SELECT1 FROM vip_blacklist b WHERE b.user_id = o.user_id)
 GROUP BY o.day, u.region
 ORDER BY o.day,销售额 DESC;
```

## 参考资料

- DM8 SQL 语言使用手册（数据库安装路径 `/dmdbms/doc`）
-达梦技术文档：SQL 开发指南 https://eco.dameng.com/document/dm/zh-cn/sql-dev/
- 单表查询 /排序 / 多表 / DML /字符串 /数字 / 日期 /范围 /触发器 /闪回 /物化视图 / DBLINK /视图同义词 / 分区 /层次 /读写分离 —全部位于 `eco.dameng.com/document/dm/zh-cn/sql-dev/` 下
