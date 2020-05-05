# postgres入门


最近需要将mysql数据库切换到pg数据库（公司要求新上的应用DB首选postgres），所以对pg进行基本学习了下，总体感觉相差不大，在一些细节以及需要上可能需要注意。

## 安装
我这里是以docker的方式来进行安装的。
> 这种安装方式只能作为作为本地开发调试用

```bash
docker run --name postgres -v /Users/silenceper/workspace/postgres/data:/var/lib/postgresql/data -e POSTGRES_PASSWORD=123 -p 5432:5432 -d postgres:10.3
```

- 指定数据存储目录映射到宿主机本地，避免容器销毁后数据丢失
- 指定密码postgres用户的密码为123
- 指定使用postgres:10.3镜像

可以选择pgAdmin4作为客户端，有图形化UI界面，或者直接使用psql命令。

## 基本操作

登录数据库（使用psql命令）：

```
# 进入容器才能使用psql命令
docker exec -it postgres bash 
# 以postgres角色登录
 psql -U postgres
```

可以通过 `\?`查看一些常用的操作。

```
\h：查看SQL命令的解释，比如\h select。
\?：查看psql命令列表。
\l：列出所有数据库。
\c [database_name]：连接其他数据库。
\d：列出当前数据库的所有表格。
\d [table_name]：列出某一张表格的结构。
\du：列出所有用户。
\e：打开文本编辑器。
\conninfo：列出当前数据库和连接的信息。
```

### 创建用户以及DB
创建用户
```
CREATE USER dbuser WITH PASSWORD 'password';
```
创建用户数据库，这里为exampledb，并指定所有者为dbuser
```
CREATE DATABASE exampledb OWNER dbuser;
```

将exampledb数据库的所有权限都赋予dbuser，否则dbuser只能登录控制台，没有任何数据库操作权限。

```
GRANT ALL PRIVILEGES ON DATABASE exampledb to dbuser;
```

最后，使用\q命令退出控制台（也可以直接按ctrl+D）。

```
\q

```

### 数据库操作

```
# 创建新表
CREATE TABLE user_tbl(name VARCHAR(20), signup_date DATE);

# 插入数据
INSERT INTO user_tbl(name, signup_date) VALUES('张三', '2013-12-22');

# 选择记录
SELECT * FROM user_tbl;

# 更新数据
UPDATE user_tbl set name = '李四' WHERE name = '张三';

# 删除记录
DELETE FROM user_tbl WHERE name = '李四' ;

# 添加栏位
ALTER TABLE user_tbl ADD email VARCHAR(40);

# 更新结构
ALTER TABLE user_tbl ALTER COLUMN signup_date SET NOT NULL;

# 更名栏位
ALTER TABLE user_tbl RENAME COLUMN signup_date TO signup;

# 删除栏位
ALTER TABLE user_tbl DROP COLUMN email;

# 表格更名
ALTER TABLE user_tbl RENAME TO backup_tbl;

# 删除表格
DROP TABLE IF EXISTS backup_tbl;
```

## 与mysql对比

- 原先在mysql中会比较建议用反引号(`)来对字段进行转义，在postgres中就不行了

MySQL 可以使用单引号（’）或者双引号（"）表示值，但是 PG 只能用单引号（’）表示值，PG 的双引号（"）是表示系统标识符的，比如表名或者字段名。MySQL可以使用反单引号（`）表示系统标识符，比如表名、字段名，PG 也是不支持的

- 时间类型：postgres中时间类型带上`with time zone`时区

这个在用的时候挺方便的，避免时区不对。

原先一直用[https://github.com/silenceper/gogen](https://github.com/silenceper/gogen)(一个自动化orm生成工具)来生成db相关的操作一开始只支持mysql，后面加入了postgres的支持就屏蔽了对底层db的选择，方便。

不过对于go开发来说，只要不是手写sql取执行，而是利用一些orm工具，底层都封装了对多种数据库的支持，切换就变得更简单。

**这里引入一个话题：golang开发者到底该不该用orm？**

我自己觉得，有个取舍，如果需要写的sql比较复杂，关联关系又太多，虽然orm工具也提供了这种关系的对应，但是用的话还不如自己来写sql简单，而且也易读。

如果是业务系统是比较清晰的，已经被拆分的够简单，我基本上都会使用orm工具。
