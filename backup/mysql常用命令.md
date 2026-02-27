### 一、连接与基础操作（最常用）
#### 1. 登录MySQL
```bash
# 本地登录（默认端口3306）
mysql -u 用户名 -p
# 示例：mysql -u root -p（回车后输密码，密码不显示）

# 远程登录（指定IP+端口）
mysql -h 192.168.1.100 -P 3306 -u root -p
```
- 口述：登录用mysql -u用户名 -p，远程加-h指定IP、-P指定端口，密码回车后输入更安全。

#### 2. 查看版本/退出
```bash
# 查看版本
select version();

# 退出MySQL
exit;  # 或 quit;
```

### 二、数据库操作（核心）
#### 1. 查看所有数据库
```sql
show databases;
```

#### 2. 创建/删除/切换数据库
```sql
# 创建（指定字符集，避免中文乱码）
create database 库名 default charset utf8mb4;
# 示例：create database test_db default charset utf8mb4;

# 删除（谨慎！不可逆）
drop database 库名;

# 切换到目标数据库（必做：操作表前先选库）
use 库名;
```
- 口述：建库一定要加utf8mb4字符集，支持emoji和所有中文；删库前务必确认，删了就找不回了。

#### 3. 查看当前所在数据库
```sql
select database();
```

### 三、数据表操作
#### 1. 查看当前库的所有表
```sql
show tables;
```

#### 2. 查看表结构（面试高频）
```sql
desc 表名;  # 简写
# 或详细版：show create table 表名;
```
- 口述：看表结构用desc表名，能快速知道字段名、类型、是否主键、是否为空。

#### 3. 创建/删除表（示例）
```sql
# 创建表（简单示例：用户表）
create table user (
  id int primary key auto_increment,  # 主键自增
  name varchar(50) not null,          # 非空字符串
  age int default 0,                  # 默认值0
  create_time datetime                # 时间字段
) default charset utf8mb4;

# 删除表（谨慎）
drop table 表名;

# 清空表数据（保留表结构）
truncate table 表名;  # 比delete快，不可回滚
```

### 四、数据操作（增删改查）
#### 1. 查询（最核心，面试必问）
```sql
# 查所有数据
select * from 表名;

# 查指定字段+条件+排序+分页（高频）
select id,name from 表名 where age > 18 order by create_time desc limit 0,10;
# 解释：查age>18的id和name，按创建时间倒序，取前10条（分页：limit 起始行, 行数）

# 去重查询
select distinct 字段名 from 表名;

# 统计总数
select count(*) from 表名 where 条件;
```

#### 2. 新增数据
```sql
# 全字段插入
insert into 表名 values (1, '张三', 20, '2026-02-27');

# 指定字段插入（推荐，字段变动不影响）
insert into 表名(name, age) values ('李四', 22);
```

#### 3. 修改数据（必加where，否则改全表）
```sql
update 表名 set age=23 where id=1;
```

#### 4. 删除数据（必加where，否则删全表）
```sql
delete from 表名 where id=1;
```
- 口述：改和删一定要加where条件，不然会把整张表的数据改了/删了，生产环境绝对要注意！

### 五、权限与用户操作（运维常用）
```sql
# 创建用户（允许远程访问）
create user 'test_user'@'%' identified by '123456';

# 授权（给test_user分配test_db所有权限）
grant all privileges on test_db.* to 'test_user'@'%';

# 刷新权限（授权后必须执行）
flush privileges;

# 查看用户权限
show grants for 'test_user'@'%';

# 修改密码
alter user 'root'@'localhost' identified by 'new_password';
```

### 六、备份与恢复（运维高频）
```bash
# 备份整个数据库（终端执行，不是MySQL里）
mysqldump -u root -p 库名 > 备份文件名.sql
# 示例：mysqldump -u root -p test_db > test_db_backup.sql

# 恢复数据库（先建空库，再导入）
mysql -u root -p 新库名 < 备份文件名.sql
```

### 七、排障常用命令
```sql
# 查看慢查询日志（找性能问题）
show variables like '%slow_query%';

# 查看当前连接数（排查连接过多）
show processlist;

# 查看锁表情况
show open tables where in_use > 0;
```

### 总结
1. 核心高频命令：登录（mysql -u -p）、查库（show databases）、选库（use）、查表结构（desc）、增删改查（insert/update/delete/select）；
2. 关键注意点：删/改数据必加where、建库/表指定utf8mb4字符集、授权后要flush privileges；
3. 运维常用：备份（mysqldump）、查看连接数（show processlist）、用户权限管理。