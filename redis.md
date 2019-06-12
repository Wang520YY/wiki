<!-- GFM-TOC -->
* [Tips](#Tips)
* [主从分离](#主从分离)
* [哨兵](#哨兵)
<!-- GFM-TOC -->

# Tips
## 对比memcache
**数据一致性**

memcache是多线程(master-worker模式)，可能出现并发锁冲突，可以通过cas命令来保证数据一致性(乐观锁解决方案，gets时获得当前值的唯一标识，修改时判断标识改变了写命令失败)

redis是单线程，串行操作不需要考虑数据一致性问题

**存储**

memcache不支持持久化，挂掉数据丢失，但是可以缓存图片视频等

redis支持持久化，并且支持多种数据类型

# 主从分离
1.修改不同的redis-conf，主要修改port和pidfile(不同服务器上就不用修改了)

2.读写分离配置，在master上加 slave-read-only yes

3.从配置加  slaveof master_ip master_port   也可以连接到从服务再执行这个命令(手动连接到master)

如果是修改配置文件连接到master，重连时会自动连接到master，并且同步master上的数据

如果是手动连接到master，重连后，从服务器会读取本地的rdb恢复数据，不会自动连接到master

## 主从复制工作原理

主从刚开始连接，进行全量同步，然后再进行增量同步，也可以在热河时候发起全量同步

如果增量同步不成功，进行全量同步，主从复制对于master和slave都是非阻塞的，可以接收外界请求

**全量同步**

![image](https://github.com/Wang520YY/wiki/blob/master/images/redis_copy.jpg)

1.slave连接master，发送sync

2.master接收到sync，fork一个子进程执行bgsave命令生成rdb，父进程继续接收client的请求并使用缓冲区记录此后执行的所有写命令

3.master生成rdb后，向所有slave发送rdb内存快照，并在发送期间继续记录被执行的写命令

4.slave接收到rdb后丢弃所有旧数据，载入收到的rdb

5.master发送rdb完毕开始想slave发送缓冲区中的命令

6.slave完成对快照载入，开始接收命令请求，并执行来自master缓存区的写命令

**增量同步**

master每执行一个写命令就会向slave发送相同的写命令，然后slave执行

**部分同步**

再2.8后，当主从断连后，主从可以不进行全量同步，而进行部分同步

master为每个slave维护了一份同步备份日志和同步标识，slave再重连后会携带同步标识和偏移量，如果在同步日志中有，那就从这个偏移量开始同步

**无盘复制**

主要用于master磁盘空间有限并且网络状况良好，master直接开启一个socket将rdb文件发送给slave

## 持久化

**RDB方式**

当指定时间内被更改的键超过指定数值就会进行快照；例如 save 300 10 三百秒内有至少10个键被更改则进行快照，可以设置多个

一旦Redis异常退出，就会丢失最后一次快照以后更改的所有数据

**RDB工作原理**

fork子进程将内存数据写入硬盘临时文件，父进程继续接收命令，当子进程写完所有数据临时文件会替换旧的RDB

**RDB手动快照**

发送save或bgsave命令，save命令会阻塞其它请求，bgsave会非阻塞fork子进程进行快照

**AOF方式**

通过设置appendonly yes开启，redis启动时会逐个执行AOF文件中的命令来讲硬盘数据载入内存，比RDB慢

一旦Redis异常退出，就会丢失最后一次同步以后更改的所有数据，一条或一秒的数据

**配置AOF自动重写条件**

为了解决AOF文件体积膨胀的问题，Redis提供了AOF重写功能，新的AOF文件不会包含任何浪费空间的冗余命令，重写的原理和rdb主从复制原理一样(父子进程)

auto-aof-rewrite-percentage 100 当AOF文件大小超过上一次重写时AOF文件大小的百分之多少时会再次进行重写

auto-aof-rewirte-min-size 64mb 允许重写的最小AOF文件大小

**AOF同步间隔**

appendfsync everysec 每秒一次

always，每次写入都会执行，不建议，安全但是很慢

no 不主动进行，完全交由操作系统30秒一次，最快但也最不安全

**建议**

在架构良好的环境中，master通常使用AOF，slave使用snapshot，主要原因是master需要首先确保数据完整性，它作为数据备份的第一选择；slave提供只读服务(目前slave只能提供读取服务)，它的主要目的就是快速响应客户端read请求；但是如果你的redis运行在网络稳定性差/物理环境糟糕情况下，建议你master和slave均采取AOF，这个在master和slave角色切换时，可以减少“人工数据备份”/“人工引导数据恢复”的时间成本；如果你的环境一切非常良好，且服务需要接收密集性的write操作，那么建议master采取snapshot，而slave采用AOF

# 哨兵
哨兵是在主从分离的基础上，Master状态监测，如果Master 异常，则会进行Master-slave 转换，将其中一个Slave作为Master，将之前的Master作为Slave

Master-Slave切换后，master_redis.conf、slave_redis.conf和sentinel.conf的内容都会发生改变

![image](https://github.com/Wang520YY/wiki/blob/master/images/sentinel1.jpg)

![image](https://github.com/Wang520YY/wiki/blob/master/images/sentinel2.jpg)![image](https://github.com/Wang520YY/wiki/blob/master/images/sentinel3.jpg)

## 哨兵工作原理

1.每个sentinel以每秒一次向master、slave、其它sentinel实例发送ping命令

2.每个实例距离最后一次有效回复ping命令时间超过down-after-milliseconds值，则这个实例被标记为主观下线

3.如果一个master被标记为主观下线，则正在监视这个master的所有sentinel都要以每秒一次的频率确认master是否进入了主观下线状态

4.当有足够数量(大于等于指定值)的sentinel在指定时间范围内确认master进入了主观下线，则master会被标记为客观下线

5.一般情况下，sentinel会以10秒频率向master、slave发送info命令

6.当master被标记为客观下线时，sentinel向下线的master的所有slave发送info命令会变为1秒一次

7.若master重新向sentinel的ping命令返回有效回复，master的主观下线就会被移除
若没有足够数量的sentinel同意master已经下线，master的客观下线状态会被移除



**简单的哨兵配置**

```
daemonize yes
port 6383
logfile "/usr/local/redis/var/sentinel.log"
#指定工作目录
dir "/usr/local/redis/var-sentinel"
#指定别名  主节点地址  端口  哨兵个数（有几个哨兵监控到主节点宕机执行转移）
sentinel monitor mymaster 127.0.0.1 6380 1
#如果哨兵3s内没有收到主节点的心跳，哨兵就认为主节点宕机了，默认是30秒
sentinel down-after-milliseconds mymaster 3000
#选举出新的主节点之后，可以同时连接从节点的个数
sentinel parallel-syncs mymaster 1
#如果10秒后,master仍没活过来，则启动failover,默认180s
sentinel failover-timeout mymaster 10000
#配置连接redis主节点密码
sentinel auth-pass mymaster 123456  
```
注意启动顺序一定要是master->slave->sentinel

