---
title: Mysql Install
date: 2020-03-04 12:13:24
tags: 
	- Linux
	- Mysql
categories:
	- Mysql
---

# 1、添加包

```shell
wget https://dev.mysql.com/get/mysql80-community-release-el7-3.noarch.rpm
rpm -ivh mysql80-community-release-el7-3.noarch.rpm
```

# 2、更新 yum 命令

```shell
yum clean all && yum makecache
```

# 3、安装

```shell
yum install -y  mysql-community-server
```

# 4、配置文件

```shell
vim /etc/my.cnf
```



```shell
[mysqld]

port = 3306

character-set-server=utf8mb4
collation-server=utf8mb4_general_ci

# 表名不区分大小写(启动前配置)
lower_case_table_names=1

#设置日志时区和系统一致
log_timestamps=SYSTEM

[client]
default-character-set=utf8mb4
```



# 5、启动服务

```shell
#启动服务
systemctl start mysqld

#查看版本信息
mysql -V

#查看状态
systemctl status mysqld

#开机启动
systemctl enable mysqld
systemctl daemon-reload
```

# 6、修改账号密码

```shell
#查看 MySQL 为 root 账号生成的临时密码
grep "A temporary password" /var/log/mysqld.log

#进入 MySQL
mysql -u root -p

#查看 validate_password 相关参数
show global variables like 'validate_password%';

#修改密码策略
set global validate_password.policy=0;
set global validate_password.length=4;
set global validate_password.check_user_name=0;

#修改密码
ALTER USER 'root'@'localhost' IDENTIFIED BY 'root';
```

# 7、设置 MySQL 允许外网访问

```shell
#进入 mysql 库
use mysql;

#更新域属性，'%'表示允许外部访问
update user set host='%' where user ='root';

#刷新权限
FLUSH PRIVILEGES;

#授权
GRANT ALL PRIVILEGES ON *.* TO 'root'@'%'WITH GRANT OPTION;
```