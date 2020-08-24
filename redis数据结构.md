# 底层数据结构
## 1、动态字符串SDS
```
struct sdshdr {
 int len;
 int free;
 char buf[];
}
```

**为什么不用原生C语言的字符串**

**1、c字符串以空格判结尾，而sds以free判结尾，二进制安全，可存图片视频等**

**2、c字符串O(n)复杂度计算长度，sds有len属性**

**3、sds也会以空格结尾，兼容部分c字符串函数**

## 2、链表
```
struct listNode {
  struct listNode *prev;
  struct listNode *next;
  void *value;
}
struct linkedlist {
 listNode *head;
 listNode *tail;
 unsigned long len;
 //复制
 void (*dup)(void *ptr);
 //释放
 void(*free)(void *ptr);
 //对比
 void(*match)(void *ptr,void *key)
}
```

**1、双向无环，head的prew和tail的next都指向NULL**

## 3、字典（map）
```
struct dictht {
  //哈希表
  dictEntry **table;
  unsigned long size;
  //总是等于size-1
  unsigned long sizemask;
  unsigned long used;
}

struct dictEntry {
  void *key;
  union {
    void *val;
    uint64_tu64;
    int64_tu64;
  }v;
  struct dictEntry *next;
}
```

**1、链地址法解决哈希冲突**

**2、负载因子 = 节点数量/哈希表大小，扩容和收缩都是原来大小的一倍**

**扩容：没有执行bgsave或bgrewriteaof，并且负载因子>=1,如果正在执行，且负载因子>=5**

**收缩：负载因子<=0.01**

**3、数据量大的时候，渐进式的rehash，保证可靠性，查询时会在新旧哈希表中查找**

## 4、跳跃表
```
struct zskiplistNode {
  struct zskiplistLevel {
    struct zskiplistNode *forward;
    //跨度
    unsigned int span;
  }level[];
  struct zskiplistNode *backword;
  double score;
  robj *obj;
}

struct zskiplist {
  struct skiplistNode *header,*tail;
  unsigned long length;
  //最大层数
  int level;
}
```

**1、搜索的时候会从最高层开始，最好的结果是log(N)**

**2、使用抛硬币决定层数，也就是趋近1/2的层数为2层**

## 5、整数集合
```
struct intset {
  uint32_t encoding;
  uint32_t length;
  int8_t contents[];
}
```

**1、递增顺序且不重复**

**2、encoding决定contents的具体类型**

**3、升级，根据新元素类型，扩展contents空间大小，将所有元素转为新元素类型（c语言是静态类型语言）**

**不支持降级，节约空间，不固定在一种类型**

## 6、压缩列表
```
//内存字节数、用于内存重新分配、计算zlend地址
uint32_t zlbytes;
//表尾节点到起始地址偏移量，用于计算表尾地址
uint32_t zltail;
//节点数量，最大65535，大于这个只能遍历计算
uint16_t zllen;
entryN...
//末端标识
uint8_t zlend;

entryN的构成
//前一个节点长度，用于计算前节点地址。当长度小于254字节占用1字节，大约时占用5字节，第一字节为OXFE(10进制的254)，后4字节记录长度
previous_entry_length;
//contents的类型及长度，最高位00、01、10表示字节数组，11表示整数，其它位表示长度
encoding;
contents;
```

**1、集中存储、节省内存，由一系列特殊编码的连续内存块组成的顺序型数据结构**

## 7、快速列表（3.2后用于list的底层数据结构）
```
struct quicklist {
   quicklistNode* head;
   quicklistNode* tail;
   //元素个数
   long count;
   //zipList节点个数
   int nodes;
   //LZF算法压缩深度
   int compressDepth;
   ...
}

struct quicklistNode {
   quicklistNode* prev;
   quicklistNode* next;
   //指向压缩列表
   ziplist* zl;
   //zipList总字节数
   int32 size;
   //zipList元素数量
   int16 count;
   //存储形式，原生字节数组还是LZF压缩存储
   int2 encoding;
   ...
}

struct quicklistLZF {
   //压缩后的ziplist大小
   unsigned int sz;
   //压缩后的字节数组
   char compressed[];
}
```

**1、quicklist是zipList和linkedList的混合体，linkedList按段切分，每一段使用zipList存储，多个zipList用双向指针串接**

**2、压缩深度默认0，如果为2，则首尾的第一个和第二个zipList不压缩，首尾不压缩便于push、pop**

**3、压缩单个zipList长度超过8k字节，就会新起一个zipList**

**4、考虑到linkedList的prew和next空间占用大，且每个节点是单独分配，内存碎片化，影响内存管理效率**

## 8、redis对象
```
struct redisObject {
   //五大数据类型之一
   unsigned type:4;
   //底层的数据结构
   unsigned encoding:4;
   //指向底层数据结构的指针
   void *ptr;
   int refcount;
   //最后一次访问的时间，用于空闲回收机制
   unsigned lru:22;
}
```

**位段：type [var] : digits  digits表示该位段所占二进制位数，节约空间**

# 五大数据类型使用的底层数据结构
## 1、string

**1、底层使用int｜embstr｜raw**

**2、int为long类型，超过long范围升级为raw**

**3、raw最大长度44字节，保存长字符串，分配两次内存（redisObject、sdshdr）**

**4、embstr最大长度44字节，保存短字符串，分配一次内存，redisObject和sds是连续的**

**增加长度需要重新分配内存，因此设置为只读**

**string最大长度为512M**

## set

**1、底层使用intset｜hashtable**

**2、不可重复的无序集合，数据量大时用hashtable，查询效率高**

**3、集合中所有元素都是整数，且数量不超过512，使用intset**

## zset

**1、底层使用ziplist｜skiplist**

**2、使用ziplist，第一个节点保存元素，第二个保存分数，按分值大小递增排序**

**3、当数量小于128且每个长度小于64字节，使用ziplist**

**4、当使用skiplist时实际zset当结构时这样的**

```
struct zset {
   zskiplist *zsl;
   dict *dict;
}
```

**实际包含一个字典和跳跃表，字典键保存元素的值，值保存元素的分值。跳跃表节点obj保存值，score保存分值，通过指针来共享元素的成员和分值**

**无序字典用于O(1)查找值与分值的对应关系；有序跳跃表用于O(logn)根据分值查询值或范围查找**

## list

**底层使用ziplist｜linkedlist｜quicklist（3.2以后）**

**列表小于512且小于64字节使用ziplist**

## hash

**底层使用ziplist｜hashtable**

**列表长度小于512且小于64字节使用ziplist**
