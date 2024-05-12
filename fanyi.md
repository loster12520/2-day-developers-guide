## 9  开发一个简单的Oracle数据库应用

通过遵循开发这个简单应用的说明，您将了解开发Oracle数据库应用程序的一般过程。

- **关于应用**

  这个应用具有以下目的、结构和命名约定。

- **为应用创建模式**

  按照本节的过程，您将在您的应用中创建模式。

- **向模式授予权限**

  使用SQL语句GRANT向模式授予权限。

- **创建结构对象和加载数据**

  本节将展示如何为您的应用创建表、编辑视图、触发器以及序列，如何向表里载入数据，以及如何将这些模式对象上的权限授予需要这些权限的用户。

- **创建employees_pkg包**

  本节将展示如何创建一个employees_pkg包，如何使其子程序运行，如何将包的执行权限授予需要的用户，以及这些用户如何调用它的子程序。

- **创建admin_pkg包**

  本节将展示如何创建一个admin_pkg包，如何使其子程序运行，如何将包的执行权限授予需要的用户，以及这些用户如何调用它的子程序。

---

### 9.1 关于应用

<text id='关于应用'></text>

这个应用具有以下目的、结构和命名约定。

- **应用的目标**

  这个应用将面向公司里的两类人群。

- **应用的结构**

  这个应用使用了以下模式对象和模式。

- **应用中的命名约定**

  应用使用了下面的命名规定。

---

#### 9.1.1 应用的目标

这个应用将面向公司里的两类人群。

- 典型用户（员工经理）
- 应用管理员

典型用户能做到以下几点：

- 获取一个部门的员工信息
- 获取一个员工的任职经历
- 展示一名员工的基础信息（姓名、所属部门、所在岗位、经理、工资等等）
- 改变一名员工的工资
- 改变一名员工的所在岗位

应用管理员能做到以下几点

- 改变一个已经存在岗位的ID、岗位名称、工资范围
- 新增一个岗位
- 改变一个已经存在部门的ID、名字、经理
- 新增一个部门

---

#### 9.1.2 应用的结构

一个应用将会使用接下来要提到的模式对象和模式

- [应用的模式对象](#应用的模式对象)
- [应用的模式](#应用的模式)

<h5 id='应用的模式对象'>9.1.2.1 应用的模式对象</h5>

这个应用由这些模式对象组成：

- 四种表格，储存这些数据：

  - 岗位
  - 部门
  - 员工
  - 员工的岗位经历

- 四个涵盖了表的编辑窗口，能使您使用基于版本的重定义(edition-based redefinition,EBR)去在使用期间更新一个已经结束的应用
- 两个执行业务规则的触发器
- 两个为新员工和新部门生成唯一主键的序列
- 两个包：
  - employees_pkg，提供给典型用户的应用程序接口(application program interface,API)
  - admin_pkg，提供给应用管理员的API

> <text class='lightblue'>参见</text>：
> - “[关于Oracle数据库](一个指向不存在的地方的链接)”，关于模式对象简介部分
> - [Oracle数据库开发指南](一个指向不存在的地方的链接)，关于EBR介绍部分

<h5 id='应用的模式'>9.1.2.1 应用的模式</h5>

为了安全起见，应用程序会使用这五个模式(或用户)，每个模式都只拥有它需要的权限:

- app_data，它拥有除了包之外的所有模式对象，并从样例模式HR中的表中加载数据。

- app_code，它只拥有employees_pkg包。

  employees_pkg的开发人员在这个模式中工作。

- app_admin，他只拥有admin_pkg包

  admin_pkg的开发人员在这个模式中工作。

- app_user，典型应用用户，它不拥有东西且只能执行employees_pkg

  中间层应用服务器以app_user的身份连接到连接池中的数据库。如果该模式受到损害(例如，由于SQL注入错误)，攻击者只能看到和更改employees_pkg子程序允许它看到和更改的内容。攻击者不能删除表、升级权限、创建或修改模式对象或其他任何东西。

- app_admin_user，一个应用管理者，它不拥有东西且只能执行admin_pkg和employees_pkg

  此模式的连接池非常小，只有拥有权限在用户才能访问它。如果这个模式受到损害，攻击者只能看到和更改admin_pkg和employees_pkg子程序允许它看到和更改的内容。

假设应用程序只有一个模式，而不是app_user和app_admin_user，该模式不拥有任何东西，并且可以同时执行employees_pkg和admin_pkg。此模式的连接池必须足够大，以满足典型用户和应用程序管理员的需要。如果employees_pkg中存在SQL注入错误，则利用该错误的典型用户可以访问admin_pkg。

假设应用程序只有一个模式，而不是app_data、app_code和app_admin，该模式拥有所有模式对象，包括包。但是，包将拥有表上的所有权限，这将是不必要和不希望的。

例如，假设您有一个审计跟踪表AUDIT_TRAIL。您希望employees_pkg的开发人员能够写入AUDIT_TRAIL，但不能读取或更改它。您希望admin_pkg的开发人员能够读取AUDIT_TRAIL并写入它，但不能更改它。如果AUDIT_TRAIL、employees_pkg和admin_pkg属于同一个模式，那么这两个包的开发人员拥有AUDIT_TRAIL上的所有权限。然而，如果AUDIT_TRAIL属于app_data, employees_pkg属于app_code, admin_pkg属于app_admin，那么你可以作为app_data连接到数据库，并这样做:

```sql
GRANT INSERT ON AUDIT_TRAIL TO app_code;
GRANT INSERT, SELECT ON AUDIT_TRAIL TO app_admin;
```

> <text class='lightblue'>参见</text>：
> - “[关于Oracle数据库](一个指向不存在的地方的链接)”，关于模式简介部分
> - “[关于示例模式HR](一个指向不存在的地方的链接)”，关于示例模式HR部分
> - “推荐的安全措施”

---

#### 9.1.3 应用的命名规范

这个应用使用了下面的命名规范

| **元素** | **名字** |
|---|---|
| 表格 | table# |
| table#的编辑视图 | table |
| 编辑视图table的触发器 | table_{a\|b}event{_fer}$^1$ |
| table#的主键约束 | table_pk |
| table#.column的非空约束 | table_column_not_null $^2$ |
| table#.column的唯一约束 | table_column_unique $^2$ |
| table#.column的检查约束 | table_column_check $^2$ |
| table1#.column到table2#.column的外键约束 | table1_to_table2_fk $^2$ |
| table1#.column1到table2#.column2的外键约束 | table1_col1_to_table2_col2_fk $^2$ $^3$ |
| table#的序列 | table_sequence |
| 参数名称 | p_name |
| 本地变量名 | l_name |

$^1$ 遵循以下标准：
  - a标识了一个AFTER触发器
  - b标识了一个BEFORE触发器
  - fer标识了一个FOR EACH ROW触发器
  - event标识了触发触发器的事件。例如:i表示INSERT, iu表示INSERT或UPDATE, d表示DELETE。

$^2$ table、table1和table2被缩写成：员工->emp；部门->dept；就职经历->job_hist

$^3$ col1和col2是列名column1和column2的缩写。一个约束的名字不允许超过30个字符

---

### 9.2 为应用创建模式

按照本节的过程，您将在您的应用中创建模式。

这些模式的名字为：

- app_data
- app_code
- app_admin
- app_user
- app_admin_user

> <text class='lightblue'>注意</text>:<br>
> 对于下面的过程，您需要具有CREATE USER和DROP USER的系统权限的用户名和密码。

*为了创建一个模式（或者用户）schema_name*：
1. 使用SQL*Plus，使用一个拥有CREATE USER和DROP USER系统权限的用户连接Oracle数据库

    将会出现`>SQL`提示符

2. 如果模式存在，便使用下面的SQL语句删除掉模式及其对象

    ```sql
    DROP USER schema_name CASCADE
    ```

    如果模式存在，系统响应：

    ```
    User dropped.
    ```

    如果模式不存在，系统响应：

    ```
    DROP USER schema_name CASCADE
              *
    ERROR at line 1:
    ORA-01918:user 'schema_name' does not exits
    ```

3. 如果*schema_name*是app_data，app_code或者app_admin其中之一，那么使用下面的SQL语句创建模式：

    ```sql
    CREATE USER schema_name IDENTIFIED BY password
    DEFAULT TABLESPACE USERS
    QUOTA UNLIMITED ON USERS
    ENABLE EDITIONS;
    ```

    否则，使用下面的SQL语句创建模式：

    ```sql
    CREATE USER schema_name IDENTIFIED BY password
    ENABLE EDITIONS;
    ```

    系统会响应
    ```
    User created
    ```

> <text class='yellow'>警告</text>:<br>
> 请选择一个安全的密码。安全密码指南请参见《Oracle数据库安全指南》。

4. （可选）在SQL Developer中，使用“从SQL Developer连接到Oracle数据库”中的说明为模式创建连接。

> <text class='lightblue'>参见</text>：
> - “[关于应用](#关于应用)”
> - “[使用SQL*Plus连接到Oracle数据库](一个指向不存在的地方的链接)”
> - 《Oracle Database SQL Language Reference》关于`DROP USER`语句使用部分
> - 《Oracle Database SQL Language Reference》关于`CREATE USER`语句使用部分

---

### 9.3 向模式授予权限

为了向模式授于权限，请使用SQL语句GRANT

你可以键入GRANT语句在SQL*Plus或者SQLDeveloper的Worksheet中。为了安全，请只授予每一个模式它需要使用到的权限

- [向app_data模式授予权限](#向app_data模式授予权限)
- [向app_code模式授予权限](#向app_code模式授予权限)
- [向app_admin模式授予权限](#向app_admin模式授予权限)
- [向app_user和app_admin_user模式授予权限](#向app_user和app_admin_user模式授予权限)

---

<h4 id='向app_data模式授予权限'>9.3.1 向app_data模式授予权限</h4>

只向app_data模式授予下面这些权限：

- 连接到Oracle数据库：

```sql
GRANt CREATE SESSION TO app_data;
```

- 在这个应用里创建表格、视图、触发器以及序列

```sql
GRANT CREATE TABLE, CREATE VIEW, CREATE TRIGGER, CREATE SEQUENCE TO app_data;
```

- 从示例模式HR的四个表中向自己的表格载入数据

```sql
GRANT SELECT ON HR.DEPARTMENTS TO app_data;
GRANT SELECT ON HR.EMPLOYEES TO app_data;
GRANT SELECT ON HR.JOB_HISTORY TO app_data;
GRANT SELECT ON HR.JOBS TO app_data;
```

---

<h4 id='向app_code模式授予权限'>9.3.2 向app_code模式授予权限</h4>

只向app_code模式授予下面这些权限：

- 连接到Oracle数据库

```sql
GRANT CREATE SESSION TO app_code;
```

- 创建employees_pkg包

```sql
GRANT CREATE PROCEDURE TO app_code;
```

- 创建一个别名（为了方便）

```sql
GRANT CREATE SYNONYM TO app_code;
```

---

<h4 id='向app_admin模式授予权限'>9.3.3 向app_admin模式授予权限</h4>

只向app_admin模式授予下面这些权限：

- 连接到Oracle数据库

```sql
GRANT CREATE SESSION TO app_admin;
```

- 创建admin_pkg包

```sql
GRANT CREATE PROCEDURE TO app_admin;
```

- 创建一个别名（为了方便）

```sql
GRANT CREATE SYNONYM TO app_admin;
```

---

<h4 id='向app_user和app_admin_user模式授予权限'>9.3.4 向app_user和app_admin_user模式授予权限</h4>

只向app_user和app_admin_user模式授予下面这些权限：

- 连接到Oracle数据库

```sql
GRANT CREATE SESSION TO app_user;
GRANT CREATE SESSION TO app_admin_user;
```

- 创建别名（为了方便）

```sql
GRANT CREATE SYNONYM TO app_user;
GRANT CREATE SYNONYM TO app_admin_user;
```

---

### 9.4 创建结构对象和加载数据

本节将展示如何为您的应用创建表、编辑视图、触发器以及序列，如何向表里载入数据，以及如何将这些模式对象上的权限授予需要这些权限的用户。

创建模式对象和加载数据需要：

1. 使用用户app_data连接到Oracle数据库。

  > <text class='lightblue'>参见</text>：
  > - “[通过SQL*Plus连接到Oracle数据库](一个指向不存在的地方的链接)”
  > - “[通过SQL Developer连接到Oracle数据库](一个指向不存在的地方的链接)”

2. 创建表格，并添加除了外键约束以外的所有约束，在导入数据之后再添加外键约束。

3. 创建编辑视图。

4. 创建触发器。

5. 创建序列。

6. 向表格导入数据。

7. 添加外键约束。

- [创建表格](#创建表格)

- [创建编辑视图](#创建编辑视图)

- [创建触发器](#创建触发器)

- [创建序列](#创建序列)

- [导入数据](#导入数据)

- [添加外键约束](#添加外键约束)

- [给模式对象的用户授予权限](#给模式对象的用户授予权限)

---

<h4 id='创建表格'>9.4.1 创建表格</h4>

本节展示了如何为应用创建表格，以及添加除了外键约束（需要导入数据之后才能添加）以外的所有约束。

> <text class='lightblue'>提示</text>：<br>
> 你必须已经通过用户app_data连接到Oracle数据库

在接下来的步骤中，你可以键入这些语句在SQL*Plus或者SQL Developer的Worksheet中。或者，你也可以使用SQL Developer的Create Table工具来创建表格

创建表格需要：

1. 创建用于储存关于在市场中职业信息的jobs#（每一行对应一个职业）：

```sql
CREATE TABLE jobs#
( job_id VARCHAR2(10)
CONSTRAINT jobs_pk PRIMARY KEY,
 job_title VARCHAR2(35)
CONSTRAINT jobs_job_title_not_null NOT NULL,
 min_salary NUMBER(6)
CONSTRAINT jobs_min_salary_not_null NOT NULL,
 max_salary NUMBER(6)
CONSTRAINT jobs_max_salary_not_null NOT NULL
)
/
```

2. 创建用于储存公司部门信息的departments#（每一行对应一个部门）：

```sql
CREATE TABLE departments#
( department_id NUMBER(4)
CONSTRAINT departments_pk PRIMARY KEY,
 department_name VARCHAR2(30)
CONSTRAINT department_name_not_null NOT NULL
CONSTRAINT department_name_unique UNIQUE,
 manager_id NUMBER(6)
)
/
```

3. 创建用于储存公司员工信息的employees#（每一行对应一个员工）：

```sql
CREATE TABLE employees#
( employee_id NUMBER(6)
CONSTRAINT employees_pk PRIMARY KEY,
 first_name VARCHAR2(20)
CONSTRAINT emp_first_name_not_null NOT NULL,
 last_name VARCHAR2(25)
CONSTRAINT emp_last_name_not_null NOT NULL,
 email_addr VARCHAR2(25)
CONSTRAINT emp_email_addr_not_null NOT NULL,
 hire_date DATE
DEFAULT TRUNC(SYSDATE)
CONSTRAINT emp_hire_date_not_null NOT NULL
CONSTRAINT emp_hire_date_check
CHECK(TRUNC(hire_date) = hire_date),
 country_code VARCHAR2(5)
CONSTRAINT emp_country_code_not_null NOT NULL,
 phone_number VARCHAR2(20)
CONSTRAINT emp_phone_number_not_null NOT NULL,
 job_id CONSTRAINT emp_job_id_not_null NOT NULL
CONSTRAINT emp_jobs_fk REFERENCES jobs#,
 job_start_date DATE
CONSTRAINT emp_job_start_date_not_null NOT NULL,
CONSTRAINT emp_job_start_date_check
CHECK(TRUNC(JOB_START_DATE) = job_start_date),
 salary NUMBER(6)
CONSTRAINT emp_salary_not_null NOT NULL,
 manager_id CONSTRAINT emp_mgr_to_empno_fk REFERENCES employees#,
 department_id CONSTRAINT emp_to_dept_fk REFERENCES departments#
)
/
```

这里加入REF约束的原因：

- 一个员工必须拥有一个存在的工作。所以，任意一个employees#.job_id都必须有对应的jobs#.job_id。
- 一个员工必须拥有一个同样是员工的经理。所以，任意一个employees#.manager_id都必须有对应的employees#.employee_id。
- 一个员工必须在一个存在的部门工作。所以，任意一个employees#.department_id都必须有对应的departments#.department_id。

此外，员工的经理必须是一个这个员工任职部门的经理。所以，任意一个employees#.manager_id都必须拥有对应的departments#.manager_id。但是，在创建departments#时你不能指定必要的约束，因为employees#还不存在。因此，你需要稍后再为departments#添加外键约束（参见[“添加外键约束”](一个指向不存在的地方的链接)）。

4. 创建job_history#，存储公司中每个员工的职位变动记录(每一行对应一个员工):

```sql
CREATE TABLE job_history#
( employee_id CONSTRAINT job_hist_to_employees_fk REFERENCES employees#,
 job_id CONSTRAINT job_hist_to_jobs_fk REFERENCES jobs#,
 start_date DATE
CONSTRAINT job_hist_start_date_not_null NOT NULL,
 end_date DATE
CONSTRAINT job_hist_end_date_not_null NOT NULL,
 department_id
CONSTRAINT job_hist_to_departments_fk REFERENCES departments#
CONSTRAINT job_hist_dept_id_not_null NOT NULL,
CONSTRAINT job_history_pk PRIMARY KEY(employee_id,start_date),
CONSTRAINT job_history_date_check CHECK( start_date < end_date )
)
/
```

添加REF约束是因为员工、岗位和部门必须存在。

- 任意的job_history#.employee_id都必须有对应的employees#.employee_id。
- 任意的job_history#.job_id都必须有对应的jobs#.job_id。
- 任意的job_history#.department_id都必须有对应的departments#.department_id。

> <text class='lightblue'>参见</text>：<br>
> [“创建表格”](一个指向不存在的地方的链接)

---

<h4 id='创建编辑视图'>9.4.2 创建编辑视图</h4>

> <text class='lightblue'>注意</text>：
> 你必须已经通过app_data用户连接到Oracle数据库。

为了创建一个可编辑视图，你需要使用下面的语句（任意顺序）。你可以键入这些语句在SQl*plus或者SQl Developer的Worksheet里面。或者，你可以使用SQl Developer的Create View工具创建可编辑视图。

```sql
CREATE OR REPLACE EDITIONING VIEW jobs AS SELECT * FROM jobs#
/
CREATE OR REPLACE EDITIONING VIEW departments AS SELECT * FROM departments#
/
CREATE OR REPLACE EDITIONING VIEW employees AS SELECT * FROM employees#
/
CREATE OR REPLACE EDITIONING VIEW job_history AS SELECT * FROM job_history#
/
```

> <text class='lightblue'>注意</text>：<br>
> 应用程序必须始终通过编辑视图引用基表。否则，可编辑视图不会覆盖表，并且您不能在使用完成的应用程序时使用EBR来升级它。

> <text class='lightblue'>参见</text>：
> - [“创建视图”](一个指向不存在的地方的链接)
> - Oracle数据库开发指南中关于可编辑视图的一般信息
> - Oracle数据库开发指南中关于初始化应用的一般信息

---

<h4 id='创建触发器'>9.4.3 创建触发器</h4>

> <text class='lightblue'>注意</text>：<br>
> 你必须已经通过app_data用户连接到Oracle数据库。

应用中的触发器遵循以下商业规则：

- 一个拥有工作j的职工，必须拥有一份在工作j工资最大值和最小值之间的工资。
- 如果一个拥有工作j的员工拥有工资s，则你不能改变j的最小工资变得比s大或者改变j的最大工资变得比s小。（这样做会使现有数据无效。）

- [创建执行第一个商业规则的触发器](#创建执行第一个商业规则的触发器)

  第一条商业原则是：一个拥有工作j的职工，必须拥有一份在工作j工资最大值和最小值之间的工资。

- [创建执行第二个商业规则的触发器](#创建执行第二个商业规则的触发器)

  第二条商业原则是：如果一个拥有工作j的员工拥有工资s，则你不能改变j的最小工资变得比s大或者改变j的最大工资变得比s小。（这样做会使现有数据无效。）

> <text class='lightblue'>参见</text>：<br>
> [使用触发器](一个指向不存在的地方的链接)中关于触发器的信息

<h5 id='创建执行第一个商业规则的触发器'>9.4.3.1 创建执行第一个商业规则的触发器</h5>

第一条商业原则是：一个拥有工作j的职工，必须拥有一份在工作j工资最大值和最小值之间的工资。

当向employees表插入新行或更新employees表的salary或job_id列时，都可能违反此规则。

要执行该规则，请在编辑视图员工上创建以下触发器。您可以在SQL*Plus或SQL Developer的Worksheet中输入CREATE TRIGGER语句。或者，您可以使用SQL Developer工具Create Trigger创建触发器。

```sql
CREATE OR REPLACE TRIGGER employees_aiufer
AFTER INSERT OR UPDATE OF salary, job_id ON employees FOR EACH ROW
DECLARE
 l_cnt NUMBER;
BEGIN
 LOCK TABLE jobs IN SHARE MODE;
     -- Ensure that jobs does not change
     -- during the following query.
 SELECT COUNT(*) INTO l_cnt
 FROM jobs
 WHERE job_id = :NEW.job_id
 AND :NEW.salary BETWEEN min_salary AND max_salary;
 IF (l_cnt<>1) THEN
 RAISE_APPLICATION_ERROR( -20002,
 CASE
 WHEN :new.job_id = :old.job_id
 THEN 'Salary modification invalid'
 ELSE 'Job reassignment puts salary out of range'
 END );
 END IF;
END;
/
```

LOCK TABLE jobs IN SHARE MODE防止其他用户在触发器查询时修改表作业。在查询期间防止对作业的更改是必要的，因为非阻塞读取会阻止触发器在更改员工时“看到”其他用户对作业所做的更改(并阻止那些用户“看到”触发器对员工所做的更改)。

防止在查询期间更改作业的另一种方法是在SELECT语句中包含FOR UPDATE子句。然而，SELECT FOR UPDATE对并发性的限制比LOCK TABLE job IN SHARE MODE更大。

LOCK TABLE jobs IN SHARE MODE可以阻止其他用户修改作业，但不能阻止自己锁定处于SHARE模式的作业。换职业可能比换员工少得多。因此，在共享模式下锁定作业比在独占模式下锁定单行作业提供更多的并发性。

> <text class='lightblue'>参见</text>：
> - Oracle数据库开发指南中关于在共享模式下锁定表格的信息
> - Oracle数据库PL/SQL与越南参考中关于SELECT FOR UPDATE的信息
> - [创建触发器](一个指向不存在的地方的链接)
> - [教程:展示employees_pkg子程序如何工作](一个指向不存在的地方的链接)，了解employees_aiufer触发器是如何工作的

<h5 id='创建执行第二个商业规则的触发器'>9.4.3.2 创建执行第二个商业规则的触发器</h5>

第二条商业原则是：如果一个拥有工作j的员工拥有工资s，则你不能改变j的最小工资变得比s大或者改变j的最大工资变得比s小。（这样做会使现有数据无效。）

当更新jobs表的min_salary或max_salary列时，可能会违反此规则。

要遵循该规则，请在编辑视图作业上创建以下触发器。您可以在SQL*Plus或SQL Developer的Worksheet中输入CREATE TRIGGER语句。或者，您可以使用SQL Developer工具Create Trigger创建触发器。

```sql
CREATE OR REPLACE TRIGGER jobs_aufer
AFTER UPDATE OF min_salary, max_salary ON jobs FOR EACH ROW
WHEN (NEW.min_salary > OLD.min_salary OR NEW.max_salary < OLD.max_salary)
DECLARE
 l_cnt NUMBER;
BEGIN
 LOCK TABLE employees IN SHARE MODE;
 SELECT COUNT(*) INTO l_cnt
 FROM employees
 WHERE job_id = :NEW.job_id
 AND salary NOT BETWEEN :NEW.min_salary and :NEW.max_salary;
 IF (l_cnt>0) THEN
Chapter 9
Creating the Schema Objects and Loading the Data
9-13
 RAISE_APPLICATION_ERROR( -20001,
 'Salary update would violate ' || l_cnt || ' existing employee records' );
 END IF;
END;
/
```

LOCK TABLE employees IN SHARE MODE防止其他用户在触发器查询时修改表员工。在查询期间防止对员工的更改是必要的，因为非阻塞读取会阻止触发器在更改作业时“看到”其他用户对员工所做的更改(并阻止那些用户“看到”触发器对作业所做的更改)。

对于这个触发器，SELECT FOR UPDATE不能替代LOCK TABLE IN SHARE MODE。当您尝试更改此工作的工资范围时，此触发器必须防止其他用户将工资更改到新范围之外。因此，触发器必须锁定employees表中具有此job_id的所有行，并锁定某人可以更新为具有此job_id的所有行。

LOCK TABLE employees IN SHARE MODE的一种替代方法是使用DBMS_LOCK包创建一个名为job_id的命名锁，然后在employees和jobs表上使用触发器来使用这个命名锁，以防止并发更新。但是，使用DBMS_LOCK和多个触发器会对运行时性能产生负面影响。

LOCK TABLE employees IN SHARE MODE的另一种替代方法是在employees表上创建一个触发器，对于每一行被更改的员工，该触发器锁定jobs中相应的作业行。但是，这种方法会导致频繁更新employees表的工作过多。

LOCK TABLE employees IN SHARE MODE比前面的选项简单，并且很少对job表进行更改，而且很可能发生在应用程序维护时，此时锁定表不会给用户带来不便。

> <text class='lightblue'>参见</text>：
> - Oracle数据库开发指南中关于在共享模式下锁定表格的信息
> - Oracle数据库PL/SQL与越南参考中关于 DBMS_LOCK包的信息
> - [创建触发器](一个指向不存在的地方的链接)
> - [教程:展示admin_pkg子程序如何工作](一个指向不存在的地方的链接)，了解employees_aiufer触发器是如何工作的

---

<h4 id='创建序列'>9.4.4 创建序列</h4>

> <text class='lightblue'>注意</text>：<br>
> 你必须已经通过app_data用户连接到Oracle数据库。

要创建为新部门和新员工生成唯一主键的序列，请使用以下语句(按任意顺序)。您可以在SQL*Plus或SQL Developer的Worksheet中输入语句。或者，您可以使用SQL Developer工具Create Sequence创建序列。

```sql
CREATE SEQUENCE employees_sequence START WITH 210;
CREATE SEQUENCE departments_sequence START WITH 275;
```

为了避免与您将从样例模式HR中的表加载的数据冲突，employees_sequence和departments_sequence的起始数字必须分别超过employees.employee_id和departments.department_id的最大值。在“加载数据”之后，这个查询显示了这些最大值:

```sql
SELECT MAX(e.employee_id), MAX(d.department_id)
FROM employees e, departments d;
```

结果：


| MAX(E.EMPLOYEE_ID) | MAX(D.DEPARTMENT_ID) |
| - | - |
| 206 | 270 |


> <text class='lightblue'>参见</text>：<br>
> “[创建和管理序列](一个指向不存在的地方的链接)”

---

<h4 id='导入数据'>9.4.5 导入数据</h4>

> <text class='lightblue'>注意</text>：<br>
> 你必须已经通过app_data用户连接到Oracle数据库。

用样例模式HR中的表中的数据加载应用程序的表。

> <text class='lightblue'>注意</text>：<br>
> 下面的过程通过它们的编辑视图引用应用程序的表。

在下面的过程中，您可以在SQL*Plus或SQL Developer的Worksheet中输入语句。

往表格里加载数据:

1. 使用表HR中的数据加载HR.JOBS:

```sql
INSERT INTO jobs (job_id, job_title, min_salary, max_salary)
SELECT job_id, job_title, min_salary, max_salary
 FROM HR.JOBS
/
```

结果：

```
19 rows created.
```

2. 向部门加载来自HR表的HR.DEPARTMENTS:

```sql
INSERT INTO departments (department_id, department_name, manager_id)
SELECT department_id, department_name, manager_id
  FROM HR.DEPARTMENTS
/
```

结果：

```
27 rows created
```

3. 从HR.employees和HR.JOB_HISTORY表中加载员工数据，使用搜索的CASE表达式和SQL函数从HR$phone_number中获取employees.country_code和employees.phone_number，使用SQL函数和标量子查询从HR.JOB_HISTORY中获取employees.job_start_date:

```sql
 INSERT INTO employees (employee_id, first_name, last_name, email_addr,
  hire_date, country_code, phone_number, job_id, job_start_date, salary,
  manager_id, department_id)
 SELECT employee_id, first_name, last_name, email, hire_date,
  CASE WHEN phone_number LIKE '011.%'
    THEN '+' || SUBSTR( phone_number, INSTR( phone_number, '.' )+1,
      INSTR( phone_number, '.', 1, 2 ) -  INSTR( phone_number, '.' ) - 1 )
    ELSE '+1'
  END country_code,
  CASE WHEN phone_number LIKE '011.%'
    THEN SUBSTR( phone_number, INSTR(phone_number, '.', 1, 2 )+1 )
    ELSE phone_number
  END phone_number,
  job_id,
  NVL( (SELECT MAX(end_date+1)
        FROM HR.JOB_HISTORY jh
        WHERE jh.employee_id = employees.employee_id), hire_date),
  salary, manager_id, department_id  
  FROM HR.EMPLOYEES
 /
```

结果：

```
 107 rows created.
```

4. 从表HR.HOB_HISTORY加载Job_history数据：

```sql
 INSERT INTO job_history (employee_id, job_id, start_date, end_date,
  department_id)
 Creating the Schema Objects and Loading the Data
 SELECT employee_id, job_id, start_date, end_date, department_id
  FROM HR.JOB_HISTORY
 /
```

结果：

```
 10 rows created.
```

5. 提交变化：

```sql
COMMIT;
```

> <text class='lightblue'>参见</text>：<br>
> - “[关于INSERT语句](一个指向不存在的地方的链接)”
> - “[关于简单模式HR](一个指向不存在的地方的链接)”
> - “[在查询时使用CASE表达式](一个指向不存在的地方的链接)”
> - “[在查询时使用NULL-Realated函数](一个指向不存在的地方的链接)”中关于NVL函数的信息
> - *Oracle数据库SQL语言*参照中关于SQL函数的信息

---

<h4 id='添加外键约束'>9.4.6 添加外键约束</h4>

> <text class='lightblue'>注意</text>：<br>
> 你必须已经通过app_data用户连接到Oracle数据库。

现在departments和employees表包含了数据，所以可以使用下面的ALTER TABLE语句添加一个外键约束。您可以在SQL*Plus或SQL Developer的Worksheet 中输入语句。或者，您可以使用SQL Developer工具Add Foreign Key添加约束。

```sql
 ALTER TABLE departments#
 ADD CONSTRAINT dept_to_emp_fk
 FOREIGN KEY(manager_id) REFERENCES employees#;
```

如果在departments#和employees#包含数据之前添加了这个外键约束，那么在尝试向它们中的任何一个加载数据时都会得到这个错误:

```
ORA-02291: integrity constraint (APP_DATA.JOB_HIST_TO_DEPT_FK) violated - parent key not found
```

> <text class='lightblue'>参见</text>：<br>
> “[教程:向现有表添加约束](一个指向不存在的地方的链接)”

---

<h4 id='给模式对象的用户授予权限'>9.4.7 给模式对象的用户授予权限</h4>

> <text class='lightblue'>注意</text>：<br>
> 你必须已经通过app_data用户连接到Oracle数据库。

要向用户授予权限，请使用SQL语句Grant。GRANT语句可以在SQL*Plus中输入，也可以在SQL Developer的Worksheet中输入。

只授予app_code创建employees_pkg所需的权限：

```sql
 GRANT SELECT, INSERT, UPDATE, DELETE ON employees TO app_code;
 GRANT SELECT ON departments TO app_code;
 GRANT SELECT ON jobs TO app_code;
 GRANT SELECT, INSERT on job_history TO app_code;
 GRANT SELECT ON employees_sequence TO app_code;
```

只授予app_admin创建admin_pkg所需的权限：

```sql
 GRANT SELECT, INSERT, UPDATE, DELETE ON jobs TO app_admin;
 GRANT SELECT, INSERT, UPDATE, DELETE ON departments TO app_admin;
 GRANT SELECT ON employees_sequence TO app_admin;
 GRANT SELECT ON departments_sequence TO app_admin;
```

> <text class='lightblue'>参见</text>：<br>
> [Oracle数据库SQL语言](一个指向不存在的地方的链接)中关于GRANT的信息

---

### 9.5 创建employees_pkg包

本节将展示如何创建一个employees_pkg包，如何使其子程序运行，如何将包的执行权限授予需要的用户，以及这些用户如何调用它的子程序。

为了创建employees_pkg包：

1. 使用app_data连接到Oracle数据库

    有关说明，请参见[“从SQL*Plus连接到Oracle数据库”](一个指向不存在的地方的链接)或[“从SQL Developer连接到Oracle数据库”](一个指向不存在的地方的链接)。

2. 创建下列别名：

    ```sql
    CREATE OR REPLACE SYNONYM employees FOR app_data.employees;
    CREATE OR REPLACE SYNONYM departments FOR app_data.departments;
    CREATE OR REPLACE SYNONYM jobs FOR app_data.jobs;
    CREATE OR REPLACE SYNONYM job_history FOR app_data.job_history;
    ```

    您可以在SQL*Plus或SQL Developer的Worksheet中输入CREATE SYNONYM语句。或者，您可以使用SQL Developer工具Create Synonym创建同义词。

3. 创建包规范

4. 创建包主体

- [为了employees_pkg创建包规范](#为了employees_pkg创建包规范)

- [为了employees_pkg创建包主体](#为了employees_pkg创建包主体)

- [教程：展示employees_pkg的子程序是如何工作的](#教程：展示employees_pkg的子程序是如何工作的)

  本教程将使用SQL*Plus展示employees_pkg包的子程序是如何工作的。本教程还展示了触发器employees_aiufer和CHECK约束job_history_date_check是如何工作的。

- [向app_user和app_admin_user授予执行权限](#向app_user和app_admin_user授予执行权限)

- 教程：作为app_user或app_admin_user调用get_job_history

  本教程将使用SQL*Plus演示如何以用户app_user(通常是管理员)或app_admin_user(应用程序管理员)的身份调用子程序app_code.employees_pkg.get_job_history。

> <text class='lightblue'>参见</text>：<br>
> - “[创建别名](一个指向不存在的地方的链接)”
> - “[关于包](一个指向不存在的地方的链接)”

---

<h4 id='为了employees_pkg创建包规范'>9.5.1 为了employees_pkg创建包规范</h4>

> <text class='lightblue'>注意</text>：<br>
> 你必须已经通过app_code用户连接到Oracle数据库。

要为employees_pkg(管理器的API)创建包规范，请使用以下CREATE PACKGAE语句。您可以在SQL*Plus或SQL Developer的Worksheet 中输入语句。或者，您可以使用SQL Developer工具Create Package创建包。

```sql
 CREATE OR REPLACE PACKAGE employees_pkg
 AS
  PROCEDURE get_employees_in_dept
    ( p_deptno     IN     employees.department_id%TYPE,
      p_result_set IN OUT SYS_REFCURSOR );
  PROCEDURE get_job_history
    ( p_employee_id  IN     employees.department_id%TYPE,
      p_result_set   IN OUT SYS_REFCURSOR );
  PROCEDURE show_employee
    ( p_employee_id  IN     employees.employee_id%TYPE,
      p_result_set   IN OUT SYS_REFCURSOR );
      PROCEDURE update_salary
    ( p_employee_id IN employees.employee_id%TYPE,
      p_new_salary  IN employees.salary%TYPE );
  PROCEDURE change_job
    ( p_employee_id IN employees.employee_id%TYPE,
      p_new_job     IN employees.job_id%TYPE,
      p_new_salary  IN employees.salary%TYPE := NULL,
      p_new_dept    IN employees.department_id%TYPE := NULL );
 END employees_pkg;
 /
```

> <text class='lightblue'>参见</text>：<br>
> - “[关于这个应用](一个指向不存在的地方的链接)”
> - “[创建和管理包](一个指向不存在的地方的链接)”
> - [Oracle数据库PL/SQL语言参照](一个指向不存在的地方的链接)中关于CREATE PACKAGE语句的信息

---

<h4 id='为了employees_pkg创建包主体'>9.5.2 为了employees_pkg创建包主体</h4>

> <text class='lightblue'>注意</text>：<br>
> 你必须已经通过app_code用户连接到Oracle数据库。

要为employees_pkg(管理器的API)创建包体，请使用以下CREATE PACKAGE BODY语句。您可以在SQL*Plus或SQL Developer的Worksheet中输入语句。或者，您可以使用SQL Developer工具Create Body创建包。

```sql
 CREATE OR REPLACE PACKAGE BODY employees_pkg
 AS
  PROCEDURE get_employees_in_dept
    ( p_deptno     IN     employees.department_id%TYPE,
      p_result_set IN OUT SYS_REFCURSOR )
  IS
     l_cursor SYS_REFCURSOR;
  BEGIN
    OPEN p_result_set FOR
      SELECT e.employee_id,
        e.first_name || ' ' || e.last_name name,
        TO_CHAR( e.hire_date, 'Dy Mon ddth, yyyy' ) hire_date,
        j.job_title,
        m.first_name || ' ' || m.last_name manager,
        d.department_name
      FROM employees e INNER JOIN jobs j ON (e.job_id = j.job_id)
        LEFT OUTER JOIN employees m ON (e.manager_id = m.employee_id)
        INNER JOIN departments d ON (e.department_id = d.department_id)
           WHERE e.department_id = p_deptno ;
  END get_employees_in_dept;
  PROCEDURE get_job_history
    ( p_employee_id  IN     employees.department_id%TYPE,
      p_result_set   IN OUT SYS_REFCURSOR )
  IS 
  BEGIN
    OPEN p_result_set FOR
      SELECT e.First_name || ' ' || e.last_name name, j.job_title,
        e.job_start_date start_date,
        TO_DATE(NULL) end_date
      FROM employees e INNER JOIN jobs j ON (e.job_id = j.job_id)
      WHERE e.employee_id = p_employee_id
      UNION ALL
      SELECT e.First_name || ' ' || e.last_name name,
        j.job_title,
        jh.start_date,
        jh.end_date
      FROM employees e INNER JOIN job_history jh
        ON (e.employee_id = jh.employee_id)
        INNER JOIN jobs j ON (jh.job_id = j.job_id)
      WHERE e.employee_id = p_employee_id
      ORDER BY start_date DESC;
  END get_job_history;
  PROCEDURE show_employee
    ( p_employee_id  IN     employees.employee_id%TYPE,
      p_result_set   IN OUT sys_refcursor )
  IS 
  BEGIN
    OPEN p_result_set FOR
      SELECT *
      FROM (SELECT TO_CHAR(e.employee_id) employee_id,
              e.first_name || ' ' || e.last_name name,
              e.email_addr,
              TO_CHAR(e.hire_date,'dd-mon-yyyy') hire_date,
              e.country_code,
              e.phone_number,
              j.job_title,
              TO_CHAR(e.job_start_date,'dd-mon-yyyy') job_start_date,
              to_char(e.salary) salary,
              m.first_name || ' ' || m.last_name manager,
              d.department_name
            FROM employees e INNER JOIN jobs j on (e.job_id = j.job_id)
              RIGHT OUTER JOIN employees m ON (m.employee_id = e.manager_id)
              INNER JOIN departments d ON (e.department_id = d.department_id)
            WHERE e.employee_id = p_employee_id)
      UNPIVOT (VALUE FOR ATTRIBUTE IN (employee_id, name, email_addr, hire_date,
        country_code, phone_number, job_title, job_start_date, salary, manager,
        department_name) );
  END show_employee;
  PROCEDURE update_salary
    ( p_employee_id IN employees.employee_id%type,
      p_new_salary  IN employees.salary%type )
  IS 
  BEGIN
    UPDATE employees
    SET salary = p_new_salary
    WHERE employee_id = p_employee_id;
     END update_salary;
  PROCEDURE change_job
    ( p_employee_id IN employees.employee_id%TYPE,
      p_new_job     IN employees.job_id%TYPE,
      p_new_salary  IN employees.salary%TYPE := NULL,
      p_new_dept    IN employees.department_id%TYPE := NULL )
  IS
  BEGIN
    INSERT INTO job_history (employee_id, start_date, end_date, job_id,
      department_id)
    SELECT employee_id, job_start_date, TRUNC(SYSDATE), job_id, department_id
      FROM employees
      WHERE employee_id = p_employee_id;
    UPDATE employees
    SET job_id = p_new_job,
      department_id = NVL( p_new_dept, department_id ),
      salary = NVL( p_new_salary, salary ),
      job_start_date = TRUNC(SYSDATE)
    WHERE employee_id = p_employee_id;
  END change_job;
 END employees_pkg;
 /
```

> <text class='lightblue'>参见</text>：<br>
> - “[关于这个应用](一个指向不存在的地方的链接)”
> - “[创建和管理包](一个指向不存在的地方的链接)”
> - [Oracle数据库PL/SQL语言参照](一个指向不存在的地方的链接)中关于CREATE PACKAGE BODY语句的信息

---

<h4 id='教程：展示employees_pkg的子程序是如何工作的'>9.5.3 教程：展示employees_pkg的子程序是如何工作的</h4>

本教程将使用SQL*Plus展示employees_pkg包的子程序是如何工作的。本教程还展示了触发器employees_aiufer和CHECK约束job_history_date_check是如何工作的。

> <text class='lightblue'>注意</text>：<br>
> 你必须已经通过app_code用户连接到Oracle数据库。

为了使用SQL*Plus展示employees_pkg子程序是如何工作的：

1. 使用格式化命令可以提高输出的可读性。例如：

```sql
 SET LINESIZE 80
 SET RECSEP WRAPPED
 SET RECSEPCHAR "="
 COLUMN NAME FORMAT A15 WORD_WRAPPED
  COLUMN HIRE_DATE FORMAT A20 WORD_WRAPPED
 COLUMN DEPARTMENT_NAME FORMAT A10 WORD_WRAPPED
 COLUMN JOB_TITLE FORMAT A29 WORD_WRAPPED
 COLUMN MANAGER FORMAT A11 WORD_WRAPPED
```

2. 为子程序参数的值声明绑定变量

```
 p_result_set:
 VARIABLE c REFCURSOR
```

3. 展示部门90中的所有员工

```sql
 EXEC employees_pkg.get_employees_in_dept( 90, :c );
 PRINT c
```

结果：

| EMPLOYEE_ID NAME | HIRE_DATE | JOB_TITLE | MANAGER | DEPARTMENT |
|-|-|-|-|-|                                                  
| 100 | Steven King | Tue Jun 17th, 2003 | President | Executive |                                                         
| 102 | Lex De Haan | Sat Jan 13th, 2001 | Administration Vice President | Steven King Executive |
| 101 | Neena Kochhar | Wed Sep 21st, 2005 | Administration Vice President | Steven King Executive |                                

4. 显示员工101的工作经历

```sql
 EXEC employees_pkg.get_job_history( 101, :c );
 PRINT c
```

结果：

| ME | JOB_TITLE | START_DAT | END_DATE |
|-|-|-|-|
| Neena | Kochhar | Administration Vice President | 16-MAR-05 |
| Neena | Kochhar | Accounting Manager | 28-OCT-01 15-MAR-05 |
| Neena | Kochhar | Public Accountant | 21-SEP-97 27-OCT-01 |
---

5. 显示员工101的详细信息：

```sql
 EXEC employees_pkg.show_employee( 101, :c );
 PRINT c
```

结果：

| ATTRIBUTE | VALUE |
|-|-|
| EMPLOYEE_ID | 101 |
| NAME | Neena Kochhar |
| EMAIL_ADDR | NKOCHHAR |
| HIRE_DATE | 21-sep-2005 |
| COUNTRY_CODE | +1 |
| PHONE_NUMBER | 515.123.4568 |
| JOB_TITLE | Administration Vice President |
| JOB_START_DATE | 16-mar-05 |
| SALARY | 17000 |
| MANAGER | Steven King |
| DEPARTMENT_NAME | Executive |

6. 显示工作Administration Vice President的所有信息：

```sql
SELECT * FROM jobs WHERE job_title = 'Administration Vice President';
```

结果：

| JOB_ID | JOB_TITLE | MIN_SALARY | MAX_SALARY |
|-|-|-|-|
| AD_VP | Administration Vice President | 15000 | 30000 |

7. 试着给101号员工一个超出她工作范围的新工资：

```sql
EXEC employees_pkg.update_salary( 101, 30001 )；
```

结果：

```
 SQL> EXEC employees_pkg.update_salary( 101, 30001 );
 BEGIN employees_pkg.update_salary( 101, 30001 ); END;
 *
 ERROR at line 1:
 ORA-20002: Salary modification invalid
 ORA-06512: at "APP_DATA.EMPLOYEES_AIUFER", line 13
 ORA-04088: error during execution of trigger 'APP_DATA.EMPLOYEES_AIUFER'
 ORA-06512: at "APP_CODE.EMPLOYEES_PKG", line 77
 ORA-06512: at line 1
```

8. 给员工101一个在工作范围内的新工资，并再次显示她的一般信息：

```sql
 EXEC employees_pkg.update_salary( 101, 18000 );
 EXEC employees_pkg.show_employee( 101, :c );
 PRINT c
```

结果：

| ATTRIBUTE | VALUE |
|-|-|
| EMPLOYEE_ID | 101 |
| NAME | Neena Kochhar |
| EMAIL_ADDR | NKOCHHAR |
| HIRE_DATE | 21-sep-2005 |
| COUNTRY_CODE | +1 |
| PHONE_NUMBER | 515.123.4568 |
| JOB_TITLE | Administration Vice President |
| JOB_START_DATE | 16-mar-05 |
| SALARY | 18000 |
| MANAGER | Steven King |
| DEPARTMENT_NAME | Executive |

```
11 rows selected.
```

9. 将员工101的工作改为薪水更低的同样工作：

```sql
EXEC employees_pkg.change_job( 101, 'AD_VP', 17500, 90 );
```

结果：

```
 SQL> exec employees_pkg.change_job( 101, 'AD_VP', 17500, 90 );
 BEGIN employees_pkg.change_job( 101, 'AD_VP', 17500, 80 ); END;
 9-24
Chapter 9
 Creating the employees_pkg Package
 *
 ERROR at line 1:
 ORA-02290: check constraint (APP_DATA.JOB_HISTORY_DATE_CHECK) violated
 ORA-06512: at "APP_CODE.EMPLOYEES_PKG", line 101 
 ORA-06512: at line 1
```

10. 显示员工的信息。(注意，前面的语句并没有改变工资;是18000，不是17500。)

```sql
 exec employees_pkg.show_employee( 101, :c );
 print c
```

结果：

| ATTRIBUTE | VALUE |
|-|-|
| EMPLOYEE_ID | 101 |
| NAME | Neena Kochhar |
| EMAIL_ADDR | NKOCHHAR |
| HIRE_DATE | 21-sep-2005 |
| COUNTRY_CODE | +1 |
| PHONE_NUMBER | 515.123.4568 |
| JOB_TITLE | Administration Vice President |
| JOB_START_DATE | 10-mar-2015 |
| SALARY | 18000 |
| MANAGER | Steven King |
| DEPARTMENT_NAME | Executive |

```
 11 rows selected.
```

> <text class='lightblue'>参见</text>：<br>
> - [SQL*Plus用户指引和参照](一个指向不存在的地方的链接)中关于SQL*Plus指令的信息
> - “[创建和管理包](一个指向不存在的地方的链接)”

---

<h4 id='向app_user和app_admin_user授予执行权限'>9.5.4 向app_user和app_admin_user授予执行权限</h4>

> <text class='lightblue'>注意</text>：<br>
> 你必须已经通过app_code用户连接到Oracle数据库。

要将包employees_pkg上的执行权限授予app_user(通常是管理员)和app_admin_user(应用程序管理员)，请使用以下grant语句(按任意顺序)。您可以在SQL*Plus或SQL Developer的工作表中输入语句。

```sql
 GRANT EXECUTE ON employees_pkg TO app_user;
 GRANT EXECUTE ON employees_pkg TO app_admin_user;
```

> <text class='lightblue'>参见</text>：<br>
> - “[应用中的模式](一个指向不存在的地方的链接)”
> - [Oracle数据库SQL语言参照](一个指向不存在的地方的链接)中关于GRANT语句的信息

## End

### Thanks

- [有道翻译](https://fanyi.youdao.com/#/)
- [文档原文地址](https://docs.oracle.com/en/database/oracle/oracle-database/19/tdddg/developing-simple-database-application.html#GUID-98FFDC3A-FD76-4466-921C-F0E411829CD7)


---
<style>
  .lightblue{
    color : lightblue
  }
  .yellow{
    color : yellow
  }
</style>