<!-- GFM-TOC -->
* [cpu基础知识](#cpu基础知识)
<!-- GFM-TOC -->

# cpu基础知识

**物理CPU** 计算机实际配置的cpu个数，查看命令 cat /proc/cpuinfo | grep "physical id" | sort -u | wc -l

2、cpu核数，cpu上集中处理数据的cpu核心个数，如果核心数1个，个数4个，就是4核， 查看命令 cat /proc/cpuinfo | grep "cpu cores" | uniq | wc -l

3、逻辑cpu 如果物理cpu支持超线程，逻辑cpu就是核心数的两倍，查看命令 cat /proc/cpuinfo | grep "processor" | wc -l

4、超线程，一项技术，使得单个处理器可以像两个逻辑处理器那样运行，这样单个处理器以并行执行线程

**单核**

![image](https://github.com/Wang520YY/wiki/blob/master/images/单核.jpg)
 
 **多核**
 
![image](https://github.com/Wang520YY/wiki/blob/master/images/多核.jpg)

**超线程**

 ![image](https://github.com/Wang520YY/wiki/blob/master/images/超线程.jpg)

5、时钟频率(主频) 用来表示cpu的运算，处理数据的速度

6、指令流水线 为提高处理器执行指令效率，把一条指令操作分为多个细小的步骤，每个步骤由专门的电路完成

7、cpu缓存 cpu与内存之间临时数据交换器，总共三级缓存（L1一级缓存、L2二级缓存、L3三级缓存）




