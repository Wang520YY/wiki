<!-- GFM-TOC -->
* [tip](#tip)
* [事务](#事务)
* [索引](索引)
* [存储引擎](#存储引擎)
* [锁](#锁)
* [explain](#explain)
* [分区分库分表](#分区分库分表)
* [主从复制原理](#主从复制原理)
<!-- GFM-TOC -->

# tip
1、like查询%在前、字符串查询不加引号、!=、or其中一列未使用索引、or两边使用联合索引的不同列都会使索引失效，但is null可以使用到索引

2、utf8mb4编码，mb4的意思就是most Bytes 4，专门针对4字节的unicode，而传统的utf8最大支持长度为3字节，Emoji就是4字节导致插入异常

3、char适用于定长，短，经常变更的列，varchar经常变更易产生碎片，char是时间换空间效率高

4、BLOB和TEXT会用到临时表

5、确保GROUP BY和ORDER BY中只有一个表中的列，这样才可能使用索引

# 事务
**四个特性**

1.原子性，事务操作要么发生要么不发生；

2.隔离性，并发事务之间是隔离的；

3.一致性，事务的执行是从一个正确的状态转为另一个正确的状态；

4.持久性，事务提交后持久保存在数据库中

**隔离级别**

1.read-uncommit，存在脏读问题(事务a对某数据做了修改，事务b使用了该数据，事务a因某些原因回滚了，导致事务b使用了脏数据)

2.read-commit,存在不可重复读问题(事务a读取某数据，事务b对该数据做了修改，事务a再次读该数据发现发生了改变，注重于修改)，select for update可解决

3.repeatable read，存在幻读问题(事务a对100条数据修改前，事务b插入了一条数据，这导致事务a实际是对101条数据做了修改，注重于插入删除)

4.serializable，串行化
    
# 索引
**其它的树结构**

1、二叉查找树：左子树<根<右子树，时间复杂度log2n

2、平衡二叉树：左右子树高度差不大于1，时间复杂度logn

3、平衡多路查找树B-Tree（B树）

    非叶子节点存放：指针p、数据data、键值[key,val]
    
    叶子节点存放：数据data、键值[key,val]
    
**B+Tree**

1.非叶子几点存放：键值和指针

2.叶子节点存放：键值，data，双向链指针(构成链式环结构)

相较于B-Tree

1、所有关键字均在叶子节点，非叶子节点可以存放更多key，降低了高度，减少了磁盘IO

2、叶子节点的链指针有利于范围查找和分页查找

**B+Tree存放**

首先了解下最小存储单元

1、磁盘存储的最小单元是扇区，512字节

2、文件系统的最小单元是块，4k，因此一个文件即使1个字节也不得不占用4KB的空间

而InnerDB最小存储单元是页，16K，也可以设置

非叶子节点(指针6B+bigint主键8B)，这样一个page能存放16384/14=1170

叶子节点如果一行数据算1K，那么高度为3的树能存放：1170*1170*16=21902400，这样基本能满足千万级的数据存储了

**二次查询**

利用索引的查找为B+tree的查找，否则就是线性查找

根据B+tree，已知主键ID能快速找出对应的数据，因此给字段加索引就是索引的字段和主键维护新的一个索引树，实质是先找出字段对应主键ID，然后二次查询

但是如果用了复合索引，就可以避免二次查询这种情况，例如name_age建立索引，那么根据name就可以直接在普通索引数上找到age

**聚簇**

InnoDb聚簇索引

由于数据存储的物理排列方式和聚簇索引的顺序相同，因此只能建立一个聚簇索引，数据都存储在叶子节点上

![image](https://github.com/Wang520YY/wiki/blob/master/images/InnerDB_B%2BTree.png)

MyIsam非聚簇索引

叶子节点有个指针指向对应的数据块

![image](https://github.com/Wang520YY/wiki/blob/master/images/myIsam_index.png)

# 存储引擎
InnoDB和MyIsam

InnoDb：

1、默认事务；

2、支持行级锁；

3、支持外键；

4、非叶子节点不存储实际数据只包含主键和索引，叶子节点双向链表相连，便于区间查找和遍历

存在表锁：SQL的更新（update）或者删除（delete）语句中未使用到索引，导致必须把整个表锁起来进行检索

MyIsam：

1、拥有全文索引（只针对英文），不建议使用(用es)；

2、不支持事务和行级锁，表锁写性能差；

# 锁
乐观锁：可读可写，使用场景在多读场景，省去了锁的开销

    实现方式1：版本号：事务a读这条数据版本号为1，事务b也读取这条数据版本号也为1，当a提交后更新版本号为2，b提交的时候发现版本号不一致导致提交失败
    
    实现方式2：CAS(compare and swap)算法，读取内存值V，进行比较的值A，写入时比较V==A时给V赋新值B，失败的线程挂起不断重试即自旋操作
    
    存在的问题：ABA，改为B后再改为A，版本号就不会有这个问题
    
悲观锁：每次在拿数据的时候都会上锁(行锁，表锁，读锁，写锁)

读锁(共享锁)：多个并发都可以读

写锁(排它锁)：阻塞其它写和读


# explain
**type**

system(const特例，表仅一行)

const(primary或unique) 

eq_ref(连表用到primary或unique) > 

ref(普通索引) 

index_merge(当使用or都用到索引，会合并结果集) 

range(区间) 

index(遍历索引树，使用了索引的排序)

all

**extra**
       
       using temporary，用到临时表，对非驱动表排序就会用到临时表，因为会对非驱动表循环查询合并后(临时表)的结果进行排序；

       using filesort，用到了mysql内部的文件排序
       
       using union 当使用or都用到索引，会合并结果集
       
# 分区分库分表
分区：可根据字段区间，一个表最多1024个分区，如果包含主键和唯一索引则分区字段必须包含

     partition by range(field) (
     
        partition <分区名称> values less than(value),
        
        partition <分区名称> values less than(maxvalue),
        
     )
     
     list分区，根据固定值分区，适用于枚举类型 partition<分区名称> vlaues IN(value1,value2...)
     
     Hash分区主要用来分散热点读，partition by hash(expr); partitins <num>; N = MOD(expr, num)
    

水平分表：

1.可根据活跃度，不同地区进行切分

2.hash拆分：ID%3这样就分为三张表

3.范围拆分：比如每台服务器计划存放一个亿的数据,先将数据写入服务器A.一旦服务器A写满,则将数据写入服务器B,以此类推.这种方式的好处是扩展方便.数据在各个服务器上分布均匀

垂直分表：常用列、不常用列、大字段列拆分等，需要join查询

全局ID生成器
1.uuid，不是自增且非数字，查询效率低
2.redis的incr自增，可以多台一起按照奇偶数，步长生成

hash获取所在表
functiong get_hash($tName, $id, $c = 100) {
    $hash = sprintf("%u", crc32($id));
    $hash = intval(fmod($hash, $c));
    return $tName . '_' . $hash;
}

分表后分页查询
1.union查询，不推荐，效率低下
2.维护一张主键和必要索引的总表
3.查询每张表的数据再合并排序返回

最终一致性2：
用户给主播送金币，扣除用户金币，push队列
pop队列，主播增加金币
定时检查对比用户减少和主播增加进行事务补偿问题排查
![image](https://github.com/Wang520YY/wiki/blob/master/images/gold.png)

最终一致性2：
维护一个全局事务表和分表消息日志表
比如转账，向队列push两个，一个为用户A -100，另一个为用户B +100
不管是数据库A或B，队列获取失败都进行回滚
事务ID，是否存在，如果没有不处理
队列日志ID，是否发过，如果有不处理，因为可能是消息超时重发导致
检查tran_log和msg_log.如果有不一致的情况,则进行事务补偿



# 主从复制原理
1.master记录更改的记录到bin-log(binary log events实质是二级制日志事件)

2.slave一个线程用于同步bin-log到中继日志(relay log)，另一个线程用于重做中继日志中的事件

![image](https://github.com/Wang520YY/wiki/blob/master/images/mysql_copy.png)

单点故障保证高可用：keepalived就可以自动切换master，实现vip漂移

**问题及解决方法**
主库宕机，数据丢失；从库只有一个sql Thread，主库写压力大时，复制延迟较长

1.半同步复制，解决数据丢失问题，性能有一定的降低，再确保同步成功后，master才返回客户端succss

![image](https://github.com/Wang520YY/wiki/blob/master/images/synccopy.png)

2.并行复制，解决从库复制延迟问题，从库多线程apply binlog，set global salve_parallel_workers=10;设置sql线程为10

3.级联复制，A->B->C，B中添加log-bin = mysql-bin;log_slave_updates = 1;将会把A的binlog记录到自己的binlog中，A是B的主，B是C的主



