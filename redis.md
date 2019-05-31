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

![image](https://github.com/Wang520YY/wiki/blob/master/images/mysql_copy.png)

# 哨兵
哨兵是在主从分离的基础上，监控master是否宕机，自动选取slave为新的master

**简单的哨兵配置**

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

注意启动顺序一定要是master->slave->sentinel

