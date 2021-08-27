---
layout: post
title: Redis数据结构研究
categories: Redis
description: 深入源码结构分析包含跳表、压缩链表、哈希表在内的数据结构及相关的代码定义
keywords: Redis,数据结构
---

## Redis数据结构研究

![Redis-DataStruct-Global-Arch](/images/Redis/Redis-DataStruct-Global-Arch.jpg)

Redis的数据结构总览见上图，Redis的核心就是一个在内存中的哈希表，哈希表的键（Key）是字符串类型，值（Value）有字符串、哈希、列表、集合、有序集合以及BitMap、Geo和HyperLogLog几种类型构成。

所有的键值对都经过散列（hash）后存储在全局哈希表中，全局哈希表使用链地址法解决哈希冲突问题，周期性使用rehash来减少哈希冲突问题。

### 字符串String

字符串在内存的存储方式有三种可能：

1. 对于长度小于21并且可以安全转换为整数类型的，会优先使用int类型进行存储；
2. 对于长度小于44字节的字符串会优先使用embstr存储，这种字符串会一次性分配44字节的空间供当前字符串值使用；
3. 否则使用raw类型存储，也就是简单动态字符串（SDS）。

SDS的存储分为两个部分，即SDS头以及数据部分，两个部分独立分配，不保证二者分配在连续的内存空间中，对SDS进行取值需要经过两次内存查询，第一次获取SDS头，第二次从SDS头中获取实际的字符串；而EMBSTR则将二者的内存分配到了一起，这样充分发挥缓存的作用，进一步减少了取值所需要的时间。

SDS的结构如下图所示（为了逻辑连贯画到了一起，但是实际可能不在一起）：

![Redis-String-SDS-Arch](/images/Redis/Redis-String-SDS-Arch.png)

其中len用来表示字符串的长度，free表示当前空闲的空间的起始点偏移值，buf则指向实际的数据的存储起点，用C语言来表示就是：

```c
struct sdshdr {
    unsigned int len; //buf中已经占用的空间的长度
    unsigned int free;//buf中剩余的可用空间长度
    char buf[];//数据存放位置
};
```

对于EMBSTR而言，它的buf指向的就是下一个字节的地址（因为连续分配）；而对于RAW类型而言，一般来说指向的不是下一个字节（多次分配）。

> EMBSTR长度之所以是44字节是因为：对于EMBSTR而言，默认使用jemelloc分配内存，其内存分配单位是2的幂次，例如2，4，8，16，32，64等。为了保证jemelloc这种分配方式能使EMBSTR的SDSHDR和数据分配在同一个块内，其长度不能超过64字节（类似缓存对齐，必须在同一个缓存行内）。所以假设当前块分配了64字节的空间，其中RedisObject占用16字节，两个uint_8各占用1字节，一个flag占用1字节，剩下的44字节就是EMBSTR的最大长度了。
>
> RedisObject是每个Redis对象都固有的对象头，它和SDS头共同存在，只不过RedisObject存在于所有对象中，而SDSHDR只存在于字符串中，其结构如下：
>
> ```c
> typedef struct redisObject {
>     unsigned type:4;//对象类型（4位=0.5字节）
>     unsigned encoding:4;//编码（4位=0.5字节）
>     unsigned lru:LRU_BITS;//记录对象最后一次被应用程序访问的时间（24位=3字节）
>     int refcount;//引用计数。等于0时表示可以被垃圾回收（32位=4字节）
>     void *ptr;//指向底层实际的数据存储结构，如：SDS等(8字节)
> } robj;
> ```
>
> 而新版本中，SDS的头发生了变化，目的就是为了省下SDS头中两个unsigned int占用的空间，因为很多短字符串根本用不到那么大的空间，假定unsigned int为4字节，其最大能表示的空间为2的32次方-1字节，这个空间太大了，所以新版本对其进行了优化，使用sdshdr8、sdshdr16，sdshdr32，sdshdr64来表示不同大小的SDS：
>
> ```c
> struct __attribute__ ((__packed__)) sdshdr8 {
>     uint8_t len; /* used */
>     uint8_t alloc; /* excluding the header and null terminator */
>     unsigned char flags; /* 3 lsb of type, 5 unused bits */
>     char buf[];
> };
> struct __attribute__ ((__packed__)) sdshdr16 {
>     uint16_t len; /* used */
>     uint16_t alloc; /* excluding the header and null terminator */
>     unsigned char flags; /* 3 lsb of type, 5 unused bits */
>     char buf[];
> };
> struct __attribute__ ((__packed__)) sdshdr32 {
>     uint32_t len; /* used */
>     uint32_t alloc; /* excluding the header and null terminator */
>     unsigned char flags; /* 3 lsb of type, 5 unused bits */
>     char buf[];
> };
> struct __attribute__ ((__packed__)) sdshdr64 {
>     uint64_t len; /* used */
>     uint64_t alloc; /* excluding the header and null terminator */
>     unsigned char flags; /* 3 lsb of type, 5 unused bits */
>     char buf[];
> }
> ```
>
> 这里新的uint8_t占用8位即1个字节，最大能表示64字节字符串，这足够EMBSTR使用了，所以两个uint8_t占用空间下降到2字节，再加上一个1字节的char类型标记，新的SDS头占用3字节的空间，所以 64 - SDSHDR8（3B）- RedisObject（16B）- C语言字符串结束符\0（1B）= 44B，也就是EMBSTR最大的使用空间了。

SDS的优点有：

- 记录了字符串的长度，用O(1)的时间复杂度可以获得字符串的长度。
- 有效的管理字符串所占用的空间，自动扩展空间等。
- 有效的防止内存越界，因为如果空间不够，sds的相关函数会自动扩展空间。
- SDS数组是二进制安全的，会将所有操作视为二进制操作，对输入的数据没有任何要求。

我们可以将字符串的存储结构表示为（不区分EMBSTR和RAW）：

<img src="/images/Redis/Redis-String-Arch.png" alt="Redis-String-Arch" style="zoom:40%;" />

### 哈希表Hash

哈希表的底层会使用两种存储结构，一种是哈希表（HashTable），另一种是压缩列表（Ziplist），下面分别介绍这两种存储结构：

#### Ziplist

链表(list)，哈希(hash)，有序集合(zset)在成员较少，成员值较小的时候都采用压缩链表(ziplist)编码方式进行存储。Ziplist本质上是一个由数组实现的双向链表，其内存空间连续分配。各个数据结构Ziplist的阈值如下：

```c
typedef hash-max-ziplist-entries 512
typedef hash-max-ziplist-value 64
typedef list-max-ziplist-entries 512
typedef list-max-ziplist-value 64
typedef zset-max-ziplist-entries 128
typedef zset-max-ziplist-value 64
```

entries是指存储在ziplist中的键的数量，value是指存储在ziplist中值的大小（单位为字节）。当超过这个阈值之后，ziplist会自动转换为其它数据结构，例如hash会转换为hashtable、list会转换为双向链表、zset会转换为跳表。

我们上面说过ziplist是一个经过特殊编码的双向链表(底层是一个数组)，他设计的目的是为了提高存储效率。但是采用数组存储的很大一个原因就是，如果构造链表结点，那么prev和next指针就要占用额外空间，这不利于块内存连续读取，也就是可能会导致需要频繁的读写缓存，同时也可能会产生内存碎片，所以底层采用数组实现之后天生自带了前后指针，并且按块内存分配，可以充分利用局部性原理，一次读取整个缓存行进行操作，写入时也避免缓存行失效问题。

从最上方架构图可以看出，Redis ziplist的结构图如下：

```c
area        |<---- ziplist header ---->|<----------- entries ------------->|<-end->|
size          4 bytes  4 bytes  2 bytes    ?        ?        ?        ?     1 byte
            +---------+--------+-------+--------+--------+--------+--------+-------+
component   | zlbytes | zltail | zllen | entry1 | entry2 |  ...   | entryN | zlend |
            +---------+--------+-------+--------+--------+--------+--------+-------+
                                       ^                          ^        ^
address                                |                          |        |
                                ZIPLIST_ENTRY_HEAD                |   ZIPLIST_ENTRY_END
                                                                  |
                                                         ZIPLIST_ENTRY_TAIL
 
area        |<------------------- entry -------------------->|
            +------------------+----------+--------+---------+
component   | pre_entry_length | encoding | length | content |
            +------------------+----------+--------+---------+
```

对于一个ziplist，它包含以下内容：

- **zlbytes**：压缩列表的字节长度，占4个字节，因此压缩列表最长(2^32)-1字节；
- **zltail**：压缩列表尾元素相对于压缩列表起始地址的偏移量，占4个字节，即ZIPLIST_ENTRY_TAIL；
- **zllen**：压缩列表的元素数目，占两个字节；支持的最大空间为(2^16)-1字节即65535字节，从上面可知，使用ziplist的list、hash、zset的阈值都小于这个值
- **entryX**：压缩列表存储的若干个元素，可以为字节数组或者整数；entry采用压缩编码，后续会详细讨论压缩方法；
- **zlend**：压缩列表的结尾，占一个字节，恒为**0xFF**，即ZIPLIST_ENTRY_END。

而对于一个entry，它包含以下内容：

- **pre_entry_length**：保存前一个entry所占用的总字节数，这个字段的内容是为了支持从后向前遍历操作，这个字段采用变长编码，长度不固定；
- **encoding**：表示当前元素的编码，即content字段存储的数据类型（整数或者字节数组），采用变长编码；
- **length**：表示当前数据项的长度，采用变长编码；
- **content**：表示实际的数据内容

1. pre_entry_length：它有两种可能，要么是1个字节，或者是5个字节。如果前面一个数据项字节占用小于等于253，那么prevrawlen就占用一个字节；如果前一个entry占用字节数大于等于254，那么第一个字节就是254，后四个字节表示一个整形值，表示prevrawlen的大小。（255不出现在entry的第一个字节，因为它表示结束）

2. 我们把encoding和length合起来一起看，因为它们合并才能表示一个完整的意义：

   1. |00xxxxxx| - 1 byte。第一个字节最高两位是00，剩余的6bit用来表示长度，最多可以表示长度为63的**字节数组**。
   2. 01xxxxxx|xxxxxxxx| - 2 bytes。 第一个字节最高两位是01，剩余的14bit用来表示长度，最多可以表示长度为16383(2^14-1)的**字节数组**。
   3. |10______|xxxxxxxx|xxxxxxxx|xxxxxxxx|xxxxxxxx - 5 bytes。第一个字节最高两位为10，第一字节其它位不可用，后4个字节共32位，可以表示长度为（2^32-1）的字节数组。
   4. |11000000| - 1 byte。 表示后面的data存储2个字节的int16_t类型。
   5. |11010000| - 1 byte。 表示后面的data存储4个字节的int32_t类型。
   6. |11100000| - 1 byte。表示后面的data存储8个字节的int64_t类型。
   7. |11110000| - 1 byte。表示后面的data存储为3个字节长的整数。
   8. |11111110| - 1 byte。表示后面的data存储为1个字节长的整数。
   9. |1111xxxx| - 1byte。这是一种特殊情况，xxxx从1到13一共13个值，就用这13种情况表示真正的数据。（0001-1101一共13个值表示0-12，这种情况下后三个字段合二为一了。）

   可以看出，根据encoding字段第一个字节的前2个比特，可以判断content字段存储的是整数，或者字节数组（以及字节数组最大长度）；当content存储的是字节数组时，后续字节标识字节数组的实际长度；当content存储的是整数时，根据第3、4比特才能判断整数的具体类型；而当encoding字段标识当前元素存储的是0~12的立即数时，数据直接存储在encoding字段的最后4个比特，此时没有content字段。

3. 插入逻辑

在ziplist的entry中插入一段新的数据，会返回一个新的ziplist，替换原来传入的旧的ziplist。因为ziplist是一段连续的地址空间，对他的追加操作，会引发内存的realloc，因此ziplist的内存位置可能会发生变化。

插入操作会因为插入位置的不同而不同，其主要的不同就在于Entry对象保存了前一个对象的长度，这个长度是会发生变化的，设想一下几种情况：

a. 如果插入的位置的数组Ziplist的尾部（包括Ziplist为空的情况），那么无需调整，分配的新空间大小就等于当前Entry大小 + Ziplist头部和尾部空间即可

b. 如果插入的位置不是尾部，那么就要依次递归调整后续Entry的pre_entry_length大小，如果新插入的Entry长度大于253字节，那么后续Entry的pre_entry_length就要由1个字节扩大为5个字节，如果后续Entry扩容后的大小也超过了253字节，那么就要依次递归调整。

删除操作也是同理。**所以Ziplist对于中间插入会导致两个明显缺点：1. 重新分配空间并复制数组带来的开销；2. 连续更新带来的开销**。由于Ziplist没有给出非常明确的结构体实现，如果Entry是以数组形式保存在Ziplist中的，那么获取某个Index处Entry的时间复杂度为O（1）不会带来额外开销（也就是将Entry整个内容封装在一个结构体中，Entry内部包含头部信息和指向实际数据的数组）；如果Entry是以字节数组形式保存在Ziplist中，仅依靠encoding和length来判别长度时，那么需要遍历才能获取到某一个index的Entry，时间复杂度为O（N），会带来额外开销。

Redis源码中关于Ziplist只定义了Entry结构体：

```c
typedef struct zlentry {
    unsigned int prevrawlensize;
    unsigned int prevrawlen;
     
    unsigned int lensize;                                 
    unsigned int len;
            
    unsigned int headersize; 
    
    unsigned char encoding;
      
    unsigned char *p;           
} zlentry;
```

并未明确entry是以字节数组形式组织的还是结构体数组形式组织的。

我们可以将压缩链表的存储结构表示为：

<img src="/images/Redis/Redis-ZipList-Arch.png" alt="Redis-ZipList-Arch" style="zoom:40%;" />

#### HashTable

当哈希表中元素个数超过512或者某个值长度超过64字节时，哈希表会转换为使用HashTable来进行存储。**Redis中的HashTable采用拉链法来解决哈希碰撞问题**。实际上对于HashTable来说，它和一般的哈希表没什么区别，唯一有区别的就在于重散列上。先来看其定义的结构图：

<img src="/images/Redis/Redis-HashTable-Arch.png" alt="Redis-HashTable-Arch" style="zoom:50%;" />

可以看到，和普通的哈希表没有太多差异，无非就是每一级都加上了若干个头部元素，以及实际上存在两个table，一个用于实际存储，另一个用于rehash使用。

每个结构的代码定义如下：

1. HashTable（dict）：

   ```c
   typedef struct dict {
       // 类型相关操作，包括计算哈希值、键赋值等操作
       dictType *type;
       // 类型私有数据
       void *privdata;
       // 实际存储数据的哈希表以及一个重散列使用的哈希表
       dictht ht[2];
       // 重散列索引，记录了当前散列的进度，如果没有进行重散列，那么这个值为-1
       long rehashidx;
       unsigned long iterators;
   } dict;
   ```

   - type属性和privdata属性是针对不同类型的键值对，为创建多态字典而设置的

     **type**属性是指向dictType结构的指针，每个dictType结构保存了用于操作特定类型键值对的函数，redis会为用途不同的字典设置不同的类型特定函数

     **privdata**属性则保存了需要传给那些类型特定函数的可选参数

   - **ht**属性是一个包含两个项的数组，数组中的每个项都是一个dictht哈希表字典只使用ht[0]哈希表，ht[1]哈希表只会在对ht[0]哈希表进行rehash时使用

   - 除了ht[1]之外另一个和rehash有关的属性就是**rehashidx**它记录了rehash目前的进度，如果目前没有进行rehash那么它的值为-1

2. dictType：

   ```c
   typedef struct dictType {
       // 计算哈希值的函数
       uint64_t (*hashFunction)(const void *key);
       // 复制键的函数
       void *(*keyDup)(void *privdata, const void *key);
       // 复制值的函数
       void *(*valDup)(void *privdata, const void *obj);
       // 对比键的函数
       int (*keyCompare)(void *privdata, const void *key1, const void *key2);
       // 销毁键的函数
       void (*keyDestructor)(void *privdata, void *key);
       // 销毁值的函数
       void (*valDestructor)(void *privdata, void *obj);
   } dictType;
   ```

   dictType会由Redis根据操作数据的不同封装好后提供，使用时只要调用对应的功能即可，目的也就是实现多态性。

3. dictEntry：

   ```c
   typedef struct dictEntry {
       // 键
       void *key;
       // 值
       union {
           void *val;
           uint64_t u64;
           int64_t s64;
           double d;
       } v;
       // 拉链法的下一个结点
       struct dictEntry *next;
   } dictEntry;
   ```

   - key 属性保存着键值对中的键、而v属性保存着键值对中的值，其中键值对中的值可以为指针 、无符号64位整数、64位整数，val指针可以指向字符串、列表、字典、集合等所有支持的数据结构；

   - next 属性指向另一个哈希表节点的指针(next 是为了解决哈希冲突,将多个哈希值相同的键值对链接在一起)；

4. dictht

   ```c
   typedef struct dictht {
       // 指向桶数组，桶数组的每个元素是一个dictEntry类型的指针
       dictEntry **table;
       // 当前桶数组的长度，一定是2的幂次
       unsigned long size;
       // 掩码，和Java中的哈希表一样，恒等于 size - 1，计算余数时直接使用 hash & sizemask 即可
       unsigned long sizemask;
       // 已使用的桶空间，目的是为了扩容或缩容
       unsigned long used;
   } dictht;
   ```

有关哈希表的操作没什么好说的，无非就是先计算hash，然后判断桶数组是否有元素，如果没有就直接挂上当前结点，否则就用拉链法解决哈希冲突问题。这里着重讨论一下rehash的过程，Redis采用的是**渐近式哈希**的方法：

随着操作的不断执行，哈希表保存的键值对会逐渐地增多或减少，为了让哈希表的负载因子维持在一个合理的

范围内当哈希表保存的键值对数量太多或者太少时程序需要对哈希表的大小进行相应的扩展或者收缩



Redis对字典的哈希表执行rehash的步骤如下;

为字典ht[1]哈希表分配空间，这个哈希表的空间大小取决于要执行的操作以及ht[0]当前包含的键值对数量（ht[0].used）

**扩展时：ht[1]的大小为第一个大于等于ht[0].used2倍的2的n次幂**

**收缩时：ht[1]的大小为第一个大于等于ht[0].used的2的n次幂**

将保存在ht[0]中的所有键值对rehash到ht[1]上面，rehash指的是重新计算键的哈希值和索引值, 然后将键值对放置到ht[1]哈希表的指定位置上

当ht[0]包含的所有键值对都迁移到了ht[1]之后释放ht[0]将ht[1]设置为ht[0], 并在ht[1]新创键一个空白哈希表为下一次rehash做准备。

为了避免rehash对服务器性能造成影响，**服务器不是一次性将ht[0]里面的所有键值对全部rehash到ht[1]**，而是分多次将ht[0]里面的键值对慢慢地rehash到ht[1]。

当rehash操作开始时会将rehashidx改成0，在渐进式rehash的过程**「更新、删除、查询会在ht[0]和ht[1]中都进行」**，比如更新一个值先更新ht[0]，然后再更新ht[1]。

而新增操作直接就新增到ht[1]表中，ht[0]不会新增任何的数据，这样保证**「ht[0]只减不增，直到最后的某一个时刻变成空表」**，这样rehash操作完成，当某个桶的rehash操作完成后，rehashidx会+1，表示下一次要操作的桶的序号（同时也可以指示hash值在rehashidx之前的，可以直接到ht[1]中寻找，不必寻找ht[0]了）。

### 列表List

#### 双向链表

List会使用两种存储结构，一种是压缩链表Ziplist，上面已经说过了，在list长度小于512字节且每个元素大小小于64字节时默认使用Ziplist；另一种就是双向链表。双向链表是一种非常简单的数据结构，这里不多说，下面是链表每个结点的定义：

```c
typedef struct listNode {
    struct listNode *prev;        // 前驱
    struct listNode *next;        // 后继
    void *value;                  // 当前节点值
} listNode;
```

链表每个结点是典型的前驱、后继和值类型（void*可以代表任何类型），整个表结构如下：

```c
typedef struct list {
    listNode *head;                           // 头结点
    listNode *tail;                           // 尾节点
    void *(*dup)(void *ptr);                  // 复制函数
    void (*free)(void *ptr);                  // 释放函数
    int (*match)(void *ptr, void *key);       // 匹对函数
    unsigned long len;                        // 节点数量
} list;
```

非常显而易见的结构，非常简单，查找时间复杂度为O（N），下面用一张图来表示整个双向链表结构：

<img src="/images/Redis/Redis-List-Arch.png" alt="Redis-List-Arch" style="zoom:100%;" />

### 集合Set

集合采用两种存储结构，**当集合中只包含整数时使用整数数组intset，否则使用哈希表hashtable进行存储**。哈希表前面已经分析过了，这里分析intset的数据结构。

#### 整数数组intset

intset用于保存整数值的数据结构类型，它可以保存`int16_t`、`int32_t` 或者`int64_t` 的整数值，**整数数组内的元素是有序的**，不可重复，intset和EMBSTR一样，内存连续，就像数组一样。

intset的定义如下：

```c
typedef struct intset {
    uint32_t encoding;  // 编码类型 int16_t、int32_t、int64_t
    uint32_t length;    // 长度 最大长度:2^32 - 1
    int8_t contents[];  // 字节数组
} intset;
```

这里比较有意思的一点就是contents数组，数组类型虽然是int8_t，但是它实际上只表示一个字节的数据，并不表示一个数，也就是说，对于int16_t而言，一个数占用contents数组两位的空间，以此类推。数组的最大长度就是length能表示的范围，即无符号32位整数能表达的范围即2^32 - 1。

由于intset的结构和前面的SDS很类似，这里就不画图了，简单解释一下这个编码encoding：Redis源码中为encoding设计了三个宏变量用来对应int16_t、int32_t和int64_t三种数据类型：

```c
// 数据以int16_t类型存放，每个占2个字节，能存放-32768~32767范围内的整数
#define INTSET_ENC_INT16 (sizeof(int16_t)) 
// 数据以int32_t类型存放，每个占4个字节，能存放-2^32-1~2^32范围内的整数
#define INTSET_ENC_INT32 (sizeof(int32_t)) 
// 数据以int64_t类型存放，每个占8个字节，能存放-2^64-1~2^64范围内的整数
#define INTSET_ENC_INT64 (sizeof(int64_t)) 
```

很显然，intset支持的最小整数长度为16字节，也就是说contents数组中至少2个元素相连才能表示一个真正的整数。而length 字段用来保存集合中元素的个数。contents数组中的元素要求**不含有重复的整数且按照从小到大的顺序排列**。在读取和写入的时候，均按照指定的 encoding 编码模式读取和写入。

对于intset来说，它的查询采用的是二分查找（因为有序），时间复杂度为O（logn），插入操作和SDS一样会重新分配内存、拷贝数组。和SDS插入操作一样的是，intset在插入时也可能会有额外操作：（https://zhuanlan.zhihu.com/p/144210551）

假设当前intset中存放了3个整数1、3、10，此时intset的encoding为INTSET_ENC_INT16，当我们要插入整数65535时发现，int16_t编码无法表示这个整数，所以要将数组的编码升级为**int32_t**，这个升级不仅仅只是修改encoding字段这么简单，contents数组中原有的整数也要全部扩容为使用int32_t编码存储，这就导致了了原本占用2个字节的整数1、3、10都会被扩容为使用4个字节的int32_t编码，如下示意图所示：

![Redis-Intset-Upgrade-Process](/images/Redis/Redis-Intset-Upgrade-Process.jpeg)

要注意的是encoding只支持扩容，不支持缩容，如果删除了65535之后，前面的1、3、10还会使用int32_t编码存储。但是由于整个intset数组是有序的，它的很多操作都可以利用有序这个特点，所以整体时间复杂度不高，是一个比较高效的数据结构。

### 有序集合ZSet

#### 跳表Skiplist

> 引用http://zhangtielei.com/posts/blog-redis-skiplist.html关于跳表数据结构的论述

跳表实际上就是在有序链表的基础上形成的一种新的数据结构，对于有序链表而言，查找一个元素的时间复杂度为O(n)，如下图所示：

![Redis-SkipList-SortedList-Arch](/images/Redis/Redis-SkipList-SortedList-Arch.png)

一种优化的思路就是在链表上增加一层，让它一次跳过若干个结点而不只是一个结点，就形成了下面这种结构：

![Redis-SkipList-Dual-Level-Sorted-List-Arch](/images/Redis/Redis-SkipList-Dual-Level-Sorted-List-Arch.png)

这样所有新增加的指针连成了一个新的链表，但它包含的节点个数只有原来的一半（上图中是7, 19, 26）。现在当我们想查找数据的时候，可以先沿着这个新链表进行查找。当碰到比待查数据大的节点时，再回到原来的链表中进行查找。比如，我们想查找23，查找的路径是沿着下图中标红的指针所指向的方向进行的：

![Redis-SkipList-Dual-Level-Sorted-List-Search-Arch](/images/Redis/Redis-SkipList-Dual-Level-Sorted-List-Search-Arch.png)

- 23首先和7比较，再和19比较，比它们都大，继续向后比较。
- 但23和26比较的时候，比26要小，因此回到下面的链表（原链表），与22比较。
- 23比22要大，沿下面的指针继续向后和26比较。23比26小，说明待查数据23在原链表中不存在，而且它的插入位置应该在22和26之间。

在这个查找过程中，由于新增加的指针，我们不再需要与链表中每个节点逐个进行比较了。需要比较的节点数大概只有原来的一半。

就按照这个思路，我们可以逐层添加，上面每一层链表的节点个数，是下面一层的节点个数的一半，这样查找过程就非常类似于一个二分查找，使得查找的时间复杂度可以降低到O(log n)。但是这种方法的一个致命缺陷就是，新插入一个节点之后，就会打乱上下相邻两层链表上节点个数严格的2:1的对应关系。如果要维持这种对应关系，就必须把新插入的节点后面的所有节点（也包括新插入的节点）重新进行调整，这会让时间复杂度重新蜕化成O(n)。删除数据也有同样的问题。

skiplist为了避免这一问题，**它不要求上下相邻两层链表之间的节点个数有严格的对应关系**，而是为每个节点随机出一个层数(level)。比如，一个节点随机出的层数是3，那么就把它链入到第1层到第3层这三层链表中。为了表达清楚，下图展示了如何通过一步步的插入操作从而形成一个skiplist的过程：

![Redis-SkipList-Insert](/images/Redis/Redis-SkipList-Insert.png)

从上面skiplist的创建和插入过程可以看出，每一个节点的层数（level）是随机出来的，而且新插入一个节点不会影响其它节点的层数。因此，插入操作只需要修改插入节点前后的指针，而不需要对很多节点都进行调整。这就降低了插入操作的复杂度。

此时查找的过程和上面列出的一致，**整个数据结构的平均时间复杂度为O(logn)**。

而Redis中的ZSet实际上由三种数据结构组成，当ZSet中数据较少时，使用压缩链表(ziplist)；当ZSet中数据多时，采用**dict + skiplist**的组合，其中dict用来保存数据到分数的映射关系，skiplist用来组织数据。

很明显单纯对于分数的查询行为是依靠dict实现的，而根据分数查询数据（包括范围查询等等）则利用skiplist实现。dict的结构已经说过就是哈希表HashTable，这里不再重复，我们直接来看SkipList数据结构的定义：

首先是整个ZSet的定义：

```c
typedef struct zset {
    dict *dict;
    zskiplist *zsl;
} zset;
```

从这里可以看出，ZSet同时使用了dict和skiplist两种数据结构来进行存储，同时，我们在Ziplist一节中也知道，当ZSet中元素少于128个或值小于64字节时，默认使用的是压缩列表Ziplist。我们接着看关于Skiplist的定义：

```c
#define ZSKIPLIST_MAXLEVEL 32
#define ZSKIPLIST_P 0.25

typedef struct zskiplistNode {
    robj *obj;                            // 数据域
    double score;                         // 分数
    struct zskiplistNode *backward;       // 指向前驱结点
    struct zskiplistLevel {               // 每一个结点的层级
        struct zskiplistNode *forward;    // 指向后继结点
        unsigned int span;                // 某一层距离下一个结点的跨度
    } level[];  						  // level本身是一个柔性数组，最大值为32，由 ZSKIPLIST_MAXLEVEL 定义
} zskiplistNode;

typedef struct zskiplist {
    struct zskiplistNode *header, *tail;  //头部和尾部
    unsigned long length;                 //长度，即一共有多少个元素
    int level;                            //最大层级，即跳表目前的最大层级
} zskiplist;
```

zskiplist的节点定义是结构体zskiplistNode，其中有以下字段。

- **obj**：存放该节点的数据。这个数据类型一定是字符串的SDS，即使插入的是整数也会被编码为SDS类型。

- **score**：数据对应的分数值，zset通过分数对数据升序排列。

- **backward**：指向前驱结点的指针。节点只有1个前向指针，所以**只有第1层链表是一个双向链表**。

- **level[]**：结构体zskiplistLevel的数组，表示跳表中的一层。每层又存放有两个字段：

  - **forward**是指向链表下一个节点的指针。从这里可以看出来，后续每一层都只有一个后向指针是单向链表
  - **span**它表示当前的指针跨越了多少个节点，用于计算元素排名。

  level[]是一个柔性数组，因此它占用的内存不在zskiplistNode结构里面，而需要插入节点的时候单独为它分配。也正因为如此，skiplist的每个节点所包含的指针数目才是不固定的，

zskiplist就是跳表本身，其中有以下字段。

- **header、tai**l：头指针和尾指针。
- **length**：跳表的长度，不包括头指针。
- **level**：跳表的层数。

我们用下面这张图来表示Skiplist的结构，为了简化图示，这里省略了RedisObject的内容:

![Redis-Skiplist-Status-Process](/images/Redis/Redis-Skiplist-Status-Process.png)

看上去很复杂，但是仔细分析一下并不复杂，首先Skiplist的header指针指向一个头结点，这个头结点没有值，只记录level，其中level[0]就是最下一层，接着向上是L1、L2一直到L32，这个level数组的大小是另外管理的，最大值由`ZSKIPLIST_MAXLEVEL`定义，为32。

所有结点都可以通过第0层访问到，第0层的所有结点都有一个backward指针用来指向前一个元素，而有一部分元素可以通过第1和第2层访问到，这样用level数组可以实现分层访问，每个实际结点都有一个level数组，如果当前这个结点是第1层的元素（如上图Ada和Kevin对应结点），那么可以通过访问level层对应的下标结点来实现快速访问，但是在0层以上的结点不存储前驱结点的指针（即可以通过Ada找到Kevin，但是Kevin结点只能找到直接前驱Tom，而无法回头去找Ada）。

当我们要查询一个结点时，例如Tom，那么我们在头结点中先访问最上层的结点，即Skiplist中存储的level-1下标的层，这里是第2层，所以从头结点中的level[2]开始访问，此时访问到的第一个结点是Ada结点，分数为78.0，小于Tom结点的分数87.5（分数和节点的对应关系在dict中可以找到，是数据到分数的映射），所以此时沿着Ada结点的L2继续向后找，发现已经是NULL了，此时从Ada的L1向后找，找到的Kevin分数为87.5但不是我们要的数据，此时从Kevin结点沿着backward向前就能找到Tom结点了。

插入过程：（引用https://www.jianshu.com/p/09c3b0835ba6）

![Redis-Skiplist-Insert-Process](/images/Redis/Redis-Skiplist-Insert-Process.webp)

流程如下：

1. 按照前面讲过的查找流程，找到合适的插入位置。注意zset允许分数score相同，这时会根据节点数据obj的字典序来排序。

2. 调用zslRandomLevel()方法，随机出要插入的节点的层数。

3. 调用zslCreateNode()方法，根据层数level、分数score和数据obj创建出新节点。

4. 每层遍历，修改新节点以及其前后节点的前向指针forward和跳跃长度span，也要更新最底层的后向指针backward。

以上就是Skiplist的整个数据结构了，Skiplist的平均时间复杂度为O(logn)，最坏时间复杂度为O(n).

这里引用一下作者关于在Redis中使用跳表而不是平衡二叉树的原因：

> 1）它们不是很耗费内存。这基本上取决于你。改变关于一个节点拥有给定层数的概率的参数将使其比btrees的内存密集度低。
>
> 2）一个排序的集合通常是许多ZRANGE或ZREVRANGE操作的目标，也就是说，把跳表作为一个链表进行遍历。在这种操作下，跳过列表的缓存定位性至少与其他类型的平衡树一样好。
>
> 3）它们在实现、调试等方面都比较简单。例如，由于跳表的简单性，我收到了一个补丁（已经在Redis主程序中），其中的增强型跳表实现了ZRANK的O（log(N)）。

