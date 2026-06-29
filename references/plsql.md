# DM8 PL/SQL 程序设计（DMSQL）

> 本章聚焦达梦数据库的过程化 SQL（DMSQL 程序），与 Oracle PL/SQL 语法高度兼容，常见业务可直接迁移。

## 一、DMSQL 程序设计概述

**DMSQL 程序**是达梦数据库对标准 SQL 的过程化扩展，类似 Oracle 的 PL/SQL。其语法与 PL/SQL 高度兼容，**约 90% 业务代码可直接迁移**（包括存储过程、函数、包、触发器、动态 SQL），剩余差异集中在自治事务、对象类型等高级特性。

### 块结构

```sql
DECLARE
  -- 声明区：变量、常量、类型、游标、异常
  v_salary NUMBER(10,2) := 0;
BEGIN
  -- 执行区：SQL 语句 + 控制结构
  SELECT SALARY INTO v_salary FROM EMPLOYEE WHERE EMPLOYEE_ID = 1001;
EXCEPTION
  WHEN NO_DATA_FOUND THEN
    DBMS_OUTPUT.PUT_LINE('员工不存在');
END;
/
```

四段式结构 `DECLARE / BEGIN / EXCEPTION / END` 是 DMSQL 最小执行单元；其中 DECLARE 与 EXCEPTION 为可选项，匿名块、命名子程序（PROCEDURE / FUNCTION）、触发器内部都遵循此结构。

### DMSQL 程序的分类

| 分类 | 存储位置 | 生命周期 | 适用场景 |
|------|----------|----------|----------|
| 存储过程（PROCEDURE） | 数据字典 | 持久 | 批量业务处理 |
| 存储函数（FUNCTION） | 数据字典 | 持久 | 计算 + 返回值（SQL 中可调用） |
| 包（PACKAGE） | 数据字典 | 持久 | 模块化封装 |
| 触发器（TRIGGER） | 数据字典 | 持久 | 事件驱动 |
| 客户端 DMSQL | 临时虚过程 | 会话级 | 脚本批处理 |

### 使用 DMSQL 的五大优点

1. **与 SQL 完美结合**——DMSQL 中可直接写 SELECT/INSERT/UPDATE/DELETE，SQL 中也可调用 DMSQL 函数。
2. **高生产率**——业务逻辑集中存储，避免网络往返。
3. **更高性能**——预编译为伪码（p-code），不需每次解析。
4. **便于维护**——一处修改全部生效，可细粒度授权。
5. **更高安全性**——只授予 EXECUTE 权限即可使用，无需开放表权限。

## 二、DMSQL 程序专用数据类型

除 SQL 常规类型（VARCHAR、NUMBER、DATE、TIMESTAMP 等）外，DMSQL 还提供以下**过程化专用类型**。

### 2.1 RECORD（记录类型）

复合类型，类似数据库中的一行。三种声明方式：

```sql
-- 1) 基于表的记录（%ROWTYPE）
DECLARE
  v_emp EMPLOYEE%ROWTYPE;
BEGIN
  SELECT * INTO v_emp FROM EMPLOYEE WHERE EMPLOYEE_ID = 1001;
  DBMS_OUTPUT.PUT_LINE(v_emp.EMPLOYEE_NAME);
END;
/

-- 2) 程序员自定义记录
DECLARE
  TYPE t_salary_rec IS RECORD (
    emp_id   EMPLOYEE.EMPLOYEE_ID%TYPE,
    emp_name EMPLOYEE.EMPLOYEE_NAME%TYPE,
    sal      NUMBER(10,2)
  );
  v_rec t_salary_rec;
BEGIN
  SELECT EMPLOYEE_ID, EMPLOYEE_NAME, SALARY
    INTO v_rec FROM EMPLOYEE WHERE EMPLOYEE_ID = 1001;
END;
/

-- 3) 游标记录
DECLARE
  CURSOR c IS SELECT EMPLOYEE_ID, SALARY FROM EMPLOYEE;
  v_row c%ROWTYPE;
BEGIN
  OPEN c; FETCH c INTO v_row; CLOSE c;
END;
/
```

### 2.2 %TYPE 与 %ROWTYPE 锚定

**强烈推荐**使用锚定，源表字段类型变更时自动跟随，避免硬编码：

```sql
DECLARE
  v_name  EMPLOYEE.EMPLOYEE_NAME%TYPE;  -- 跟随源列类型
  v_emp   EMPLOYEE%ROWTYPE;             -- 跟随整张表
BEGIN
  NULL;
END;
/
```

### 2.3 集合（TABLE）

DMSQL 提供三种集合，对应 Oracle 三种类型：

| 类型 | 索引 | 边界 | 持久化 | 使用场景 |
|------|------|------|--------|----------|
| 关联数组 (INDEX BY) | INT 或 VARCHAR | 无界 | 仅 PL/SQL | 内存临时表 |
| 嵌套表 (NESTED TABLE) | INT（从 1 开始） | 无界 | 可存表 | BULK COLLECT |
| 可变数组 (VARRAY) | INT（从 1 开始） | 有界 | 可存表 | 数量已知 |

```sql
-- 关联数组（字符串索引）
DECLARE
  TYPE t_salary IS TABLE OF NUMBER INDEX BY VARCHAR(30);
  v_sal t_salary;
BEGIN
  v_sal('张三') := 8000;
  v_sal('李四') := 9500;
  DBMS_OUTPUT.PUT_LINE('张三 工资 = ' || v_sal('张三'));
END;
/
```

### 2.4 REF CURSOR / SYS_REFCURSOR

用于在程序间传递结果集：

```sql
DECLARE
  TYPE t_emp_cur IS REF CURSOR;       -- 弱类型
  c_emp SYS_REFCURSOR;                -- 强类型（DM 预定义）
BEGIN
  OPEN c_emp FOR SELECT EMPLOYEE_ID, EMPLOYEE_NAME FROM EMPLOYEE;
  -- 返回给调用者
END;
/
```

### 2.5 伪记录 :OLD / :NEW

仅在行级触发器中可用，分别表示变更前/后的行：

```sql
CREATE OR REPLACE TRIGGER tr_emp_audit
AFTER UPDATE OF SALARY ON EMPLOYEE
FOR EACH ROW
BEGIN
  IF :OLD.SALARY != :NEW.SALARY THEN
    INSERT INTO SALARY_LOG(EMP_ID, OLD_SAL, NEW_SAL, CHANGE_TIME)
    VALUES(:OLD.EMPLOYEE_ID, :OLD.SALARY, :NEW.SALARY, SYSDATE);
  END IF;
END;
/
```

## 三、控制结构

DMSQL 提供与 PL/SQL 一致的完整控制流。

### 3.1 条件分支

```sql
-- IF ... ELSIF ... ELSE
IF v_score >= 90 THEN
  v_grade := 'A';
ELSIF v_score >= 80 THEN
  v_grade := 'B';
ELSE
  v_grade := 'C';
END IF;

-- 搜索式 CASE
CASE
  WHEN v_grade = 'A' THEN v_desc := '优秀';
  WHEN v_grade = 'B' THEN v_desc := '良好';
  ELSE v_desc := '合格';
END CASE;

-- 简单 CASE（DM 扩展）
CASE v_status
  WHEN 0 THEN v_label := '待支付';
  WHEN 1 THEN v_label := '已支付';
  WHEN 2 THEN v_label := '已发货';
  ELSE         v_label := '未知';
END CASE;
```

### 3.2 循环

```sql
-- 基本 LOOP（需 EXIT）
LOOP
  v_cnt := v_cnt + 1;
  EXIT WHEN v_cnt >= 10;
END LOOP;

-- WHILE
WHILE v_cnt < 10 LOOP
  v_cnt := v_cnt + 1;
END LOOP;

-- 数字 FOR
FOR i IN 1..10 LOOP
  INSERT INTO TEST_TAB VALUES(i);
END LOOP;

-- 反向 FOR
FOR i IN REVERSE 10..1 LOOP
  DBMS_OUTPUT.PUT_LINE(i);
END LOOP;

-- 游标 FOR（隐式 OPEN/FETCH/CLOSE）
FOR rec IN (SELECT EMPLOYEE_ID, SALARY FROM EMPLOYEE WHERE DEPT_ID = 10) LOOP
  DBMS_OUTPUT.PUT_LINE(rec.EMPLOYEE_ID || ':' || rec.SALARY);
END LOOP;
```

### 3.3 EXIT / CONTINUE / GOTO

```sql
FOR i IN 1..100 LOOP
  CONTINUE WHEN MOD(i, 2) = 0;   -- 跳过偶数
  EXIT WHEN i > 50;               -- 跳出循环
  DBMS_OUTPUT.PUT_LINE(i);
END LOOP;

-- GOTO（谨慎使用）
BEGIN
  GOTO end_block;
  <<middle>>
  NULL;
  GOTO end_block;
  <<end_block>>
  DBMS_OUTPUT.PUT_LINE('done');
END;
/
```

### 3.4 嵌套块与作用域

变量可见范围：内层块可访问外层块变量，外层块不可访问内层块变量：

```sql
DECLARE
  v_outer VARCHAR(20) := 'outer';
BEGIN
  DECLARE
    v_inner VARCHAR(20) := 'inner';
  BEGIN
    DBMS_OUTPUT.PUT_LINE(v_outer);  -- OK
  END;
  -- DBMS_OUTPUT.PUT_LINE(v_inner);  -- 编译错误
END;
/
```

## 四、存储过程

### 4.1 创建与调用

```sql
-- 简单无参过程
CREATE OR REPLACE PROCEDURE proc_init_test
AS
BEGIN
  DELETE FROM TEST_TAB;
  DBMS_OUTPUT.PUT_LINE('清空完成');
END;
/

-- 调用方式
CALL proc_init_test();        -- 标准 SQL 调用
EXEC proc_init_test;          -- DIsql 简写
proc_init_test;               -- 块中调用
```

### 4.2 三种参数模式

| 模式 | 用途 | 实参要求 |
|------|------|----------|
| `IN` | 输入（默认） | 常量、变量、表达式 |
| `OUT` | 输出（清空传入值） | 必须为变量 |
| `INOUT` | 输入并输出 | 必须为变量 |

```sql
CREATE OR REPLACE PROCEDURE proc_calc(
  p_id      IN  NUMBER,           -- 输入
  p_name    OUT VARCHAR(50),      -- 输出
  p_salary  IN OUT NUMBER         -- 输入输出
)
AS
BEGIN
  SELECT EMPLOYEE_NAME, SALARY
    INTO p_name, p_salary
    FROM EMPLOYEE
   WHERE EMPLOYEE_ID = p_id;
  p_salary := p_salary * 12;      -- 年薪
EXCEPTION
  WHEN NO_DATA_FOUND THEN
    p_name := 'NOT_FOUND';
    p_salary := 0;
END;
/

-- 调用
DECLARE
  v_name VARCHAR2(50);
  v_sal  NUMBER := 8000;
BEGIN
  proc_calc(1001, v_name, v_sal);
  DBMS_OUTPUT.PUT_LINE(v_name || ' 年薪=' || v_sal);
END;
/
```

### 4.3 NOCOPY 与默认值

```sql
-- NOCOPY：OUT/INOUT 参数按引用传递（大对象性能优化）
CREATE OR REPLACE PROCEDURE proc_big(
  p_arr IN OUT NOCOPY CLOB
) AS
BEGIN
  NULL;
END;
/

-- 默认值（DM 支持 IN 参数默认值，调用可省略）
CREATE OR REPLACE PROCEDURE proc_log(
  p_msg VARCHAR(200) := 'ok',
  p_level INT := 0
) AS
BEGIN
  INSERT INTO LOG_TAB(MSG, LEVEL) VALUES(p_msg, p_level);
END;
/
CALL proc_log();                  -- 全部用默认
CALL proc_log('error', 1);
```

## 五、函数

### 5.1 标量函数

```sql
CREATE OR REPLACE FUNCTION get_annual_salary(p_emp_id NUMBER)
RETURN NUMBER
AS
  v_sal NUMBER;
BEGIN
  SELECT SALARY * 12 INTO v_sal FROM EMPLOYEE WHERE EMPLOYEE_ID = p_emp_id;
  RETURN v_sal;
EXCEPTION
  WHEN NO_DATA_FOUND THEN
    RETURN 0;
END;
/

-- SQL 中可直接调用
SELECT get_annual_salary(1001) FROM DUAL;
```

### 5.2 DETERMINISTIC（确定性声明）

**同输入必返回同结果**的函数可声明 DETERMINISTIC，启用函数缓存：

```sql
CREATE OR REPLACE FUNCTION fn_md5(p_str VARCHAR)
RETURN VARCHAR
DETERMINISTIC
AS
BEGIN
  -- 自定义 MD5 逻辑
  RETURN DBMS_OBFUSCATION_TOOLKIT.MD5(input_string => p_str);
END;
/
```

⚠️ 不要在 DETERMINISTIC 函数中读取外部变量或表（会引发"ORA-14551: cannot perform a DML operation inside a query"等错误）。

### 5.3 PIPELINED（管道函数）

逐行返回结果，类似"流式"输出，可在 SQL 中用 `TABLE()` 解引用：

```sql
-- 1) 定义对象类型
CREATE OR REPLACE TYPE t_num_row AS OBJECT (n NUMBER);
/

-- 2) 定义嵌套表类型
CREATE OR REPLACE TYPE t_num_tab IS TABLE OF t_num_row;
/

-- 3) 管道函数
CREATE OR REPLACE FUNCTION gen_numbers(p_max NUMBER)
RETURN t_num_tab PIPELINED
AS
BEGIN
  FOR i IN 1..p_max LOOP
    PIPE ROW(t_num_row(i));
  END LOOP;
  RETURN;
END;
/

-- 4) SQL 中使用
SELECT * FROM TABLE(gen_numbers(5));
-- 输出 1,2,3,4,5
```

### 5.4 自定义函数 vs 内置函数

DM 提供 200+ 内置函数（数值、字符、日期、转换、聚合、分析），**优先使用内置函数**：性能经过优化、已规避常见 bug。例如优先用 `INSTR/SUBSTR/REPLACE` 而非自写字符串解析逻辑。

## 六、触发器

### 6.1 DML 触发器（BEFORE / AFTER / INSTEAD OF）

```sql
-- BEFORE INSERT 行级：用序列自增
CREATE OR REPLACE TRIGGER trg_order_bi
BEFORE INSERT ON ORDERS
FOR EACH ROW
BEGIN
  :NEW.ORDER_ID := SEQ_ORDER.NEXTVAL;
  :NEW.CREATE_TIME := SYSDATE;
END;
/

-- AFTER UPDATE OF 列名 行级：审计
CREATE OR REPLACE TRIGGER trg_order_bu
AFTER UPDATE OF STATUS ON ORDERS
FOR EACH ROW
WHEN (OLD.STATUS != NEW.STATUS)        -- WHEN 内不加冒号
BEGIN
  INSERT INTO ORDER_AUDIT(ORDER_ID, FROM_STATUS, TO_STATUS, OP_TIME)
  VALUES(:OLD.ORDER_ID, :OLD.STATUS, :NEW.STATUS, SYSDATE);
END;
/

-- INSTEAD OF（仅视图）：用触发器替换原始 DML
CREATE OR REPLACE TRIGGER trg_v_order_iu
INSTEAD OF UPDATE ON V_ORDER_DETAIL
FOR EACH ROW
BEGIN
  UPDATE ORDER_DETAIL SET QTY = :NEW.QTY WHERE ORDER_ID = :OLD.ORDER_ID;
END;
/
```

**行级 vs 语句级**：`FOR EACH ROW` 省略则为语句级（每条 SQL 触发一次）。⚠️ 水平分区子表、HUGE 表**不支持表级触发器**。

### 6.2 时间触发器

最低精度 1 分钟，可替代 cron 调度：

```sql
CREATE OR REPLACE TRIGGER trg_daily_job
AFTER TIMER ON DATABASE
FOR EACH 1 DAY FOR EACH 1 HOUR
BEGIN
  -- 每天整点执行
  DBMS_STATS.GATHER_SCHEMA_STATS('DMHR');
END;
/
```

### 6.3 DDL 触发器与系统触发器

```sql
-- 阻止对核心表的 DROP
CREATE OR REPLACE TRIGGER trg_prevent_drop
BEFORE DROP ON SCHEMA
BEGIN
  IF DICT_OBJ_NAME IN ('ORDERS', 'EMPLOYEE') THEN
    RAISE_APPLICATION_ERROR(-20001, '禁止删除核心表');
  END IF;
END;
/

-- 用户登录审计
CREATE OR REPLACE TRIGGER trg_logon_audit
AFTER LOGON ON DATABASE
BEGIN
  INSERT INTO LOGON_LOG(USERNAME, LOGON_TIME, IP)
  VALUES(USER, SYSDATE, SYS_CONTEXT('USERENV', 'IP_ADDRESS'));
END;
/
```

### 6.4 触发器管理

```sql
ALTER TRIGGER trg_order_bi DISABLE;   -- 禁用
ALTER TRIGGER trg_order_bi ENABLE;    -- 启用
DROP TRIGGER trg_order_bi;            -- 删除
SELECT * FROM USER_TRIGGERS;          -- 查看
```

⚠️ **DM 数据守护备库上的触发器不会被触发**，避免双写误判。

## 七、异常处理

### 7.1 预定义异常

DM 预定义常用异常，**可在 WHEN 子句中直接使用**：

| 异常名 | 错误号 | 描述 |
|--------|--------|------|
| `NO_DATA_FOUND` | -7065 | SELECT INTO 无数据 |
| `TOO_MANY_ROWS` | -7046 | SELECT INTO 多行 |
| `DUP_VAL_ON_INDEX` | -6602 | 唯一性冲突 |
| `ZERO_DIVIDE` | -6103 | 除零 |
| `INVALID_CURSOR` | -4535 | 非法游标操作 |
| `VALUE_ERROR` | -6149 | 类型转换错误 |
| `OTHERS` | — | 通配所有未列出的异常（必须放最后） |

```sql
DECLARE
  v_sal NUMBER;
BEGIN
  SELECT SALARY INTO v_sal FROM EMPLOYEE WHERE EMPLOYEE_ID = 9999;
EXCEPTION
  WHEN NO_DATA_FOUND THEN
    DBMS_OUTPUT.PUT_LINE('员工不存在');
  WHEN TOO_MANY_ROWS THEN
    DBMS_OUTPUT.PUT_LINE('匹配多行');
  WHEN OTHERS THEN
    DBMS_OUTPUT.PUT_LINE(SQLCODE || ':' || SQLERRM);
END;
/
```

### 7.2 PRAGMA EXCEPTION_INIT（绑定错误码）

将 DM 内部错误码与自定义异常名绑定：

```sql
DECLARE
  e_too_many EXCEPTION;
  PRAGMA EXCEPTION_INIT(e_too_many, -7046);
BEGIN
  SELECT SALARY INTO v_sal FROM EMPLOYEE;  -- 多行
EXCEPTION
  WHEN e_too_many THEN
    DBMS_OUTPUT.PUT_LINE('捕获到 TOO_MANY_ROWS');
END;
/
```

### 7.3 EXCEPTION FOR（DM 扩展语法）

更简洁的"声明 + 绑定"一步完成：

```sql
DECLARE
  e_no_emp EXCEPTION FOR -20001, '员工不存在';
BEGIN
  IF NOT EXISTS(SELECT 1 FROM EMPLOYEE WHERE ID = 1) THEN
    RAISE e_no_emp;
  END IF;
EXCEPTION
  WHEN e_no_emp THEN
    DBMS_OUTPUT.PUT_LINE(SQLERRM);
END;
/
```

错误号 `[-30000, -15000]` 区间系统自动分配；自定义码必须是负数。

### 7.4 RAISE_APPLICATION_ERROR

在存储子程序中主动抛出用户异常并向调用方返回错误码/信息：

```sql
CREATE OR REPLACE PROCEDURE check_age(p_age NUMBER)
AS
BEGIN
  IF p_age < 18 OR p_age > 65 THEN
    RAISE_APPLICATION_ERROR(-20001, '年龄必须在 18-65 之间');
  END IF;
END;
/

-- 错误码范围 -30000 ~ -20000，信息 ≤ 2000 字节
```

### 7.5 SQLCODE / SQLERRM

```sql
EXCEPTION
  WHEN OTHERS THEN
    DBMS_OUTPUT.PUT_LINE('CODE=' || SQLCODE);
    DBMS_OUTPUT.PUT_LINE('MSG='  || SQLERRM);                    -- 描述
    DBMS_OUTPUT.PUT_LINE('SPEC=' || SQLERRM(SQLCODE));           -- 带错误号
    DBMS_OUTPUT.PUT_LINE('TRACE=' || DBMS_UTILITY.FORMAT_ERROR_BACKTRACE);
    RAISE;                                                       -- 重新抛出
```

### 7.6 异常传播与作用域

- 块 A 内捕获 → 不再外传
- 块 B 未捕获 → 沿调用栈向外层传播
- 全部未捕获 → 传给主机环境（程序异常终止）
- ⚠️ **声明区异常无法被本块 EXCEPTION 捕获**（编译时错误）：

```sql
-- 错误示例：值超长在声明区就报错
DECLARE
  v_x CONSTANT NUMBER(3) := 5000;
BEGIN
  NULL;
EXCEPTION
  WHEN VALUE_ERROR THEN NULL;   -- 捕获不到
END;
/

-- 正确做法：把声明放入内层块
BEGIN
  DECLARE
    v_x CONSTANT NUMBER(3) := 5000;
  BEGIN
    NULL;
  END;
EXCEPTION
  WHEN VALUE_ERROR THEN DBMS_OUTPUT.PUT_LINE('ok');
END;
/
```

### 7.7 FORALL SAVE EXCEPTIONS

批量 DML 跳过单行错误继续执行：

```sql
DECLARE
  TYPE t_id IS TABLE OF NUMBER;
  v_ids t_id := t_id(1, 2, 9999999, 4);
BEGIN
  FORALL i IN 1..v_ids.COUNT SAVE EXCEPTIONS
    UPDATE EMPLOYEE SET SALARY = SALARY * 1.1 WHERE EMPLOYEE_ID = v_ids(i);
EXCEPTION
  WHEN OTHERS THEN
    FOR i IN 1..SQL%BULK_EXCEPTIONS.COUNT LOOP
      DBMS_OUTPUT.PUT_LINE('错误行 ' || SQL%BULK_EXCEPTIONS(i).ERROR_INDEX ||
                           ' 错误码 ' || SQL%BULK_EXCEPTIONS(i).ERROR_CODE);
    END LOOP;
    COMMIT;
END;
/
```

## 八、游标

### 8.1 隐式游标（SQL 游标）

DML 语句自动使用隐式游标，通过 `SQL%` 属性访问：

```sql
BEGIN
  UPDATE EMPLOYEE SET SALARY = SALARY * 1.1 WHERE DEPT_ID = 10;
  IF SQL%FOUND THEN
    DBMS_OUTPUT.PUT_LINE('更新了 ' || SQL%ROWCOUNT || ' 行');
  END IF;
  IF SQL%NOTFOUND THEN
    DBMS_OUTPUT.PUT_LINE('无数据');
  END IF;
END;
/
```

| 属性 | 含义 |
|------|------|
| `SQL%FOUND` | 影响 1+ 行 → TRUE |
| `SQL%NOTFOUND` | 影响 0 行 → TRUE |
| `SQL%ROWCOUNT` | 影响行数 |
| `SQL%ISOPEN` | 隐式游标恒为 FALSE（执行后即关闭） |

### 8.2 显式游标

程序员手动 `OPEN / FETCH / CLOSE`：

```sql
DECLARE
  CURSOR c_emp IS SELECT EMPLOYEE_ID, EMPLOYEE_NAME, SALARY
                    FROM EMPLOYEE WHERE DEPT_ID = 10;
  v_row c_emp%ROWTYPE;
BEGIN
  OPEN c_emp;
  LOOP
    FETCH c_emp INTO v_row;
    EXIT WHEN c_emp%NOTFOUND;
    DBMS_OUTPUT.PUT_LINE(v_row.EMPLOYEE_NAME);
  END LOOP;
  CLOSE c_emp;
END;
/

-- 简化：游标 FOR 循环（自动 OPEN/FETCH/CLOSE）
FOR rec IN c_emp LOOP
  DBMS_OUTPUT.PUT_LINE(rec.EMPLOYEE_NAME);
END LOOP;

-- 带参数的游标
DECLARE
  CURSOR c_emp(p_dept NUMBER) IS
    SELECT * FROM EMPLOYEE WHERE DEPT_ID = p_dept;
BEGIN
  FOR r IN c_emp(20) LOOP
    DBMS_OUTPUT.PUT_LINE(r.EMPLOYEE_NAME);
  END LOOP;
END;
/
```

### 8.3 FOR UPDATE / WHERE CURRENT OF

加行锁并支持游标内"按当前位置更新"：

```sql
DECLARE
  CURSOR c IS SELECT EMPLOYEE_ID, SALARY FROM EMPLOYEE
                WHERE DEPT_ID = 10 FOR UPDATE;
BEGIN
  FOR r IN c LOOP
    UPDATE EMPLOYEE SET SALARY = r.SALARY * 1.1
      WHERE CURRENT OF c;
  END LOOP;
  COMMIT;
END;
/
```

⚠️ `FOR UPDATE NOWAIT` 在 DM 中支持，`SKIP LOCKED` 视版本支持（⚠️ 待官方手册确认）。

### 8.4 REF CURSOR 与 SYS_REFCURSOR

动态结果集，常用于存储过程返回多行：

```sql
CREATE OR REPLACE PROCEDURE list_emp_by_dept(
  p_dept NUMBER,
  p_cur  OUT SYS_REFCURSOR
) AS
BEGIN
  OPEN p_cur FOR
    SELECT EMPLOYEE_ID, EMPLOYEE_NAME, SALARY
      FROM EMPLOYEE
     WHERE DEPT_ID = p_dept
     ORDER BY EMPLOYEE_ID;
END;
/

-- Java/C# 侧通过 OUT 游标获取 ResultSet
```

## 九、包（PACKAGE）

包 = 包规范（Specification，接口）+ 包体（Body，实现）。包头对外可见，包体可隐藏细节。

### 9.1 包规范

```sql
CREATE OR REPLACE PACKAGE pkg_order
AS
  -- 公有常量
  c_pending   CONSTANT VARCHAR(20) := 'PENDING';
  c_paid      CONSTANT VARCHAR(20) := 'PAID';
  -- 公有变量（会话级）
  g_current_user VARCHAR(30);

  -- 公有子程序
  FUNCTION  get_status_desc(p_status VARCHAR) RETURN VARCHAR;
  PROCEDURE create_order(p_amt NUMBER, p_id OUT NUMBER);
  PROCEDURE pay_order(p_id NUMBER);
END pkg_order;
/
```

### 9.2 包体

```sql
CREATE OR REPLACE PACKAGE BODY pkg_order
IS
  -- 私有变量（包内可见）
  g_audit_log BOOLEAN := TRUE;

  -- 公有函数实现
  FUNCTION get_status_desc(p_status VARCHAR) RETURN VARCHAR
  IS
  BEGIN
    RETURN CASE p_status
             WHEN c_pending THEN '待支付'
             WHEN c_paid    THEN '已支付'
             ELSE '未知'
           END;
  END;

  PROCEDURE create_order(p_amt NUMBER, p_id OUT NUMBER)
  IS
  BEGIN
    INSERT INTO ORDERS(ORDER_ID, AMT, STATUS) VALUES(SEQ_ORDER.NEXTVAL, p_amt, c_pending);
    p_id := SEQ_ORDER.CURRVAL;
  END;

  PROCEDURE pay_order(p_id NUMBER)
  IS
  BEGIN
    UPDATE ORDERS SET STATUS = c_paid, PAY_TIME = SYSDATE WHERE ORDER_ID = p_id;
    IF SQL%NOTFOUND THEN
      RAISE_APPLICATION_ERROR(-20001, '订单不存在');
    END IF;
  END;

  -- 私有过程
  PROCEDURE log(p_msg VARCHAR) IS
  BEGIN
    IF g_audit_log THEN
      INSERT INTO ORDER_LOG(MSG) VALUES(p_msg);
    END IF;
  END;

-- 初始化块：每个会话第一次引用包时执行一次
BEGIN
  g_current_user := USER;
END pkg_order;
/
```

### 9.3 重载（Overload）

包内允许同名不同参数的子程序：

```sql
-- 包头中声明两个 calc_area
FUNCTION calc_area(a NUMBER, b NUMBER, c NUMBER) RETURN NUMBER;
FUNCTION calc_area(side_list VARCHAR(24)) RETURN NUMBER;

-- 包体中分别实现
```

⚠️ Schema 级别的存储过程**不支持重载**（参数类型不同但名字相同时会编译失败），只有包内或嵌套子程序可重载。

### 9.4 编译与删除

```sql
ALTER PACKAGE pkg_order COMPILE PACKAGE;         -- 重新编译（同时校验头+体）
ALTER PACKAGE pkg_order COMPILE SPECIFICATION;   -- 仅编译头
ALTER PACKAGE pkg_order COMPILE BODY;            -- 仅编译体
DROP PACKAGE pkg_order;                          -- 同时删头+体
DROP PACKAGE pkg_order KEEP BODY;                -- ⚠️ 待官方手册确认是否支持
```

⚠️ **DM 不会自动校验包体是否仍符合包头**——修改包头后必须手动重新编译包体，否则运行期报错。

### 9.5 嵌套子程序 + 前置声明

DECLARE 区中定义的子程序称"嵌套子程序"，仅在块内可见：

```sql
DECLARE
  PROCEDURE pro1(n NUMBER);  -- 前置声明（forward declaration）
  PROCEDURE pro2(n NUMBER) IS BEGIN pro1(n); END;
  PROCEDURE pro1(n NUMBER) IS BEGIN DBMS_OUTPUT.PUT_LINE(n); END;
BEGIN
  pro2(100);
END;
/
```

## 十、动态 SQL

DMSQL 支持在程序运行时拼接并执行 DDL/DML/匿名块，配套批量绑定与异常继续机制，对应 PL/SQL 的 `EXECUTE IMMEDIATE` + `BULK COLLECT` + `FORALL`。

### 10.1 EXECUTE IMMEDIATE（四种形态）

| 形态 | 适用场景 |
|------|----------|
| `EXECUTE IMMEDIATE sql_str` | 静态 SQL、单条语句 |
| `EXECUTE IMMEDIATE sql_str USING v1, v2` | 带绑定变量 |
| `EXECUTE IMMEDIATE sql_str USING ... RETURNING INTO v_ret` | DML 返回值（如 INSERT 主键） |
| `EXECUTE IMMEDIATE sql_str USING ... BULK COLLECT INTO coll` | 多行结果批量收集 |

```sql
DECLARE
  v_table VARCHAR(30) := 'EMPLOYEE';
  v_cnt   NUMBER;
BEGIN
  -- 1) 静态 SQL：建表
  EXECUTE IMMEDIATE 'CREATE TABLE T_DYN_TEST(ID INT, NAME VARCHAR(50))';

  -- 2) USING 绑定变量
  EXECUTE IMMEDIATE
    'INSERT INTO EMPLOYEE(EMPLOYEE_ID, EMPLOYEE_NAME) VALUES(:1, :2)'
    USING 9001, '动态员工';

  -- 3) RETURNING INTO 获取序列值
  EXECUTE IMMEDIATE
    'INSERT INTO ORDERS(ORDER_ID, AMT) VALUES(SEQ_ORDER.NEXTVAL, :1)
     RETURNING ORDER_ID INTO :2'
    USING 999.00
    RETURNING INTO v_cnt;

  -- 4) 动态 BULK COLLECT
  EXECUTE IMMEDIATE
    'SELECT EMPLOYEE_ID FROM EMPLOYEE WHERE DEPT_ID = :1'
    BULK COLLECT INTO v_ids  -- v_ids 需在 DECLARE 区预定义 t_id 集合
    USING 10;
END;
/
```

⚠️ 绑定变量占位符 DM 兼容 `:1, :2` 与 `:name` 两种；建议统一使用 `:1` 命名以减少与 H2 / PostgreSQL 习惯冲突。

### 10.5 综合示例：ETL 一把走（DDL + BULK + FORALL + SAVE EXCEPTIONS）

将四种动态 SQL 能力串成"建临时表 → 批量抓数 → 批量写入 → 错误汇总"的 ETL 闭环，是数据迁移脚本最常用的写法：

```sql
DECLARE
  TYPE t_id   IS TABLE OF NUMBER;
  TYPE t_name IS TABLE OF VARCHAR(50);
  v_ids   t_id;
  v_names t_name;
  dml_errors EXCEPTION;
  PRAGMA EXCEPTION_INIT(dml_errors, -24381);
BEGIN
  -- 1) DDL：动态建临时表（表名带会话唯一后缀，避免冲突）
  EXECUTE IMMEDIATE 'CREATE GLOBAL TEMPORARY TABLE TMP_EMP_ETL_' ||
                    USER || '(ID INT, NAME VARCHAR(50))';

  -- 2) BULK COLLECT：把源库整批拉过来
  SELECT EMPLOYEE_ID, EMPLOYEE_NAME
    BULK COLLECT INTO v_ids, v_names
    FROM EMPLOYEE WHERE DEPT_ID = 10;

  -- 3) FORALL：批量写入临时表（性能比游标循环高 10-50 倍）
  FORALL i IN 1..v_ids.COUNT SAVE EXCEPTIONS
    INSERT INTO TMP_EMP_ETL_DMHR VALUES(v_ids(i), v_names(i));

  -- 4) 一条 RETURNING BULK COLLECT 立即验证
  EXECUTE IMMEDIATE 'SELECT ID, NAME FROM TMP_EMP_ETL_' || USER || ' ORDER BY ID'
    BULK COLLECT INTO v_ids, v_names;
  DBMS_OUTPUT.PUT_LINE('临时表共 ' || v_ids.COUNT || ' 行');

EXCEPTION
  WHEN dml_errors THEN
    FOR i IN 1..SQL%BULK_EXCEPTIONS.COUNT LOOP
      DBMS_OUTPUT.PUT_LINE(
        '索引 ' || SQL%BULK_EXCEPTIONS(i).ERROR_INDEX ||
        ' 错误码 ' || SQL%BULK_EXCEPTIONS(i).ERROR_CODE);
    END LOOP;
  WHEN OTHERS THEN
    DBMS_OUTPUT.PUT_LINE(DBMS_UTILITY.FORMAT_ERROR_BACKTRACE);
    RAISE;
END;
/
```

最佳实践：(a) 表名/列名走白名单校验，避免 SQL 注入；(b) 集合类型声明为 `TABLE OF ... INDEX BY PLS_INTEGER` 可省去"下标必须从 1 开始"的约束；(c) 临时表用 `GLOBAL TEMPORARY`，会话结束自动清空。

### 10.2 BULK COLLECT INTO（一次性收集多行）

把查询结果整批装入集合变量，避免逐行 `FETCH` 引发的高频上下文切换：

```sql
DECLARE
  TYPE t_id_tab IS TABLE OF NUMBER;
  v_ids t_id_tab;
BEGIN
  SELECT EMPLOYEE_ID BULK COLLECT INTO v_ids FROM EMPLOYEE WHERE DEPT_ID = 10;
  DBMS_OUTPUT.PUT_LINE('共 ' || v_ids.COUNT || ' 行');
END;
/
```

### 10.3 FORALL（批量 DML）

类似 BULK COLLECT 的"反向"——把集合变量一次性喂给 DML，性能比循环单条 UPDATE 高 5-50 倍：

```sql
DECLARE
  TYPE t_id IS TABLE OF NUMBER;
  TYPE t_amt IS TABLE OF NUMBER;
  v_ids t_id := t_id(1001, 1002, 1003);
  v_raise t_amt := t_amt(500, 800, 1200);
BEGIN
  FORALL i IN 1..v_ids.COUNT
    UPDATE EMPLOYEE SET SALARY = SALARY + v_raise(i)
      WHERE EMPLOYEE_ID = v_ids(i);
  DBMS_OUTPUT.PUT_LINE('更新 ' || SQL%ROWCOUNT || ' 行');
END;
/
```

### 10.4 FORALL SAVE EXCEPTIONS（部分失败继续）

`SAVE EXCEPTIONS` 让 FORALL 遇到单行错误不中断，全部跑完后通过 `SQL%BULK_EXCEPTIONS` 一次性收集所有错误索引与错误码：

```sql
DECLARE
  TYPE t_id IS TABLE OF NUMBER;
  v_ids t_id := t_id(1001, 9999999, 1002, 8888888, 1003);  -- 含 2 个不存在的 ID
  dml_errors EXCEPTION;
  PRAGMA EXCEPTION_INIT(dml_errors, -24381);  -- DM/ORA 通用 FORALL 错误码
BEGIN
  FORALL i IN 1..v_ids.COUNT SAVE EXCEPTIONS
    UPDATE EMPLOYEE SET SALARY = SALARY * 1.1 WHERE EMPLOYEE_ID = v_ids(i);
EXCEPTION
  WHEN dml_errors THEN
    DBMS_OUTPUT.PUT_LINE('失败 ' || SQL%BULK_EXCEPTIONS.COUNT || ' 条');
    FOR i IN 1..SQL%BULK_EXCEPTIONS.COUNT LOOP
      DBMS_OUTPUT.PUT_LINE(
        '行 ' || SQL%BULK_EXCEPTIONS(i).ERROR_INDEX ||
        ' 错误码 ' || SQL%BULK_EXCEPTIONS(i).ERROR_CODE);
    END LOOP;
    COMMIT;  -- 已成功的行仍生效
END;
/
```

## 十一、调试技巧

### 11.1 DBMS_OUTPUT + SERVEROUTPUT

最常用的"打桩"调试方式。DM DIsql 默认关闭输出，必须显式打开：

```sql
SET SERVEROUTPUT ON            -- DIsql 打开缓冲（默认 20000 字节）
SET SERVEROUTPUT ON SIZE 100000  -- 大对象加大缓冲
DBMS_OUTPUT.PUT_LINE('当前 ID=' || v_id);
DBMS_OUTPUT.PUT(v_part);         -- 不换行拼接
DBMS_OUTPUT.NEW_LINE();          -- 手动换行
```

### 11.2 SQL Trace + tkprof（重型调试）

当存储过程逻辑复杂、PUT_LINE 难以定位时，开启 SQL Trace 抓取每次执行的真实 SQL、等待事件、绑定变量、耗时：

```sql
-- 1) 启动跟踪（会话级 / 实例级）
ALTER SESSION SET SQL_TRACE = TRUE;
-- 或 DBMS_MONITOR.SESSION_TRACE_ENABLE（推荐，可指定 binds / waits）

-- 过程主体
BEGIN
  pkg_order.create_order(100, v_id);
END;
/

-- 2) 关闭
ALTER SESSION SET SQL_TRACE = FALSE;
-- 或 DBMS_MONITOR.SESSION_TRACE_DISABLE
```

跟踪文件落地在 `dm.ini` 配置的 `TRACE_PATH`（默认 `LOG`），命名 `SID_ora_<SPID>.trc`。再用 tkprof 工具解析（DM 自带）：

```bash
tkprof dm8_ora_12345.trc output.txt sys=no sort=elapsed
# output.txt 中可看：SQL 文本、解析/执行次数、elapsed/cpu 时间、磁盘读、绑定变量值
```

### 11.3 DBMS_UTILITY.FORMAT_ERROR_BACKTRACE

异常发生时打印**精确的调用栈**（行号 + 子程序名），比 `SQLERRM` 强 N 倍：

```sql
CREATE OR REPLACE PROCEDURE proc_inner(p_id NUMBER) AS
BEGIN
  UPDATE EMPLOYEE SET SALARY = NULL WHERE EMPLOYEE_ID = p_id;
END;
/
CREATE OR REPLACE PROCEDURE proc_outer(p_id NUMBER) AS
BEGIN
  proc_inner(p_id);
EXCEPTION
  WHEN OTHERS THEN
    DBMS_OUTPUT.PUT_LINE(DBMS_UTILITY.FORMAT_ERROR_BACKTRACE);
    -- 输出形如：ORA-01400 at line 2 in "DMHR.PROC_INNER"
    RAISE;
END;
/
```

### 11.4 5 条常见错误速查

| 错误现象 | 根因 | 解决 |
|----------|------|------|
| `ORA-01403: no data found` | `SELECT INTO` 无返回 | 加 `IF SQL%NOTFOUND` 或改用聚合 `MAX/MIN` |
| `ORA-01422: exact fetch returns more than requested number of rows` | `SELECT INTO` 返回多行 | 改用 `BULK COLLECT` 或加 `WHERE rownum=1` |
| `ORA-06502: numeric or value error` | 类型溢出 / 字符串截断 | 扩大变量长度（`%TYPE` 锚定），确认 `NUMBER(p,s)` 精度 |
| `ORA-04091: table X is mutating` | 行级触发器内又读/写同一张表 | 拆分为语句级 + `AFTER` + 自治事务，或改用预计算表 |
| `ORA-14551: cannot perform a DML inside a query` | 函数体内做了 DML | 改用存储过程；或加 `PRAGMA AUTONOMOUS_TRANSACTION`（⚠️ 见十二节差异） |

### 11.5 调试权限与日志

普通用户开启 SQL Trace 需要 `DBA` 角色或 `DBMS_MONITOR` 的 EXECUTE 权限；生产环境调试结束后记得 `ALTER SESSION SET SQL_TRACE = FALSE` 并定期清理 `LOG` 目录，否则跟踪文件可能膨胀到 GB 级。

## 十二、DM8 与 Oracle PL/SQL 差异

### 12.1 兼容点（直接迁移，5 项）

| # | 兼容点 | 说明 |
|---|--------|------|
| 1 | **块结构 / 流程控制** | DECLARE/BEGIN/EXCEPTION/END、IF/CASE/LOOP/WHILE/FOR/EXIT/CONTINUE/GOTO 完全一致 |
| 2 | **数据类型** | VARCHAR2 / NUMBER / DATE / BLOB / CLOB 与 Oracle 等价；%TYPE / %ROWTYPE 完整支持 |
| 3 | **游标 + REF CURSOR** | 显式游标、FOR UPDATE、WHERE CURRENT OF、SYS_REFCURSOR 均兼容 |
| 4 | **包（PACKAGE）** | 规范/主体分离、公有/私有、重载、前置声明语法一致 |
| 5 | **异常处理** | 预定义异常、PRAGMA EXCEPTION_INIT、RAISE_APPLICATION_ERROR、SQLCODE/SQLERRM 一致 |

### 12.2 不兼容点（迁移需手工调整，5 项）

| # | 不兼容点 | Oracle 行为 | DM8 行为 / 解决方案 |
|---|----------|-------------|---------------------|
| 1 | **自治事务 (PRAGMA AUTONOMOUS_TRANSACTION)** | 标准支持，可在触发器/函数内独立 COMMIT | DM 行为差异较大，⚠️ 高并发场景触发器内禁止自治事务，否则会破坏守护集群一致性。改用包变量 + 异步日志表 |
| 2 | **PIPELINED 函数** | 用对象类型 + 嵌套表 | DM 需要先 `CREATE TYPE` 对象 + 嵌套表（语法略不同），单条 SQL `TABLE()` 解引用一致 |
| 3 | **MERGE / UPSERT 语法** | 完整 `MERGE INTO ... USING ... ON ... WHEN MATCHED` | DM 支持但 `UPDATE` 子句中**不允许引用目标表的非连接列**，需先 `SELECT ... INTO` 再 UPDATE |
| 4 | **序列（SEQUENCE）`CURRVAL`** | 会话级缓存，NEXTVAL 之后立即可读 | DM 跨会话**首次 NEXTVAL 后即可 CURRVAL**，但**同会话内未先 NEXTVAL 而读 CURRVAL 会报错**（与 Oracle 行为一致，但报错码不同） |
| 5 | **`BULK_EXCEPTIONS` 错误码** | `ORA-24381` | DM 行为**近似但错误码不同**，需用 `WHEN OTHERS` + `SQLCODE` 模糊匹配；如要严格兼容需用 `PRAGMA EXCEPTION_INIT` 手动绑定 |

> 总体策略：先静态语法分析（`grep` 关键字），再在测试环境跑 `DMHS` 数据同步 + 单元测试，最后做 7×24 小时的压测验证。剩余 ~5% 差异主要集中在自治事务、对象类型高级特性、并行 PL/SQL 上。

## 十三、实战：订单状态机

完整可运行示例：订单从 `PENDING → PAID → SHIPPED → COMPLETED` 流转，使用**表 + 包 + 触发器 + 动态 SQL** 完整实现状态校验与历史记录。

### 13.1 表结构

```sql
-- 订单主表
CREATE TABLE ORDERS(
  ORDER_ID     INT PRIMARY KEY,
  AMT          NUMBER(12,2) NOT NULL,
  STATUS       VARCHAR(20)  NOT NULL,   -- PENDING/PAID/SHIPPED/COMPLETED/CANCELLED
  CREATE_TIME  DATETIME     DEFAULT SYSDATE,
  UPDATE_TIME  DATETIME
);

-- 状态变更历史
CREATE TABLE ORDER_STATUS_LOG(
  LOG_ID     INT IDENTITY(1,1) PRIMARY KEY,
  ORDER_ID   INT,
  FROM_ST    VARCHAR(20),
  TO_ST      VARCHAR(20),
  OPERATOR   VARCHAR(30),
  CHG_TIME   DATETIME DEFAULT SYSDATE
);

-- 状态字典（可选，便于前端下拉）
CREATE TABLE ORDER_STATUS_DICT(
  CODE  VARCHAR(20) PRIMARY KEY,
  DESC  VARCHAR(50)
);
INSERT INTO ORDER_STATUS_DICT VALUES
  ('PENDING',   '待支付'),
  ('PAID',      '已支付'),
  ('SHIPPED',   '已发货'),
  ('COMPLETED', '已完成'),
  ('CANCELLED', '已取消');
```

### 13.2 包规范

```sql
CREATE OR REPLACE PACKAGE pkg_order_sm
AS
  -- 合法状态转换矩阵
  FUNCTION is_valid_transition(p_from VARCHAR, p_to VARCHAR) RETURN BOOLEAN;
  -- 推进状态（核心入口）
  PROCEDURE transit(p_order_id INT, p_to VARCHAR, p_operator VARCHAR);
  -- 列出某订单所有历史
  PROCEDURE list_history(p_order_id INT, p_cur OUT SYS_REFCURSOR);
END pkg_order_sm;
/
```

### 13.3 包体（核心逻辑 + 动态 SQL）

```sql
CREATE OR REPLACE PACKAGE BODY pkg_order_sm
IS
  -- 转换规则：PENDING→PAID/CANCELLED；PAID→SHIPPED/CANCELLED；SHIPPED→COMPLETED；终态不变
  FUNCTION is_valid_transition(p_from VARCHAR, p_to VARCHAR) RETURN BOOLEAN
  IS
    v_ok BOOLEAN := FALSE;
  BEGIN
    -- 使用动态 SQL 拼转换规则，便于热更新（生产环境可改查规则表）
    EXECUTE IMMEDIATE
      'SELECT CASE WHEN :1 IN (''PENDING'')  AND :2 IN (''PAID'',''CANCELLED'')
                    WHEN :1 IN (''PAID'')     AND :2 IN (''SHIPPED'',''CANCELLED'')
                    WHEN :1 IN (''SHIPPED'')  AND :2 IN (''COMPLETED'')
                    ELSE ''N'' END FROM DUAL'
      INTO v_ok USING p_from, p_to, p_from, p_to, p_from, p_to;
    RETURN v_ok;
  END;

  PROCEDURE transit(p_order_id INT, p_to VARCHAR, p_operator VARCHAR)
  IS
    v_from VARCHAR(20);
    e_invalid_transition EXCEPTION;
    PRAGMA EXCEPTION_INIT(e_invalid_transition, -20010);
  BEGIN
    -- 1) 加行锁，避免并发改状态
    SELECT STATUS INTO v_from FROM ORDERS
      WHERE ORDER_ID = p_order_id FOR UPDATE;
    -- 2) 校验
    IF NOT is_valid_transition(v_from, p_to) THEN
      RAISE_APPLICATION_ERROR(-20010,
        '非法状态转换：' || v_from || ' → ' || p_to);
    END IF;
    -- 3) 更新 + 写历史
    UPDATE ORDERS SET STATUS = p_to, UPDATE_TIME = SYSDATE
      WHERE ORDER_ID = p_order_id;
    INSERT INTO ORDER_STATUS_LOG(ORDER_ID, FROM_ST, TO_ST, OPERATOR)
      VALUES(p_order_id, v_from, p_to, p_operator);
    COMMIT;
  EXCEPTION
    WHEN NO_DATA_FOUND THEN
      RAISE_APPLICATION_ERROR(-20011, '订单不存在：' || p_order_id);
  END;

  PROCEDURE list_history(p_order_id INT, p_cur OUT SYS_REFCURSOR)
  IS
  BEGIN
    OPEN p_cur FOR
      SELECT LOG_ID, FROM_ST, TO_ST, OPERATOR, CHG_TIME
        FROM ORDER_STATUS_LOG
       WHERE ORDER_ID = p_order_id
       ORDER BY CHG_TIME;
  END;
END pkg_order_sm;
/
```

### 13.4 触发器（自动记录创建时间 + 防非法直接 UPDATE）

```sql
-- 自动写入创建时间
CREATE OR REPLACE TRIGGER trg_orders_bi
BEFORE INSERT ON ORDERS
FOR EACH ROW
WHEN (NEW.STATUS IS NULL)   -- 调用方未显式指定时默认 PENDING
BEGIN
  :NEW.STATUS := 'PENDING';
  :NEW.CREATE_TIME := SYSDATE;
END;
/

-- 拦截直接改状态（必须走包）
CREATE OR REPLACE TRIGGER trg_orders_bu
BEFORE UPDATE OF STATUS ON ORDERS
FOR EACH ROW
WHEN (OLD.STATUS != NEW.STATUS)
DECLARE
  v_pkg VARCHAR(30) := OWA_UTIL.WHO_CALLED_ME;  -- ⚠️ 待官方手册确认 DM 是否支持
BEGIN
  -- 简易做法：检查调用栈中是否包含 pkg_order_sm
  IF INSTR(v_pkg, 'PKG_ORDER_SM') = 0 THEN
    RAISE_APPLICATION_ERROR(-20020,
      '禁止直接 UPDATE STATUS，必须调用 pkg_order_sm.transit()');
  END IF;
END;
/
```

### 13.5 完整业务跑通

```sql
-- 1) 创建订单（触发器自动写 PENDING）
INSERT INTO ORDERS(ORDER_ID, AMT) VALUES(1001, 599.00);
INSERT INTO ORDERS(ORDER_ID, AMT) VALUES(1002, 1299.00);

-- 2) 走包推进
BEGIN
  pkg_order_sm.transit(1001, 'PAID',      'user_alice');   -- OK
  pkg_order_sm.transit(1001, 'SHIPPED',   'sys_logistics');-- OK
  pkg_order_sm.transit(1001, 'COMPLETED', 'user_alice');   -- OK
END;
/

-- 3) 非法转换应被拒绝
BEGIN
  pkg_order_sm.transit(1001, 'PENDING', 'user_bob');  -- -20010 非法转换
EXCEPTION
  WHEN OTHERS THEN
    DBMS_OUTPUT.PUT_LINE(SQLCODE || ':' || SQLERRM);
END;
/

-- 4) 直 UPDATE 被触发器拦截
UPDATE ORDERS SET STATUS = 'CANCELLED' WHERE ORDER_ID = 1002;
-- 抛出 -20020: 禁止直接 UPDATE STATUS

-- 5) 查历史
VAR v_cur REFCURSOR;
CALL pkg_order_sm.list_history(1001, :v_cur);
PRINT v_cur;
```

业务收益：状态校验 + 历史审计 + 防绕过三层防护在数据库内聚，跨语言（Java/C#/Go）调用契约完全一致。

## 参考资料

1. DM8 SQL 概述与块结构 — https://eco.dameng.com/document/dm/zh-cn/pm/dm8_sql-overview.html
2. DM8 数据类型与运算符 — https://eco.dameng.com/document/dm/zh-cn/pm/dm8_sql-data-types-operators.html
3. DM8 SQL 定义与删除 — https://eco.dameng.com/document/dm/zh-cn/pm/dm8_sql-definition-deletion.html
4. DM8 流程控制语句 — https://eco.dameng.com/document/dm/zh-cn/pm/dm8_sql-various-control.html
5. DM8 SQL 语句参考 — https://eco.dameng.com/document/dm/zh-cn/pm/dm8_sql-sql-statement.html
6. DM8 异常处理 — https://eco.dameng.com/document/dm/zh-cn/pm/dm8_sql-exception-handling.html
7. DM8 PL/SQL 调试指南 — https://eco.dameng.com/document/dm/zh-cn/pm/dm8_sql-debug.html
8. DMPL-SQL 数据类型 — https://eco.dameng.com/document/dm/zh-cn/sql-dev/dmpl-sql-datatype.html
9. DMPL-SQL 异常处理 — https://eco.dameng.com/document/dm/zh-cn/sql-dev/dmpl-sql-exception.html
10. DMPL-SQL 集合与游标 — https://eco.dameng.com/document/dm/zh-cn/sql-dev/dmpl-sql-set.html
11. 包（PACKAGE）实践 — https://eco.dameng.com/document/dm/zh-cn/sql-dev/practice-package.html
12. 存储过程实践 — https://eco.dameng.com/document/dm/zh-cn/sql-dev/practice-pro.html
13. 函数实践 — https://eco.dameng.com/document/dm/zh-cn/sql-dev/practice-func.html
14. 触发器实践 — https://eco.dameng.com/document/dm/zh-cn/sql-dev/practice-trg.html
