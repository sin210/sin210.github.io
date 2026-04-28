### 一、前置准备：连接 Oracle 数据库
Oracle 连接方式主要有两种（以 Windows/Linux 通用的 `sqlplus` 为例），先确保环境已配置好 Oracle 客户端。

#### 1. 本地连接（数据库装在本机）
```sql
-- 方式1：使用系统用户（sys/ system）连接，as sysdba 表示以管理员身份
sqlplus sys/你的密码 as sysdba

-- 方式2：普通用户连接（需先创建用户）
sqlplus 用户名/密码
```

#### 2. 远程连接（指定 IP+端口+服务名）
```sql
-- 格式：sqlplus 用户名/密码@IP:端口/服务名
sqlplus scott/tiger@192.168.1.100:1521/orcl
```

#### 3. 常用退出/切换命令
```sql
exit; -- 退出 sqlplus
conn 用户名/密码; -- 切换用户（无需退出）
```

### 二、核心操作1：用户与权限管理（运维常用）
Oracle 严格区分用户和权限，运维中常需要创建用户、分配权限、修改密码。

#### 1. 创建用户（需管理员权限）
```sql
-- 创建用户，指定密码和表空间（默认表空间 USERS）
CREATE USER test_user IDENTIFIED BY 123456 
DEFAULT TABLESPACE USERS; -- 指定默认表空间
```

#### 2. 分配权限（关键）
```sql
-- 给普通用户分配最常用的权限（连接数据库+创建表）
GRANT CONNECT, RESOURCE TO test_user;

-- 分配查询/修改其他用户表的权限（按需）
GRANT SELECT ON scott.emp TO test_user; -- 允许查询 scott 的 emp 表
GRANT UPDATE ON scott.emp TO test_user; -- 允许修改 scott 的 emp 表

-- 回收权限
REVOKE UPDATE ON scott.emp FROM test_user;
```

#### 3. 修改用户密码/解锁用户
```sql
-- 修改密码
ALTER USER test_user IDENTIFIED BY 654321;

-- 解锁用户（比如 scott 用户默认锁定）
ALTER USER scott ACCOUNT UNLOCK;
```

### 三、核心操作2：表操作（建表/修改/删除）
#### 1. 创建表（开发/运维都常用）
以创建「员工表」为例，指定字段、类型、主键：
```sql
CREATE TABLE emp (
    empno NUMBER(4) PRIMARY KEY, -- 员工编号（主键，4位数字）
    ename VARCHAR2(20), -- 姓名（可变字符串，最多20字符）
    job VARCHAR2(10), -- 职位
    sal NUMBER(7,2), -- 工资（7位数字，2位小数）
    hiredate DATE -- 入职日期
);
```

#### 2. 修改表结构
```sql
-- 添加字段
ALTER TABLE emp ADD (deptno NUMBER(2)); -- 新增部门编号字段

-- 修改字段类型（注意：字段无数据时才能改）
ALTER TABLE emp MODIFY (ename VARCHAR2(30)); -- 姓名改为30字符

-- 删除字段
ALTER TABLE emp DROP COLUMN deptno;

-- 重命名表
RENAME emp TO employee;
```

#### 3. 删除表/清空表
```sql
-- 删除表（彻底删除，包括结构和数据）
DROP TABLE emp;

-- 清空表数据（两种方式）
TRUNCATE TABLE emp; -- 快速清空，不可回滚（运维常用，效率高）
DELETE FROM emp; -- 逐行删除，可回滚（开发常用）
```

### 四、核心操作3：数据增删改查（DML）
这是最基础的业务操作，和 MySQL 类似，但语法有细微差异。

#### 1. 插入数据（INSERT）
```sql
-- 插入完整数据（字段顺序和建表一致）
INSERT INTO emp VALUES (7369, 'SMITH', 'CLERK', 800, TO_DATE('1980-12-17', 'YYYY-MM-DD'));

-- 插入指定字段数据（推荐，字段顺序可自定义）
INSERT INTO emp (empno, ename, sal) VALUES (7499, 'ALLEN', 1600);
```

#### 2. 查询数据（SELECT，最常用）
```sql
-- 查询所有字段
SELECT * FROM emp;

-- 查询指定字段，加条件
SELECT ename, sal FROM emp WHERE sal > 1000;

-- 排序查询（升序 ASC，降序 DESC）
SELECT ename, sal FROM emp ORDER BY sal DESC;

-- 模糊查询（Oracle 用 LIKE + %）
SELECT * FROM emp WHERE ename LIKE 'S%'; -- 查姓名以 S 开头的员工

-- 统计查询（计数/求和/平均值）
SELECT COUNT(*) FROM emp; -- 总员工数
SELECT SUM(sal) FROM emp; -- 工资总和
SELECT AVG(sal) FROM emp; -- 平均工资
```

#### 3. 修改数据（UPDATE）
```sql
-- 修改指定数据（必须加 WHERE，否则改全表！）
UPDATE emp SET sal = 900 WHERE empno = 7369;
```

#### 4. 删除数据（DELETE）
```sql
-- 删除指定数据（加 WHERE，否则删全表）
DELETE FROM emp WHERE empno = 7369;
```

### 五、常用辅助操作
#### 1. 提交/回滚事务
Oracle 默认开启事务，增删改后需手动提交才生效：
```sql
COMMIT; -- 提交事务（确认修改）
ROLLBACK; -- 回滚事务（撤销未提交的修改）
```

#### 2. 查看表结构（运维排障常用）
```sql
-- 查看表的字段、类型、长度等信息
DESC emp; -- 简写，等价于 DESCRIBE emp;
```

#### 3. 查看表空间（运维核心）
```sql
-- 查看所有表空间（管理员权限）
SELECT tablespace_name FROM dba_tablespaces;

-- 查看用户所属表空间
SELECT username, default_tablespace FROM dba_users WHERE username = 'TEST_USER';
```

### 总结
1. Oracle 核心操作先掌握**连接方式**和**用户权限管理**（运维重点），再学表操作和数据增删改查（开发重点）；
2. 增删改操作后必须 `COMMIT` 才生效，`UPDATE/DELETE` 务必加 `WHERE` 条件，避免误操作；
3. 常用辅助命令：`DESC 表名`（查结构）、`COMMIT/ROLLBACK`（事务）、`conn`（切换用户）是高频操作。