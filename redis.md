<!-- GFM-TOC -->
* [主从分离](#主从分离)
* [哨兵](#哨兵)
<!-- GFM-TOC -->



# 主从分离
1.修改不同的redis-conf，主要修改port和pidfile(不同服务器上就不用修改了)

2.读写分离配置，在master上加 slave-read-only yes

3.从配置加  slaveof master_ip master_port   也可以连接到从服务再执行这个命令(手动连接到master)

如果是修改配置文件连接到master，重连时会自动连接到master，并且同步master上的数据

如果是手动连接到master，重连后，从服务器会读取本地的rdb恢复数据，不会自动连接到master

**主从复制工作原理**

![image](https://github.com/Wang520YY/wiki/blob/master/images/redis_copy.jpg)

1.slave连接master，发送sync

2.master接收到sync，fork一个子进程执行bgsave命令生成rdb，父进程继续接收client的请求并使用缓冲区记录此后执行的所有写命令

3.master生成rdb后，向所有slave发送rdb内存快照，并在发送期间继续记录被执行的写命令

4.slave接收到rdb后丢弃所有旧数据，载入收到的rdb

5.master发送rdb完毕开始想slave发送缓冲区中的命令

6.slave完成对快照载入，开始接收命令请求，并执行来自master缓存区的写命令

# 哨兵
哨兵是在主从分离的基础上，Master状态监测，如果Master 异常，则会进行Master-slave 转换，将其中一个Slave作为Master，将之前的Master作为Slave

Master-Slave切换后，master_redis.conf、slave_redis.conf和sentinel.conf的内容都会发生改变

![image](https://github.com/Wang520YY/wiki/blob/master/images/sentinel1.jpg)

![image](https://github.com/Wang520YY/wiki/blob/master/images/sentinel2.jpg)![image](https://github.com/Wang520YY/wiki/blob/master/images/sentinel3.jpg)

**哨兵工作原理**

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

