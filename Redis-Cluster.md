# Redis-Cluster
## redis使用中遇到的瓶颈
  我们日常在对于redis的使用中，经常会遇到一些问题

　　1. 高可用问题，如何保证redis的持续高可用性。

　　2. 容量问题，单实例redis内存无法无限扩充，达到32G后就进入了64位世界，性能下降。

　　3. 并发性能问题，redis号称单实例10万并发，但也是有尽头的。

 

## redis-cluster的优势　　
　　1. 官方推荐，毋庸置疑。

　　2. 去中心化，集群最大可增加1000个节点，性能随节点增加而线性扩展。

　　3. 管理方便，后续可自行增加或摘除节点，移动分槽等等。

　　4. 简单，易上手。

 

## redis-cluster名词介绍
　　1. master　　主节点、

　　2. slave　　　从节点

　　3. slot　　　　槽，一共有16384数据分槽，分布在集群的所有主节点中。

 

## redis-cluster简介
 

![redis-cluster](https://images2015.cnblogs.com/blog/997621/201703/997621-20170330105832764-1351765071.png)

 

 

图中描述的是六个redis实例构成的集群

6379端口为客户端通讯端口

16379端口为集群总线端口

集群内部划分为16384个数据分槽，分布在三个主redis中。

从redis中没有分槽，不会参与集群投票，也不会帮忙加快读取数据，仅仅作为主机的备份。

三个主节点中平均分布着16384数据分槽的三分之一，每个节点中不会存有有重复数据，仅仅有自己的从机帮忙冗余。

 

集群部署
测试部署方式，一台测试机多实例启动部署。

安装redis
```
$ wget http://download.redis.io/releases/redis-3.2.8.tar.gz
$ tar xzf redis-3.2.8.tar.gz
$ cd redis-3.2.8
$ make
```
修改配置文件 redis.conf
```
#redis.conf默认配置
daemonize yes
pidfile /var/run/redis/redis.pid  #多实例情况下需修改，例如redis_6380.pid
port 6379　　　　　　　　#多实例情况下需要修改,例如6380
tcp-backlog 511
bind 0.0.0.0　　　　　　
timeout 0
tcp-keepalive 0
loglevel notice
logfile /var/log/redis/redis.log　　　　　　#多实例情况下需要修改，例如6380.log
databases 16
save 900 1
save 300 10
save 60 10000
stop-writes-on-bgsave-error yes
rdbcompression yes
rdbchecksum yes
dbfilename dump.rdb　　#多实例情况下需要修改，例如dump.6380.rdb
slave-serve-stale-data yes
slave-read-only yes
repl-diskless-sync no
repl-diskless-sync-delay 5
repl-disable-tcp-nodelay no
slave-priority 100
appendonly yes
appendfilename "appendonly.aof"　　#多实例情况下需要修改,例如 appendonly_6380.aof
appendfsync everysec
no-appendfsync-on-rewrite no
auto-aof-rewrite-percentage 100
auto-aof-rewrite-min-size 64mb
aof-load-truncated yes
lua-time-limit 5000
slowlog-log-slower-than 10000
slowlog-max-len 128
latency-monitor-threshold 0
notify-keyspace-events ""
hash-max-ziplist-entries 512
hash-max-ziplist-value 64
list-max-ziplist-entries 512
list-max-ziplist-value 64
set-max-intset-entries 512
zset-max-ziplist-entries 128
zset-max-ziplist-value 64
hll-sparse-max-bytes 3000
activerehashing yes
client-output-buffer-limit normal 0 0 0
client-output-buffer-limit slave 256mb 64mb 60
client-output-buffer-limit pubsub 32mb 8mb 60
hz 10

#################自定义配置
#系统配置
#vim /etc/sysctl.conf
#vm.overcommit_memory = 1

aof-rewrite-incremental-fsync yes
maxmemory 4096mb
maxmemory-policy allkeys-lru
dir /opt/redis/data　　　　　　#多实例情况下需要修改，例如/data/6380

#集群配置
cluster-enabled yes
cluster-config-file /opt/redis/6380/nodes.conf   #多实例情况下需要修改，例如/6380/
cluster-node-timeout 5000


#从ping主间隔默认10秒
#复制超时时间
#repl-timeout 60

#远距离主从
#config set client-output-buffer-limit "slave 536870912 536870912 0"
#config set repl-backlog-size 209715200
```
启动六个实例：
```
/编译安装目录/src/redis-server redis.conf
```
注意，redis.conf应为6个不同的修改过的多实例配置文件。 

注意，配置文件复制六分后，有许多需要你修改的地方。

 

 

## 创建redis-cluster
redis-trib.rb命令与redis-cli命令放置在同一个目录中，可全路径执行或者创建别名。
```
redis-trib.rb create --replicas 0 127.0.0.1:6310 127.0.0.1:6320 127.0.0.1:6330 127.0.0.1:6340 127.0.0.1:6350 127.0.0.1:6360
```
只要缺失了任意一部分的槽，redis-cluster便无法读取。
测试强行停机一台，既显示：
127.0.0.1:6310> get key
(error) CLUSTERDOWN The cluster is down
注：这里分片设置为了0
启动丢失的那一台后既恢复。数据不会丢失。
 
 
 
 
## 移动槽
```
redis-trib.rb reshard 127.0.0.1:6360
```
执行集群reshard操作，通过集群中127.0.0.1:6360这一台机器
 ```
[OK] All 16384 slots covered.
How many slots do you want to move (from 1 to 16384)? 2731
```
输入需要移动的槽数量
 ```
What is the receiving node ID? 21c93aa709e10f7a9064faa04539b3ecd
```
输入接收的节点的ID
```
How many slots do you want to move (from 1 to 16384)? 2731
What is the receiving node ID? 0abf4ca21c93aa709e10f7a9064faa04539b3ecd
Please enter all the source node IDs.
  Type 'all' to use all the nodes as source nodes for the hash slots.
  Type 'done' once you entered all the source nodes IDs.
Source node #1:0ddb4e430dda8778ac873dd169951c7d71b8235e
Source node #2:done
```
输入所有被移动的节点ID，确认后输入done
```
    Moving slot 5460 from 0ddb4e430dda8778ac873dd169951c7d71b8235e
    Moving slot 13653 from 0ddb4e430dda8778ac873dd169951c7d71b8235e
Do you want to proceed with the proposed reshard plan (yes/no)?
```
检查后输入yes进行移动分槽
至此，分槽移动完毕。
 
 
## 删除节点
``` 
redis-trib.rb del-node 127.0.0.1:6360 'f24c0c1ecf629b5413cbca632d389efcad7c8346'
```
最后跟着的是这个节点的ID，可在redis-cli终端中使用CLUSTER NODES查看
必要条件，此节点所有分槽均已移除。
 
 
## 添加master节点
```
redis-trib.rb add-node 127.0.0.1:6360 127.0.0.1:6350
```
新节点必须是空的，不能包含任何数据。请把之前aof和dump文件删掉，并且若有nodes.conf也需要删除。
add-node  将一个节点添加到集群里面， 第一个是新节点ip:port, 第二个是任意一个已存在节点ip:port
node:新节点没有包含任何数据， 因为它没有包含任何slot。新加入的加点是一个主节点， 当集群需要将某个从节点升级为新的主节点时， 这个新节点不会被选中，同时新的主节点因为没有包含任何slot，不参加选举和failover。
 
 
## 添加一个从节点
前三步操作同添加master一样
第四步:redis-cli连接上新节点shell,输入命令:cluster replicate 对应master的node-id
