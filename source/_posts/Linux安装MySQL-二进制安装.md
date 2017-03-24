---
title: Linux安装MySQL-二进制安装
date: 2016-03-08 19:12:04
tags: MySQL
---

## 安装步骤

### 下载MySQL二进制包

下载地址: [mysql-5.6.28-linux-glibc2.5-x86_64.tar.gz](http://dev.mysql.com/downloads/mysql/5.6.html#downloads)

### 创建用户和组

```powershell
[root@shark]# groupadd mysql
[root@shark]# useradd -r -g mysql mysql
```

### 解压安装MySQL

```powershell
[root@shark]# cd /usr/local
[root@shark]# tar -zxvf mysql-5.6.28-linux-glibc2.5-x86_64.tar.gz
[root@shark]# mv mysql-5.6.20-linux-glibc2.5-i686 mysql
```

### 修改MySQL的各级文件夹和文件的创建者、创建组为mysql

```powershell
[root@shark]# cd /usr/local/mysql
[root@shark]# chown -R mysql .
[root@shark]# chgrp -R mysql .
```
提示："."代表的就是当前目录，选项-R表示递归当前目录及其子目录

### 初始化MySQL

```powershell
[root@shark]# scripts/mysql_install_db --user=mysql --basedir=/usr/local/mysql --datadir=/usr/local/mysql/data
[root@shark]# cp support-files/mysql.server /etc/init.d/mysqld
```
提示：初始化时，有可能会报错误，安装需要的包即可:libstdc++.so.6和libaio.so.1和libncurses.so.5
### 修改配置文件

```powershell
[root@shark]# cp support-files/my-default.cnf /etc/my.cnf （5.6.x复制my-default.cnf，5.5.x复制my-medium.cnf）
[root@shark]# vim /etc/my.cnf
```
在[mysqld]中添加：

```powershell
basedir = /usr/local/mysql
datadir = /usr/local/mysql/data
port = 3306
server_id = 1
```

### 启动mySQL服务

```powershell
[root@shark]# service mysqld start
```
### 登陆MySQL并且删除空用户，并设置root账户密码，开启root远程访问权限等

```powershell
[root@shark]# mysql -uroot
……

mysql> use mysql;

mysql> SELECT user, host, password FROM mysql.user;
+------+-----------+----------+
| user | host      | password |
+------+-----------+----------+
| root | localhost |          |
| root | 10-4-5-9  |          |
| root | 127.0.0.1 |          |
| root | ::1       |          |
|      | localhost |          |
|      | 10-4-5-9  |          |
+------+-----------+----------+
6 rows in set (0.00 sec)
 
mysql> UPDATE mysql.user set password = PASSWORD('123456') WHERE user = 'root';

mysql> GRANT ALL PRIVILEGES ON *.* TO root@"%" IDENTIFIED BY "123456";　　//为root添加远程连接的能力，123456为root账户密码

mysql> DROP USER ''@localhost;

mysql> FLUSH PRIVILEGES;
```
