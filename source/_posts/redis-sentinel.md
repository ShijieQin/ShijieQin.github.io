---
title: redis sentinel
comments: true
date: 2017-06-15 13:37:36
updated: 2017-06-15 13:37:36
tags:
- redis
- sentinel
- HA
categories:
- redis
---
### 前言
该文章只是简单的记录redis sentinel的安装过程，并没有深度优化，只能保证安装完成后可以使用。后续再继续优化。
master ip: 100.65.9.250
slave1 ip: 100.65.16.158
slave2 ip: 100.65.16.162
<!-- more -->
### Redis安装
redis的安装包去redis官网下载
#### 安装依赖
```
yum -y install gcc gcc-c++ tcl
```
#### 编译
```
tar -zxvf redis-3.2.9.tar.gz
cd redis-3.2.9
make
```
这一步可能出现如下错误
```
cd src && make all
make[1]: Entering directory `/root/redis-3.2.9/src'
    CC adlist.o
In file included from adlist.c:34:
zmalloc.h:50:31: error: jemalloc/jemalloc.h: No such file or directory
zmalloc.h:55:2: error: #error "Newer version of jemalloc required"
make[1]: *** [adlist.o] Error 1
make[1]: Leaving directory `/root/redis-3.2.9/src'
make: *** [all] Error 2
```
执行如下命令解决
```
make distclean
```
再重新make
#### 安装
```
cd src
make install PREFIX=/usr/local/redis
cd ..
cp redis.conf /usr/local/redis/
cp sentinel.conf /usr/local/redis/
```
#### 配置master和slave的配置文件
编辑/usr/local/redis/redis.conf
修改如下内容
```
bind 0.0.0.0
# 访问master的密码 由于master和slave会互换，所以两个密码都最好配上
masterauth 123456
# redis的访问密码
requirepass 123456
# 开启守护模式  
daemonize yes  
# 指定数据存储目录  
dir /data/redis  
# 打开aof持久化  
appendonly yes  
# 每秒一次aof写  
appendfsync everysec
# 修改日志路径
logfile "/var/log/redis.log"
```
#### 配置slave的配置文件
在以上配置文件的基础上添加如下内容
```
# 指定所属的主机  
slaveof 100.65.9.250 6379  
# 指定从机"只读"  
slave-read-only yes
```
#### 配置sentinel配置文件
编辑所有服务器的/usr/local/redis/sentinel.conf文件  
添加或修改如下内容
```
bind 0.0.0.0
daemonize yes
logfile "/var/log/sentinel.log"
# sentinel需要监控的master/slaver信息，格式为sentinel monitor <mastername> <masterIP> <masterPort> <quorum>  
# 其中<quorum>应该小于集群中slave的个数，当失效的节点数超过了<quorum>,则认为整个体系结构失效
sentinel monitor mymaster 100.65.9.250 6379 2
# master被当前sentinel实例认定为失效的间隔时间，格式为sentinel down-after-milliseconds <mastername> <milliseconds>
sentinel down-after-milliseconds mymaster 10000
# 当新master产生时，同时进行“slaveof”到新master并进行同步复制的slave个数  
# 在salve执行salveof同步时，将会终止客户端请求。  
# 此值较大,意味着“集群”终止客户端请求的时间总和和较大。  
# 此值较小,意味着“集群”在故障转移期间，多个salve向客户端提供服务时仍然使用旧数据。
sentinel parallel-syncs mymaster 1
# failover过期时间。当failover开始后，在此时间内仍然没有触发任何failover操作，当前sentinel将会认为此次failoer失败。
sentinel failover-timeout mymaster 60000
```
#### 启动三台服务器的redis
```
/usr/local/redis/bin/redis-server /usr/local/redis/redis.conf
```
#### 启动三台服务器的sentinel
先确保redis已经启动
```
/usr/local/redis/bin/redis-sentinel /usr/local/redis/sentinel.conf
```
### 测试
#### 在任意一个节点上查看主从机的复制信息
##### 查看master节点信息
```
[root@redis ~]# /usr/local/redis/bin/redis-cli -h 100.65.9.250 -p 6379 -a 123456 info Replication
# Replication
role:master
connected_slaves:2
slave0:ip=100.65.16.158,port=6379,state=online,offset=2437,lag=0
slave1:ip=100.65.16.162,port=6379,state=online,offset=2437,lag=0
master_repl_offset:2437
repl_backlog_active:1
repl_backlog_size:1048576
repl_backlog_first_byte_offset:2
repl_backlog_histlen:2436
```
##### 查看slave节点信息
```
[root@redis ~]# /usr/local/redis/bin/redis-cli -h 100.65.16.158 -p 6379 -a 123456 info Replication
# Replication
role:slave
master_host:100.65.9.250
master_port:6379
master_link_status:up
master_last_io_seconds_ago:9
master_sync_in_progress:0
slave_repl_offset:2605
slave_priority:100
slave_read_only:1
connected_slaves:0
master_repl_offset:0
repl_backlog_active:0
repl_backlog_size:1048576
repl_backlog_first_byte_offset:0
repl_backlog_histlen:0
```
#### 测试数据
##### 客户端连接master set一条测试数据
```
[root@redis ~]# /usr/local/redis/bin/redis-cli -h 100.65.9.250 -p 6379 -a 123456
100.65.9.250:6379> set test thisistest
OK
100.65.9.250:6379>
```
##### 客户端连接任意slave查看数据
```
[root@redis ~]# /usr/local/redis/bin/redis-cli -h 100.65.16.158 -p 6379 -a 123456
100.65.16.158:6379> get test
"thisistest"
100.65.16.158:6379>
```
#### 测试主从检测
关闭一台slave节点，然后检测节点信息
```
[root@redis ~]# /usr/local/redis/bin/redis-cli -h 100.65.9.250 -p 6379 -a 123456 info Replication
# Replication
role:master
connected_slaves:1
slave0:ip=100.65.16.162,port=6379,state=online,offset=4082,lag=0
master_repl_offset:4082
repl_backlog_active:1
repl_backlog_size:1048576
repl_backlog_first_byte_offset:2
repl_backlog_histlen:4081
```
发现少了一个slave节点  
再启动关闭的slave节点，并检测
```
[root@redis ~]# /usr/local/redis/bin/redis-cli -h 100.65.9.250 -p 6379 -a 123456 info Replication
# Replication
role:master
connected_slaves:2
slave0:ip=100.65.16.162,port=6379,state=online,offset=4362,lag=0
slave1:ip=100.65.16.158,port=6379,state=online,offset=4362,lag=0
master_repl_offset:4362
repl_backlog_active:1
repl_backlog_size:1048576
repl_backlog_first_byte_offset:2
repl_backlog_histlen:4361
```
发现检测到slave节点又变为了2  
关闭master，然后检测节点信息
```
[root@redis ~]# /usr/local/redis/bin/redis-cli -h 100.65.16.162 -p 6379 -a 123456 info Replication
# Replication
role:slave
master_host:100.65.9.250
master_port:6379
master_link_status:down
master_last_io_seconds_ago:-1
master_sync_in_progress:0
slave_repl_offset:4586
master_link_down_since_seconds:27
slave_priority:100
slave_read_only:1
connected_slaves:0
master_repl_offset:0
repl_backlog_active:0
repl_backlog_size:1048576
repl_backlog_first_byte_offset:0
repl_backlog_histlen:0
```
发现并没有马上选举出新的master，还是就得master，这段时间是不可写的。  
过一段时间后，重新检测，发现选举出了新的master。
