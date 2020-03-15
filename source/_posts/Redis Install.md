---
title: Redis Install
date: 2018-03-09 19:07:29
tags:
    - Linux
    - Redis
categories:
	- Redis
---



```
sudo yum install gcc-c++    --配置编译环境

wget http://download.redis.io/releases/redis-3.2.8.tar.gz   --下载源码

tar -zxvf redis-3.2.8.tar.gz    --解压源码

cd redis-3.2.8

make    --执行 make 编译 Redis
```
注意：make命令执行完成编译后，会在src目录下生成6个可执行文件，分别是 redis-server、redis-cli、redis-benchmark、redis-check-aof、redis-check-rdb、redis-sentinel。

```
make install    --安装 Redis

./utils/install_server.sh   --配置Redis能随系统启动
```

# Redis 服务查看、开启、关闭:

```
ps -ef|grep redis               --命令查看 Redis 进程
/etc/init.d/redis_6379 start    --开启 Redis 服务 （service redis_6379 start）
/etc/init.d/redis_6379 stop     --关闭 Redis 服务 （service redis_6379 stop）
```

# redis.conf 的配置信息

```
daemonize         如果需要在后台运行，把该项改为yes
pidfile           配置多个 pid 的地址 默认在 /var/run/redis.pid
bind              绑定 ip，设置后只接受来自该 ip 的请求
port              监听端口，默认是 6379
loglevel          分为4个等级： debug verbose notice warning
logfile           用于配置 log 文件地址
databases         设置数据库个数，默认使用的数据库为0
save              设置redis进行数据库镜像的频率。
rdbcompression    在进行镜像备份时，是否进行压缩
dbfilename        镜像备份文件的文件名
Dir               数据库镜像备份的文件放置路径
Slaveof           设置数据库为其他数据库的从数据库
Masterauth        主数据库连接需要的密码验证
Requriepass       设置 登陆时需要使用密码
Maxclients        限制同时使用的客户数量
Maxmemory         设置 redis 能够使用的最大内存
Appendonly        开启 append only 模式
vm-enabled        是否开启虚拟内存支持 （vm开头的参数都是配置虚拟内存的）
vm-swap-file      设置虚拟内存的交换文件路径
vm-max-memory     设置 redis 使用的最大物理内存大小
vm-page-size      设置虚拟内存的页大小
vm-pages          设置交换文件的总的 page 数量
vm-max-threads    设置 VM IO 同时使用的线程数量
Glueoutputbuf     把小的输出缓存存放在一起
Activerehashing   重新 hash
hash-max-zipmap-entries 设置 hash 的临界值
```