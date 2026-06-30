# 迁移后问题研判与复盘

用于分析达梦迁移后记录的问题清单、缺陷台账、接口报错表或上线回归问题。目标不是把所有问题都归因于达梦，而是区分“达梦迁移系统性风险”和“非达梦迁移问题”，并把可复用规则回写到迁移计划。

## 研判流程

1. 先抽取台账字段：菜单、模块、功能、接口、问题现象、报错日志、处理措施、标签。截图公式或图片 ID 只作为证据索引，不作为文字结论。
2. 对每条问题判定类别：达梦 SQL/DDL 兼容、达梦类型语义、ORM 映射、数据规则、应用 I/O、非达梦迁移问题。
3. 对达梦相关类别做系统性排查，不只修当前接口。每个问题都要回答：是否同类代码/表结构也存在、如何扫描、如何验证。
4. 输出复盘结论：按类别统计数量，列代表性接口，写风险等级、补充扫描命令、验证方式和需要业务确认项。
5. 只把可复用的迁移规则写回技能；不要把单个业务状态值、项目名或一次性修复写进通用技能。

## 分类标准

| 类别 | 典型信号 | 研判结论 |
|------|----------|----------|
| 达梦 SQL/DDL 兼容 | SQL 改写错误、保留字、表别名、同名字段、`GROUP BY`、大小写、反引号、函数差异 | 高相关，需要补静态扫描和 SQL 可执行验证 |
| 达梦类型语义 | `CHAR` 补空格、`IDENTITY` 自增、日期零值、布尔值、字符/字节长度 | 高相关，需要查 DDL 和代码读写方式 |
| ORM 映射 | `@TableField`、`@TableId`、`useGeneratedKeys`、插入 ID 为 null、字段大小写 | 高相关，需要查实体、mapper、生成 SQL |
| 数据规则 | 字典排序、状态枚举、逻辑删除过滤、历史脏数据 | 中相关，迁移可能暴露问题，但通常需业务确认 |
| 应用 I/O / 外部集成 | 文件解析、上传下载、消息连接、前端状态、工具类封装等无法追溯到达梦方言的问题 | 低相关，按项目问题记录，不纳入通用达梦技能 |
| 非达梦迁移问题 | 无法追溯到达梦 SQL、类型、驱动、连接池或 ORM 方言的问题 | 低相关，忽略具体项目修复细节，只记录“不纳入通用达梦技能”的结论 |

## 从现场问题推导出的高价值规则

### 1. 保留字和对象名大小写

出现 `reference`、`model`、`number`、`user`、`order`、`level`、`comment`、`group`、`index` 等字段问题时，先确认目标库真实列名，再决定是否加双引号。不要凭 Java 字段名直接写 `@TableField("\"xxx\"")`。

验证目标列名：

```sql
SELECT OWNER, TABLE_NAME, COLUMN_NAME
FROM ALL_TAB_COLUMNS
WHERE TABLE_NAME = UPPER('YOUR_TABLE')
  AND UPPER(COLUMN_NAME) IN ('REFERENCE','MODEL','NUMBER','USER','ORDER','LEVEL','COMMENT');
```

项目扫描：

```bash
rg -n --glob '*.java' --glob '*.xml' '\b(reference|model|number|user|order|level|comment|group|index)\b|@TableField|`[^`]+`' src
```

判定规则：
- 如果 DTS 已把对象名统一转大写，实体映射优先写 `@TableField("\"REFERENCE\"")` 这种目标库真实大小写。
- 如果目标对象是带双引号保留的大小写敏感名称，映射必须完全匹配库中大小写。
- 同一项目不要混用“统一大写”和“保留小写双引号”两种策略。

### 2. `CHAR` 补空格

迁移后接口返回字段包含多个空格，常见根因是 MySQL 短状态码迁到 DM `CHAR(n)`。DM 会按固定长度补空格，列表、枚举、JSON 返回和等值比较都可能异常。

数据库扫描：

```sql
SELECT OWNER, TABLE_NAME, COLUMN_NAME, DATA_TYPE, DATA_LENGTH
FROM ALL_TAB_COLUMNS
WHERE DATA_TYPE = 'CHAR'
  AND DATA_LENGTH > 1
ORDER BY OWNER, TABLE_NAME, COLUMN_NAME;
```

处理优先级：
1. 状态码、枚举、逻辑删除、启停标识优先改为 `VARCHAR` 或缩短为真实长度。
2. 无法改 DDL 时，在查询或 DTO 层统一 `RTRIM`/trim，但要记录技术债。
3. 对所有列表接口补充状态字段回归，检查返回 JSON 是否带尾随空格。

### 3. 表别名与同名字段

迁移后多表查询报字段歧义、列名无效或执行结果异常时，优先检查别名规则。达梦比 MySQL 更容易暴露“裸列名可解析但语义不清”的问题。

处理规则：
- 多表 SQL 中不同表包含相同字段时，`SELECT`、`WHERE`、`JOIN ON`、`GROUP BY`、`HAVING`、`ORDER BY` 都必须使用表别名限定字段，例如 `p.id`、`u.id`，不要写裸 `id`。
- 表一旦在 `FROM`/`JOIN` 中定义别名，后续引用必须使用该别名；不要再写原表名，也不要省略别名。
- 子查询、派生表、CTE 也要给清晰别名；外层只能引用外层可见别名。
- 列别名优先用于结果展示和外层查询；`WHERE` 中不要引用同层 `SELECT` 列别名，`GROUP BY`/`HAVING` 是否能用列别名要按达梦实测，稳妥做法是重复原表达式或包一层子查询。

扫描方向：

```bash
# 初筛多表查询、别名和分组相关 SQL，命中后人工检查裸字段和别名一致性
rg -n --glob '*.xml' --glob '*.sql' '\b(FROM|JOIN|LEFT JOIN|RIGHT JOIN|INNER JOIN)\b|GROUP BY|HAVING|ORDER BY' src/main/resources
```

验证方式：
1. 对每个多表 SQL 列出表别名表，例如 `project p`、`user u`。
2. 标记同名字段：`id`、`name`、`status`、`create_time`、`update_time`、`del_flag` 等。
3. 检查所有同名字段是否都带表别名。
4. SQL 中已有表别名时，检查是否还混用了原表名或裸列名。
5. 在达梦执行核心 SQL，确认无字段歧义且结果行数与原库一致。

### 4. `IDENTITY` 自增与插入 null

达梦 `IDENTITY` 列不要在业务 INSERT 中显式传 `NULL`。迁移后常见错误是 mapper 或手写 SQL 包含 `id` 列并赋值 `null`，导致新增失败。

扫描方向：

```bash
rg -n --glob '*.xml' --glob '*.java' 'INSERT\s+INTO|batchInsert|insertBatch|saveBatch|<foreach|useGeneratedKeys|@TableId|IdType\.AUTO|IDENTITY|AUTO_INCREMENT|setId\(null\)|\bid\s*=\s*null' src
```

处理规则：
- MyBatis XML：目标表 ID 为 `IDENTITY` 时，INSERT 列表中去掉 ID 列，让数据库生成。
- `batchInsert`、`insertBatch`、`saveBatch`、`<foreach>` 批量插入路径要单独核对列清单；只要目标表主键是 `IDENTITY`，批量 SQL 中也不得包含 ID 列，不要传 `id = null`。
- MyBatis-Plus：实体主键使用 `@TableId(type = IdType.AUTO)`；不要在 insert 前手动 `setId(null)` 作为“占位”。
- 如果业务需要可控 ID，改用序列或雪花 ID，不要同时依赖 `IDENTITY`。
- 迁移验收必须覆盖“新增后返回 ID”和“批量新增”路径；批量路径至少验证连续插入不冲突、导入数据不覆盖已有 ID。

### 5. MyBatis `<if>` 空串条件

迁移后“传空字符串查不到数据”通常不是单一达梦问题，而是动态 SQL 条件只判断 `null`，没有排除空串，导致生成 `= ''` 或 `LIKE '%%'` 之外的异常条件。

扫描方向：

```bash
rg -n --glob '*.xml' '<if\s+test="[^"]*!=\s*null' src/main/resources
rg -n --glob '*.java' 'StringUtils\.isNotBlank|StringUtils\.isBlank|isEmpty\(' src/main/java
```

处理规则：
- MyBatis XML 字符串条件必须同时判断 `xxx != null and xxx != ''`；只判断 `!= null` 的都要补空串判断。
- 非字符串字段不要机械加 `!= ''`，先确认字段类型；数字、日期、布尔字段按类型判断。
- 如果使用 OGNL 方法或工具类判断，必须保证空白字符串也被排除。
- Java 条件判断优先用 `StringUtils.isNotBlank`。
- 验收时覆盖 `null`、空串、空白字符串和真实关键词四种输入，检查生成 SQL 和接口结果。

### 6. `GROUP` 相关问题

`group问题` 需要按具体报错细分，迁移复盘里至少检查三类：

1. **`GROUP BY` 严格性**：查询列中非聚合字段必须出现在 `GROUP BY` 中；MySQL 宽松模式能跑的 SQL，达梦可能报错或结果不稳定。
2. **聚合拼接**：`GROUP_CONCAT` 改 `LISTAGG` 时必须补 `WITHIN GROUP (ORDER BY ...)`，否则拼接顺序可能漂移。
3. **保留字/字段名**：字段或别名叫 `group` 时，按目标库真实列名加双引号或改名，避免和 `GROUP BY` 语法冲突。

扫描方向：

```bash
rg -n --glob '*.xml' --glob '*.sql' 'GROUP BY|GROUP_CONCAT\(|LISTAGG\(|\bgroup\b' src/main/resources
```

验证方式：
- 对 `GROUP BY` SQL，逐个核对 `SELECT` 中非聚合表达式是否都在 `GROUP BY`。
- 对 `LISTAGG`，确认排序字段业务稳定且结果与原系统一致。
- 对分组查询涉及多表字段的 SQL，同时执行“表别名与同名字段”检查。

### 7. 逻辑删除和业务状态

迁移后出现同名任务重复、转办状态查不到、字典排序不一致时，先判定是否为业务数据规则问题。迁移可能放大这些问题，但修复点通常是业务条件或数据，不是数据库兼容参数。

必须检查：
- 唯一性/重复性查询是否过滤逻辑删除字段，例如 `del_flag = 0`。
- 状态枚举是否覆盖迁移后现场流程的全部状态值。
- 字典排序字段是否有有效差异；如果所有值相同，排序结果本来就不稳定。
- 迁移前后是否需要补数据修复脚本，并明确脚本只执行在哪个库。

### 8. 应用 I/O 和外部集成边界

迁移后问题台账里常混入文件导入导出、模板解析、上传下载、消息连接、前端状态和工具类封装问题。它们通常不是达梦 SQL 问题，除非能追溯到达梦 SQL、类型、驱动、连接池或 ORM 方言，否则不要写入通用达梦技能。

处理规则：
- 先确认失败链路是否实际访问达梦；如果没有 SQL、JDBC、连接池或 ORM 证据，默认归为项目问题。
- 只记录问题边界、影响范围和是否需要项目专项回归，不沉淀具体实现细节。
- 如果文件或外部集成问题最终定位到达梦字段类型、字符集、大小写或 SQL 写法，再回到对应的达梦分类处理。

### 9. 非达梦迁移问题边界

无法追溯到达梦 SQL、类型、驱动、连接池或 ORM 方言的问题，只需要在复盘中标记为“非达梦迁移问题”，不要沉淀为达梦兼容规则。

发布通用技能时不要保留具体项目问题名称、接口名、页面名、业务状态值或一次性修复方案。达梦技能只保留驱动版本、连接池、MyBatis/MyBatis-Plus、Hibernate 方言、SQL/DDL/类型语义等与达梦直接相关的升级和适配规则。

## 复盘输出模板

```markdown
## 迁移后问题研判

- 问题总数：
- 达梦高相关：
- 达梦中相关：
- 非达梦迁移问题：

### 系统性风险
| 风险 | 代表问题 | 扫描方式 | 验证方式 | 结论 |
|------|----------|----------|----------|------|

### 不纳入达梦技能的单点问题
| 问题 | 原因 |
|------|------|

### 需要补充的技能规则
1. ...
```

## 是否需要更新技能的判定

需要更新：
- 同一类问题可通过静态扫描提前发现。
- 问题来自达梦类型语义、保留字、主键生成、兼容函数、大小写策略。
- 问题影响迁移流程、验证清单、上线回归用例。

不需要更新：
- 无法追溯到达梦 SQL、类型、驱动、连接池或 ORM 方言的非达梦迁移问题。
- 只对单个项目成立的接口名、页面名、业务状态值或一次性修复方案。
