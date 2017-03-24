---
title: MySQL data目录迁移
date: 2016-06-16 00:03:16
tags: MySQL
---


[参考链接]
[1] http://blog.csdn.net/chenpy/article/details/47046113
[2] https://www.zybuluo.com/oro-oro/note/310783

## 方法一：

(1) 停止mysql服务

```
service mysqld stop
```

(2) 移动数据到数据盘目录

```
mkdir -p /data/mysql
chown -R mysql:mysql /data
mv /usr/local/mysql/data/* /data/mysql/
```

(3) 修改配置/etc/my.cnf

提示：以下配置为示例，修改以实际情况为准，将`/usr/local/mysql`修改为`/data/mysql`，如：

`datadir = /data/mysql`
`socket = /data/mysql/mysql.sock`
`log_error = /data/mysql/mysql-error.log`
`slow_query_log_file = /data/mysql/mysql-slow.log`

```
[mysqld]

datadir = /data/mysql
port = 3306
server_id = 1
socket = /data/mysql/mysql.sock

default_storage_engine = InnoDB

#INNODB
innodb_buffer_pool_size = 4G
innodb_log_file_size = 8M
innodb_file_per_table = 1
innodb_flush_method = O_DIRECT

#MyISAM
key_buffer_size = 64M

#LOGGING
log_error = /data/mysql/mysql-error.log
slow_query_log = 1
slow_query_log_file = /data/mysql/mysql-slow.log

#OTHER
tmp_table_size = 64M
max_heap_table_size = 64M
query_cache_type = 1
query_cache_size = 64M
max_connections = 1000
thread_cache_size = 256
table_open_cache = 10000
open_files_limit = 65535

innodb_thread_concurrency = 4

skip_name_resolve
character_set_server=utf8

sql_mode=NO_ENGINE_SUBSTITUTION,STRICT_TRANS_TABLES

[client]
socket = /data/mysql/mysql.sock
port = 3306
host = localhost
```

## 方法二：

使用linux下的软连接方法，具体操作如下：（以迁移到/data/mysql/目录为例)

(1) 停止mysql服务

```
service mysqld stop
```

(2) 移动数据到数据盘目录

```
mkdir -p /data/mysql
chown -R mysql:mysql /data
mv /usr/local/mysql/data/* /data/mysql/
```

(3) 创建软连接/usr/local/mysql/data指向/data/mysql/目录

```
rm /usr/local/mysql/data //删除原有的data目，然后创建data软链接
ln -s /data/mysql/ /usr/local/mysql/data
```

(4) 启动mysql服务

```
service mysqld start
```

## 常见问题

(1) 首先迁移目标目录的权限问题

```
ERROR! The server quit without updating PID file
```

这个是权限问题，mysql没有这个目录下的权限 
因为刚复制过来的mysql文件的拥有者是root，所以要改变该目录的拥有者，执行下面命令来改变 
chown -R mysql:mysql /data/mysql/

(2) 用mysql客户端连接不上服务器

```
Can't connect to local MySQL server through socket '/var/lib/mysql/mysql.sock'
```

其实是你的mysql的配置文件没有配置正确。 
mysql启动如果没有读到

```
[client]
socket = /data/mysql/mysql.sock
port = 3306
host = localhost
```

这些配置，则mysql会默认找 /var/lib/mysql/mysql.sock文件， 
如果要解决这个问题就需要把上面的东西加入到你的/etc/my.cnf配置文件中。

下面普及下mysql的有关配置文件的知识，以前我也是迷迷糊糊，现在搞清楚了一些。 
用 service mysql start 命令启动mysql，mysql会默认找/etc/my.cnf 配置文件，如果没有找到，则会按照默认的启动目录来启动mysql 
就是【/var/lib/mysql】 
会在此目录下生成 10-4-19-161.pid 类似的pid文件 和 sock文件【mysql.sock】 
但是如果你建立的了my.cnf 文件则mysql会按照你的配置文件来启动mysql。

(3) 用户密码过期问题

```
Your password has expired. To log in you must change it using a client that supports expired passwords
```

则需要修改 mysql的用户表 
刚开始以为修改用户密码就可以解决密码过期的问题，其实不是的。需要修改msyql.user表的 password_expired值 
Y表示已经过期，需要修改成N。 
首先在本机进入mysql

```sql
> mysql -uroot -pxxxxx 
> use mysql
> select host,user,password_expired from user;
+-------------+----------+------------------+
| host        | user     | password_expired |
+-------------+----------+------------------+
| localhost   | root     | Y                |
| 10-4-19-161 | root     | Y                |
| 127.0.0.1   | root     | Y                |
| localhost   | data     | N                |
| 127.0.0.1   | shangbao | N                |
+-------------+----------+------------------+
```

如果有些是Y,则需要吧哪些是Y的改为N。 
update set user.password_expired = ‘N’ where user.user = ‘root’; 
root是你密码过期的用户

然后刷新权限
```sql
mysql> flush privileges;
```

因为mysql的权限是启动的时候缓存起来的，如果修改的用户的权限，需要手动刷新用户权限才会生效。