

# 前言

好像目录展示不出来…… 将就的看把

# Redis数据结构

[TOC]



## 简单动态字符串 SDS

SDS又称为简单动态字符串（Simple Dynamic String）。它的结构如下

```c
struct sdshdr {
    // 记录buf中已经使用字节的数量
    // 相当于SDS所保存字符串的长度
    int len;
    // 记录buf中未使用字节的数量
    int free;
    // 字节数组，用于保存字符串
    char buf[];
}
```



- buf 相当于C中的字符串，一般称之为字节数组，为一个char类型的数组。
- len记录了buf的长度，不计算最后一个'/0'
  - 该记录使查询字符串长度的时间复杂度变为了O(1)
- free中保存了额外申请的空间
  - 在进行字符串拼接的时候
    - 如果空间充足，会直接使用之前申请的额外空间
    - 如果空间不充足（free < 申请拼接的字符串的长度），那么会申请额外的空间。如果所需要的小于1m，那么直接申请字符串长度的空间（如申请13个字符，则free也为13）。如果大于1M，那么直接申请1M。
  - 在进行字符串裁剪时
    - 多余的空间会存到free中，以便下次使用





## ~~链表（Redis 3.2以后已经不再使用）~~

 不赘述

**Redis中链表的特性**

- **双向链表**，能够很方便地获取一个节点的前驱节点或后继节点
- **带头尾指针**，list中的head与tail分别保存了链表的头结点和尾节点
- **获取长度方便**，list中的len属性使得获取链表长度的时间复杂度变为了O(1)
- **多态**，链表节点使用void *指针来保存节点值，并且可以通过list结构的dup、free、match三个属性为节点值设置类型特定函数，所以链表可以用于保存各种不同类型的值



## 字典

Redis的字典使用**哈希表**作为底层实现，一个哈希表里面可以有**多个**哈希表节点，而每个哈希表节点就保存了字典中的**一个**键值对



### **哈希表**

Redis中的哈希表的实现如下：

```c
typedef struct dictnt {
    // 哈希数组 
    // 类似java中的HashMap
    // transient Node<K,V>[] table;
    dictEntry **table;
    
    // 哈希表的大小
    unsigned long size;
    
    // 哈希表的掩码 大小为size - 1
    unsigned long sizemask;
    
    //哈希表中已有的节点数
    unsigned long used;
} dictnt;
```

- table为一个dictEntry类型的数组
  - 每个dictEntry中保存了一个键值对
- size记录了哈希表的大小
- sizemask为size-1，用于哈希计算，决定一个键应该被放到哪个桶中
- used记录了哈希表目前已有节点（**键值对**）的数量

![](http://badwomen.asia/202111302330.png)

### **哈希节点**

Redis中的哈希节点的实现如下：

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
    // 指向下一个哈希节点，形成链表
    struct dictEntry *next;
} dictEntry;
```

类似于Java中HashMap的Node

```java
static class Node<K,V> implements Map.Entry<K,V> {
    final int hash;
    final K key;
    V value;
    Node<K,V> next;
    
	// 方法省去
    ...
}
```

- key保存了键值对中 键的值
- value保存了键值对中 值的值，其中值可以为指针类型(*val)，uint64_t、int64_t和double
- next用于解决哈希冲突，使用拉链法

### **字典**

Redis中的字典实现如下

```c
typedef struct dict {
    dictType *type;
    void *privdata;
    dictht ht[2];
    long rehashidx; /* rehashing not in progress if rehashidx == -1 */
    unsigned long iterators; /* number of iterators currently running */
} dict;
```

- type属性是一个指向**dictType**结构的指针，每个dictType结构保存了一簇用于操作特定类型键值对的函数，Redis会为用途不同的字典设置不同的类型特定函数
- 而privdata属性则保存了需要传给那些类型特定函数的可选参数

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

- ht属性为包含了两个ditht元素的数组
  - 一般情况下，只是用ht[0]作为哈希表，ht[1]只会在对ht[0]进行rehash时才会使用
- rehashidx是除了ht[1]以外，另一个与rehash有关的属性，它**记录了rehash目前的进度**，如果没有rehash，那么它的值为-1

**一个普通状态下（未进行rehash）的字典如下图所示**

![](http://badwomen.asia/202112011234.png)

### rehash

随着操作的不断执行，哈希表保存的键值对会逐渐地增多或者减少，为了让哈希表的负载因子（load_factor）维持在一个合理的范围之内（可以减少出现哈希冲突的几率），当哈希表保存的键值对数量太多或者太少时，程序需要对哈希表的大小进行相应的**扩展或者收缩**。

扩展和收缩哈希表的工作可以通过执行**rehash（重新散列）**操作来完成，Redis对字典的哈希表执行rehash的步骤如下：

- 为字典的ht[1]哈希表分配空间，这个哈希表的空间大小取决于要执行的操作，以及ht[0]当前包含的键值对数量（dictht.used的大小）

  - 如果执行的是**扩展操作**，那么ht[1]的大小为**第一个**大于ht[0].used * 2 的 2n （和Java 中的 HashMap一样，这样可以保证sizemask的值必定为11…11）

  - 如果执行的是

    收缩操作，那么ht[1]的大小为第一个小于ht[0].used的 2^n

    - 注：Redis中的字典是有**缩容**操作的，而Java中的HashMap没有缩容操作

- 将保存在ht[0]中的所有键值对rehash到ht[1]上面

  - rehash指的是重新计算键的哈希值和索引值，然后将键值对放置到ht[1]哈希表的指定位置上

- 当ht[0]包含的所有键值对都迁移到了ht[1]之后（ht[0]变为空表），释放ht[0]，将ht[1]设置为ht[0]，并在ht[1]新创建一个空白哈希表，为下一次rehash做准备

  - 上面有两步有点像垃圾回收算法中的**标记-复制算法**（FROM-TO，然后交换FROM 和 TO）



**哈希表的扩展与收缩**

当以下条件中的任意一个被满足时，程序会自动开始对哈希表执行扩展操作

- 服务器目前**没有在执行**BGSAVE命令或者BGREWRITEAOF命令，并且哈希表的负载因子大于等于1

- 服务器目前**正在执行**BGSAVE命令或者BGREWRITEAOF命令，并且哈希表的负载因子大于等于5

  - 负载因子的计算方法如下

  ```
  // 负载因子的计算
  load_factory = ht[0].used/ht[0].size
  ```

根据BGSAVE命令或BGREWRITEAOF命令是否正在执行，服务器执行扩展操作所需的负载因子并不相同，这是因为在执行BGSAVE命令或BGREWRITEAOF命令的过程中，Redis需要创建当前服务器进程的子进程，而大多数操作系统都采用写时复制（copy-on-write）技术来优化子进程的使用效率，所以在子进程存在期间，服务器会提高执行扩展操作所需的负载因子，从而**尽可能地避免在子进程存在期间进行哈希表扩展操作，这可以避免不必要的内存写入操作，最大限度地节约内存**。

另一方面，**当哈希表的负载因子小于0.1时，程序自动开始对哈希表执行收缩操作**。



### 渐进rehash

扩展或收缩哈希表需要将ht[0]里面的所有键值对rehash到ht[1]里面，但是，**这个rehash动作并不是一次性、集中式地完成的，而是分多次、渐进式地完成的。**这样做主要因为在数据量较大时，如果一次性，集中式地完成，庞大的计算量可能会导致服务器在一段时间内停止服务。

**详细步骤**

- 为ht[1]分配空间，让字典同时持有ht[0]和ht[1]两个哈希表
- 在字典中维持一个索引计数器变量rehashidx，并将它的值设置为0，表示rehash工作正式开始
  - 索引计数器rehashidx类似程序计数器PC，用于保存进行rehash的进度（rehash到哪个索引了）
- 在rehash进行期间，每次对字典执行添加、删除、查找或者更新操作时，程序除了执行指定的操作以外，还会顺带将ht[0]哈希表在rehashidx索引上的键值对rehash到ht[1]，当rehash工作完成之后，程序将rehashidx属性的值增一（指向下一个索引）
- 随着字典操作的不断执行，最终在某个时间点上，ht[0]的所有键值对都会被rehash至ht[1]，这时程序将rehashidx属性的值设为-1，表示rehash操作已完成



因为在进行渐进式rehash的过程中，**字典会同时使用ht[0]和ht[1]两个哈希表**，所以在渐进式rehash进行期间，字典的删除（delete）、查找（find）、更新（update）等操作会**在两个哈希表上进行**

 例如，要在字典里面查找一个键的话，程序会先在ht[0]里面进行查找，如果没找到的话，就会继续到ht[1]里面进行查找，诸如此类

另外，在渐进式rehash执行期间，**新添加到字典的键值对一律会被保存到ht[1]里面**，而ht[0]则不再进行任何添加操作，这一措施保证了ht[0]包含的键值对数量会只减不增，并随着rehash操作的执行而最终变成空表

## 跳跃表

### 跳跃表原理

**搜索链表中的元素时，无论链表中的元素是否有序，时间复杂度都为O(n)**，如下图，搜索103需要查询9次才能找到该节点

![1](http://badwomen.asia/redis%E6%95%B0%E6%8D%AE%E7%BB%93%E6%9E%84%E8%B7%B3%E8%A1%A81.png)

但是能够提高搜索的其他数据结构，如：二叉排序树、红黑树、B树、B+树等等的实现又过于复杂。有没有一种相对简单，同时又能提搜索效率的数据结构呢，跳跃表就是这样一种数据结构。

Redis中使用跳跃表好像就是因为一是B+树的实现过于复杂，二是Redis只涉及内存读写，所以最后选择了跳跃表。

#### 跳跃表实现——搜索

为了能够更快的查找元素，我们可以在该链表之上，再添加一个新链表，新链表中保存了部分旧链表中的节点，以加快搜索的速度。如下图所示

![1](http://badwomen.asia/redis%E6%95%B0%E6%8D%AE%E7%BB%93%E6%9E%84%E8%B7%B3%E8%A1%A82.png)

我们搜索元素时，从最上层的链表开始搜索。当找到某个节点大于目标值或其后继节点为空时，从该节点向下层链表搜寻，然后顺着该节点到下一层继续搜索。

比如我们要找103这个元素，则会经历：2->23->54->87->103

这样还是查找了5次，当我们再将链表的层数增高以后，查找的次数会明显降低，如下图所示。3次便找到了目标元素103

![1](http://badwomen.asia/redis%E6%95%B0%E6%8D%AE%E7%BB%93%E6%9E%84%E8%B7%B3%E8%A1%A83.png)

**代码中实现的跳表结构如下图所示**

一个节点拥有多个指针，指向不同的节点

![1](http://badwomen.asia/redis%E6%95%B0%E6%8D%AE%E7%BB%93%E6%9E%84%E8%B7%B3%E8%A1%A84.png)



#### 跳跃表实现——插入

跳跃表的插入策略如下

- 先找到合适的位置以便插入元素

- 找到后，将该元素插入到最底层的链表中，并且

  抛掷硬币（1/2的概率）

  - 若硬币为正面，则将该元素晋升到上一层链表中，**并再抛一次**
  - 若硬币为反面，则插入过程结束

- 为了避免以下情况，需要在每个链表的头部设置一个 **负无穷** 的元素

![1](http://badwomen.asia/redis%E6%95%B0%E6%8D%AE%E7%BB%93%E6%9E%84%E8%B7%B3%E8%A1%A85.png)

![1](http://badwomen.asia/redis%E6%95%B0%E6%8D%AE%E7%BB%93%E6%9E%84%E8%B7%B3%E8%A1%A86.png)

**插入图解**

- 若我们要将45插入到跳跃表中

![1](http://badwomen.asia/redis%E6%95%B0%E6%8D%AE%E7%BB%93%E6%9E%84%E8%B7%B3%E8%A1%A87.png)

- 先找到插入位置，将45插入到合适的位置

![1](http://badwomen.asia/redis%E6%95%B0%E6%8D%AE%E7%BB%93%E6%9E%84%E8%B7%B3%E8%A1%A88.png)

- 抛掷硬币：**为正**，晋升

![1](http://badwomen.asia/redis%E6%95%B0%E6%8D%AE%E7%BB%93%E6%9E%84%E8%B7%B3%E8%A1%A89.png)

- 假设硬币一直为正，插入元素一路晋升，当晋升的次数超过跳跃表的层数时，**需要再创建新的链表以放入晋升的插入元素。新创建的链表的头结点为负无穷**

![1](http://badwomen.asia/redis%E6%95%B0%E6%8D%AE%E7%BB%93%E6%9E%84%E8%B7%B3%E8%A1%A810.png)



MySQL使用B+树的是因为：**叶子节点存储数据，非叶子节点存储索引**，B+树的每个节点可以存储多个关键字，它将节点大小设置为磁盘页的大小，**充分利用了磁盘预读的功能**。每次读取磁盘页时就会读取一整个节点,每个叶子节点还有指向前后节点的指针，为的是最大限度的降低磁盘的IO;因为数据在内存中读取耗费的时间是从磁盘的IO读取的百万分之一

而Redis是**内存中读取数据，不涉及IO，因此使用了跳跃表**

### Redis中的跳跃表

Redis中的sort_set主要由跳表实现，sort_set的添加语句如下

```
zadd key score1 member1 score2 member2 ...
```

Redis中的跳表结构如下

![1](http://badwomen.asia/redis%E6%95%B0%E6%8D%AE%E7%BB%93%E6%9E%84%E8%B7%B3%E8%A1%A811.png)



Redis中的跳表主要由节点**zskiplistNode**和跳表**zskiplist**来实现，他们的源码如下

#### zskiplistNode

```c
typedef struct zskiplistNode {
    // 存储的元素 就是语句中的member
    sds ele;
    
    // 分值,就是语句中的score
    double score;
    
    // 指向前驱节点
    struct zskiplistNode *backward;
    
    // 层，每个节点有1~32个层，除头结点外（32层），其他节点的层数是随机的
    struct zskiplistLevel {
        // 每个层都保存了该节点的后继节点
        struct zskiplistNode *forward;
        
        // 跨度，用于记录该节点与forward指向的节点之间，隔了多少各节点。主要用于计算Rank
        unsigned long span;
    } level[];
} zskiplistNode;
```

**各个属性的详细解释**

- ele：sds变量，保存member。
- score：double变量，用于保存score
  - **注意**：**score和ele共同来决定一个元素在跳表中的顺序**。score不同则根据score进行排序，score相同则根据ele来进行排序
  - **跳表中score是可以相同的，而ele是肯定不同的**
- backward：前驱指针，用于保存节点的前驱节点，**每个节点只有一个backward**

- level[]：节点的层，每个节点拥有1~32个层，除头结点外（32层），其他节点的层数是随机的。**注意**：Redis中没有使用抛硬币的晋升策略，而是直接随机一个层数值。下图展示了层数不同的几个节点

- level：保存了该节点指向的下一个节点，但是不一定是紧挨着的节点。还保存了两个节点之间的跨度
  - forward：后继节点，该节点指向的下一个节点，但是不一定是紧挨着的节点
  - span：跨度，用于记录从该节点到forward指向的节点之间，要走多少步。主要用于计算Rank
    - rank：排位，头节点开始到目标节点的跨度，由沿途的span相加获得

#### **zskiplist**

zskiplist的源码如下

```c
typedef struct zskiplist {
    // 头尾指针，用于保存头结点和尾节点
    struct zskiplistNode *header, *tail;
    
    // 跳跃表的长度，即除头结点以外的节点数
    unsigned long length;
    
    // 最大层数，保存了节点中拥有的最大层数（不包括头结点）
    int level;
} zskiplist;
```

#### 遍历过程

遍历需要访问跳表中的每个节点，直接走底层的节点即可依次访问搜索过程

如我们要访问该跳表中score = 2.0的节点

![1](http://badwomen.asia/redis%E6%95%B0%E6%8D%AE%E7%BB%93%E6%9E%8412.png)

从高层依次向下层遍历

- 头结点的L6~L32的 forward 都为空，从L5开始访问
- 头结点L5的 forward 的指向的node3的score为3.0，小于2.0，返回头结点
- 从头结点L4 的 forward 指向的node1开始访问，节点的score = 1.0，继续向下执行
- 从node1的 L4 开始访问，为node3，返回node1
- 从node1的L3开始访问，为node3，返回node1
- 从node1的L2开始访问，为node2，score = 2.0，即为所要的节点

#### 插入过程

插入节点时，需要找到节点的插入位置。并给节点的各个属性赋值。插入后判断是否需要拓展上层



## 整数集合

### 1、简介

整数集合（intset）是集合键的底层实现之一，**当一个集合只包含整数值元素，并且这个集合的元素数量不多时**，Redis就会使用整数集合作为集合键的底层实现，例如

### 2、整数集合的实现

整数集合（intset）是Redis用于保存整数值的集合抽象数据结构，它可以保存类型为int16_t、int32_t或者int64_t的整数值，并且**保证集合中不会出现重复元素且按照从小到大的顺序排列**

```c
typedef struct intset {
    // 编码方式
    uint32_t encoding;
    // contents数组的长度
    uint32_t length;
    // 保存元素的数组，也就是set集合
    int8_t contents[];
} intset;
```

- length：记录了contents数组中元素的个数
- contents：为intset的底层实现，用于存放不重复的元素，且**元素按照从小到大的顺序排列。数组类型由encoding决定，与int8_t无关**

### 3、升级

每当我们要将一个新元素添加到整数集合里面，并且**新元素的类型比整数集合现有所有元素的类型都要长时**，整数集合需要先进行**升级**（upgrade），然后才能将新元素添加到整数集合里面

#### 具体过程

- 根据新元素的类型，**扩展**整数集合底层数组的空间大小，并为新元素**分配空间**

- 将底层数组(contents[])现有的所有元素都**转换**成与新元素相同的类型，并将类型转换后的元素放置到正确的位上，而且在放置元素的过程中，需要继续维持底层数组的**有序性质不变**

- 将新元素添加到底层数组里面

  - 因为新元素的长度大于数组中所有其他元素的长度，所以

    该元素要么是最小的，要么是最大的

    - 若为最小值，放在数组开头
    - 若为最大值，放在数组末尾

![](http://badwomen.asia/redis%E6%95%B4%E6%95%B0%E9%9B%86%E5%90%88%E6%95%B0%E6%8D%AE%E7%BB%93%E6%9E%841.png)

将数组中的元素**类型改为int32_t**，并放入扩展后的contents中。最后添加新插入的元素。

![](http://badwomen.asia/redis%E6%95%B0%E6%8D%AE%E7%BB%93%E6%9E%84%E6%95%B4%E6%95%B0%E9%9B%86%E5%90%882.png)



### 4、升级的好处

- **自适应**：会根据contents中的元素位数选择最合适的类型，来进行内存的分配
- **节约内存**：基于自适应，不会为一个位数较少的整数分配较大的空间



## 压缩列表

### 1、简介

压缩列表（ziplist）是列表键(list)和哈希键(hash)的底层实现之一

- 当**一个列表键（list）只包含少量列表项，并且每个列表项要么就是小整数值，要么就是长度比较短的字符串**，那么Redis就会使用压缩列表来做列表键的底层实现

- 当**一个哈希键只包含少量键值对，并且每个键值对的键和值要么就是小整数值，要么就是长度比较短的字符串**，那么Redis就会使用压缩列表来做哈希键的底层实现

### 2、压缩列表的组成

压缩列表是Redis为了**节约内存**而开发的，是由一系列特殊编码的连续内存块组成的顺序型（sequential）数据结构。一个压缩列表可以包含任意多个节点（entry），每个节点可以保存**一个字节数组**或者**一个整数值**。具体组成如下

[![img](https://nyimapicture.oss-cn-beijing.aliyuncs.com/img/20201113163516.png)](https://nyimapicture.oss-cn-beijing.aliyuncs.com/img/20201113163516.png)

#### 属性介绍

- zlbytes：表示压缩列表占用的内存（单位：字节）
- zltail：压缩列表起始指针到尾节点的偏移量
  - 如果我们有一个指向压缩列表起始地址的指针p，通过p+zltail就能直接访问压缩列表的最后一个节点
- zllen：压缩列表中的**节点数**
- entry：压缩列表中的节点

### 3、压缩列表中的节点

每个压缩列表节点都由**previous_entry_length、encoding、content**三个部分组成，如下图

[![img](https://nyimapicture.oss-cn-beijing.aliyuncs.com/img/20201113164801.png)](https://nyimapicture.oss-cn-beijing.aliyuncs.com/img/20201113164801.png)

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

#### previous_entry_length

节点的previous_entry_length属性以**字节为单位，记录了压缩列表中前一个节点的长度**，其值长度为1个字节**或者**5个字节

- 如果前一节点的长度小于254字节，那么previous_entry_length属性的长度为1字节
  - 前一节点的长度就保存在这一个字节里面
- 如果前一节点的长度大于等于254字节，那么previous_entry_length属性的长度为5字节
  - 其中属性的第一字节会被设置为0xFE（十进制值254），而之后的四个字节则用于保存前一节点的长度

若前一个节点的长度为5个字节，那么压缩列表的previous_entry_length属性为0x05（1个字节保存长度）

若前一个节点的长度为10086(0x2766)，那么压缩列表中previous_entry_length属性为0xFE00002766（后4个字节保存长度）

通过previous_entry_length属性，可以方便地访问当前节点的前一个节点

#### encoding

节点的encoding属性记录了**节点的content属性所保存数据的类型以及长度**

#### content

节点的content属性负责保存节点的值，**节点值可以是一个字节数组或者整数，值的类型和长度由节点的encoding属性决定**

## 快表

### 1、简介

quicklist是Redis 3.2中新引入的数据结构，**能够在时间效率和空间效率间实现较好的折中**。Redis中对quciklist的注释为A doubly linked list of ziplists。顾名思义，quicklist是一个双向链表，链表中的每个节点是一个ziplist结构。quicklist可以看成是用双向链表将若干小型的ziplist连接到一起组成的一种数据结构。当ziplist节点个数过多，quicklist退化为双向链表，一个极端的情况就是每个ziplist节点只包含一个entry，即只有一个元素。当ziplist元素个数过少时，quicklist可退化为ziplist，一种极端的情况就是quicklist中只有一个ziplist节点

### 2、快表的结构

quicklist是由quicklist node组成的双向链表，quicklist node中又由ziplist充当节点。quicklist的存储结构如图

![](http://badwomen.asia/快表.png)

#### **quicklist**

```c
typedef struct quicklist {
    quicklistNode *head;
    quicklistNode *tail;
    unsigned long count;        /* total count of all entries in all ziplists */
    unsigned long len;          /* number of quicklistNodes */
    int fill : QL_FILL_BITS;              /* fill factor for individual nodes */
    unsigned int compress : QL_COMP_BITS; /* depth of end nodes not to compress;0=off */
    unsigned int bookmark_count: QL_BM_BITS;
    quicklistBookmark bookmarks[];
} quicklist;
```

**head和tail**

- head和tail分别指向快表的首位节点

**count**

- count为quicklist中元素总数

**len**

- len为quicklist Node（节点）个数

**fill**

fill用来指明每个quicklistNode中ziplist长度

#### **quicklistNode**

```c
typedef struct quicklistNode {
    struct quicklistNode *prev;
    struct quicklistNode *next;
    unsigned char *zl;  		 /* 指向压缩列表的指针 */
    unsigned int sz;             /* ziplist size in bytes */
    unsigned int count : 16;     /* count of items in ziplist */
    unsigned int encoding : 2;   /* RAW==1 or LZF==2 */
    unsigned int container : 2;  /* NONE==1 or ZIPLIST==2 */
    unsigned int recompress : 1; /* was this node previous compressed? */
    unsigned int attempted_compress : 1; /* node can't compress; too small */
    unsigned int extra : 10; /* more bits to steal for future usage */
} quicklistNode;
```

**prev和next**

- 因为quicklist为双向链表，所以有prev和next指针，分别指向前驱节点和后继节点

**zl**

- zl指向该节点对应的**ziplist结构**

**encoding**

- encoding代表采用的编码方式
  - 1代表是原生的ziplist（未进行再次压缩）
  - 2代表使用LZF进行压缩

**container**

- container为quicklistNode节点zl指向的容器类型
  - 1代表none
  - 2代表使用ziplist存储数据

**recompress**

- recompress代表这个节点之前是否是压缩节点，若是，则在使用压缩节点前先进行解压缩，使用后需要重新压缩，此外为1，代表是压缩节点

**attempted_compress**

- attempted_compress测试时使用

**extra**

- extra为预留

#### quickLZF

**quicklist允许ziplist进行再次压缩**。当我们对ziplist利用LZF算法进行压缩时，quicklistNode节点指向的结构为**quicklistLZF**。其中sz表示compressed所占字节大小，quicklistLZF结构如下所示

```c
typedef struct quicklistLZF {
    unsigned int sz; /* LZF size in bytes*/
    char compressed[];
} quicklistLZF;
```

#### **quicklistEntry**

当我们使用quicklistNode中**ziplist中的一个节点**时，Redis提供了quicklistEntry结构以便于使用，该结构如下

可以理解为其为**ziplist中的一个节点**，只不过记录了更详细的信息

```c
typedef struct quicklistEntry {
    // 指向当前元素所在的quicklist
    const quicklist *quicklist;
    
    // 指向当前元素所在的quicklistNode结构
    quicklistNode *node;
    
    // 指向当前元素所在的ziplist
    unsigned char *zi;
    
    // 指向该节点的字符串内容
    unsigned char *value;
    
    // 该节点的整型值
    long long longval;
    
    // 该节点的大小
    unsigned int sz;
    
    // 该节点相对于整个ziplist的偏移量，即该节点是ziplist第多少个entry
    int offset;
} quicklistEntry;
```

#### 初始化

初始化是构建quicklist结构的第一步，由quicklistCreate函数完成，该函数的主要功能就是初始化quicklist结构。初始化后的quicklist如下图所示

[![img](http://badwomen.asia/20201117210043.png)](https://nyimapicture.oss-cn-beijing.aliyuncs.com/img/20201117210043.png)

#### 插入操作

插入操作有

- 插入quicklist node
- 插入ziplist中的节点

插入时可以选择头插和尾插，对应list的lpush和rpush，底层调用的是quicklistPushHead与quicklistPushTail方法

- quicklistPushHead的**基本思路**是：查看quicklist原有的head节点是否可以插入，如果可以就直接利用ziplist的接口进行插入，否则新建quicklistNode节点进行插入。函数的入参为待插入的quicklist，需要插入的数据value及其大小sz；函数返回值代表是否新建了head节点，0代表没有新建，1代表新建了head

当quicklist中只有一个节点时，其结构如下图所示

[![img](http://badwomen.asia/20201117210218.png)](https://nyimapicture.oss-cn-beijing.aliyuncs.com/img/20201117210218.png)

具体的插入（zlentry）情况如下

- 当前插入位置所在的quicklistNode仍然**可以继续插入**，此时可以直接插入

- 当前插入位置所在的quicklistNode

  不能继续插入

  ，此时可以分为如下几种情况

  - 需要向当前quicklistNode第一个元素（entry1）前面插入元素，当前ziplist所在的quicklistNode的**前一个**quicklistNode可以插入，则将数据插入到前一个quicklistNode。如果**前一个quicklistNode不能插入**（不包含前一个节点为空的情况），则**新建**一个quicklistNode插入到当前quicklistNode**前面**
  - 需要向当前quicklistNode的最后一个元素（entryN）后面插入元素，当前ziplist所在的quicklistNode的**后一个**quicklistNode可以插入，则直接将数据插入到后一个quicklistNode。如果**后一个quicklistNode不能插入**（不包含为后一个节点为空的情况），则**新建**一个quicklistNode插入到当前quicklistNode的**后面**
  - **不满足前面2个条件的所有其他种情况**，将当前所在的quicklistNode以当前待插入位置为基准，拆分成左右两个quicklistNode，之后将需要插入的数据插入到其中一个拆分出来的quicklistNode中

#### 查找操作

quicklist查找元素主要是针对index，即通过元素在链表中的下标查找对应元素。基本思路是，**首先找到index对应的数据所在的quicklistNode节点，之后调用ziplist的接口函数ziplistGet得到index对应的数据**。简而言之就是：定位quicklistNode，再在quicklistNode中的ziplist中寻找目标节点



## 对象

### 1、简介

#### 基本数据结构与对象的关系

Redis**并没有直接使用**简单动态字符串（SDS）、双端链表、字典、压缩列表、整数集合等等这些数据结构来实现键值对数据库，而是基于这些数据结构创建了一个对象系统，这个系统包含**字符串对象、列表对象、哈希对象、集合对象和有序集合对象**这五种类型的对象

#### 使用对象的好处

- 通过这五种不同类型的对象，Redis可以在执行命令之前，根据对象的类型来判断一个对象是否可以执行给定的命令
- 我们可以针对不同的使用场景，为对象设置多种不同的数据结构实现，从而**优化对象在不同场景下的使用效率**

#### 对象的回收——引用计数法

Redis的对象系统还实现了基于**引用计数**技术的内存回收机制

Redis还**通过引用计数技术实现了对象共享机制**，这一机制可以在适当的条件下，通过让多个数据库键共享同一个对象来节约内存

### 2、对象的类型与编码

Redis使用对象来表示数据库中的键和值，每次当我们在Redis的数据库中新创建一个键值对时，我们**至少会创建两个对象**，一个对象用作键值对的键（键对象），另一个对象用作键值对的值（值对象），如

```
set hello "hello world"
```

其中键值对的**键**是一个包含了字符串值”hello”的对象，而键值对的**值**则是一个包含了字符串值”hello world”的对象

Redis中的每个对象都由一个**redisObject**结构表示，该结构中和保存数据有关的三个属性分别是type属性、encoding属性和ptr属性

```c
typedef struct redisObject {
    // 类型(对象类型)
    unsigned type:4;
    // 编码(对象底层使用的数据结构)
    unsigned encoding:4;
    // 指向底层数据结构的指针
    void *ptr;
    
    ....
        
} robj;
```

#### 类型

对象的type属性记录了对象的类型，这个属性的值可以是下标所示的值

| 类型常量     | 对象名称     |
| ------------ | ------------ |
| REDIS_STRING | 字符串对象   |
| REDIS_LIST   | 列表对象     |
| REDIS_HASH   | 哈希对象     |
| REDIS_SET    | 集合对象     |
| REDIS_ZSET   | 有序集合对象 |

对于Redis数据库保存的键值对来说，**键总是一个字符串对象**，而值**则可以是字符串对象、列表对象、哈希对象、集合对象或者有序集合对象的其中一种**，因此

- 当我们称呼一个数据库键为**字符串**键时，我们指的是这个数据库键所对应的**值为字符串对象**
- 当我们称呼一个键为**列表键**时，我们指的是这个数据库键所对应的值为**列表对象**

#### 编码

对象的ptr指针指向对象的底层实现数据结构，而这些数据结构由对象的encoding属性决定

**encoding属性记录了对象所使用的编码**，也即是说这个对象使用了什么数据结构作为对象的底层实现

| 编码常量                | 编码所对应的底层数据结构   |
| ----------------------- | -------------------------- |
| OBJ_ENCODING_INT        | long类型的整数             |
| OBJ_ENCODING_EMBSTR     | embstr编码的简单动态字符串 |
| OBJ_ENCODING_RAW        | 简单动态字符串             |
| OBJ_ENCODING_HT         | 字典                       |
| OBJ_ENCODING_LINKEDLIST | 双向链表                   |
| OBJ_ENCODING_ZIPLIST    | 压缩列表                   |
| OBJ_ENCODING_INTSET     | 整数集合                   |
| OBJ_ENCODING_SKIPLIST   | 跳跃表                     |
| OBJ_ENCODING_QUICKLIST  | 快表                       |
| OBJ_ENCODING_ZIPMAP     | 压缩哈希表                 |
| OBJ_ENCODING_STREAM     | 消息流（用于消息队列）     |

```
#define OBJ_ENCODING_RAW 0     /* Raw representation */
#define OBJ_ENCODING_INT 1     /* Encoded as integer */
#define OBJ_ENCODING_HT 2      /* Encoded as hash table */
#define OBJ_ENCODING_ZIPMAP 3  /* Encoded as zipmap */
#define OBJ_ENCODING_LINKEDLIST 4 /* No longer used: old list encoding. */
#define OBJ_ENCODING_ZIPLIST 5 /* Encoded as ziplist */
#define OBJ_ENCODING_INTSET 6  /* Encoded as intset */
#define OBJ_ENCODING_SKIPLIST 7  /* Encoded as skiplist */
#define OBJ_ENCODING_EMBSTR 8  /* Embedded sds string encoding */
#define OBJ_ENCODING_QUICKLIST 9 /* Encoded as linked list of ziplists */
#define OBJ_ENCODING_STREAM 10 /* Encoded as a radix tree of listpacks */
```

每种类型的对象都**至少使用了两种**不同的编码，可以通过

```
OBJECT ENCODING key
```

指令来查看

**使用不同编码带来的好处**

通过encoding属性来设定对象所使用的编码，而不是为特定类型的对象关联一种固定的编码，**极大地提升了Redis的灵活性和效率**，因为Redis可以根据不同的使用场景来为一个对象设置不同的编码，从而优化对象在某一场景下的效率

### 3、字符串对象

字符串对象的编码可以是**int、raw或者embstr**

[![img](http://badwomen.asia/20201116154658.png)](https://nyimapicture.oss-cn-beijing.aliyuncs.com/img/20201116154658.png)

#### int

如果一个字符串对象保存的是**整数值**，并且这个整数值**可以用long类型来表示**，那么字符串对象会将整数值保存在字符串对象结构的ptr属性里面（将void*转换成long），并将字符串对象的编码设置为int

[![img](http://badwomen.asia/20201116153722.png)](https://nyimapicture.oss-cn-beijing.aliyuncs.com/img/20201116153722.png)

#### raw

如果字符串对象保存的是一个**字符串值**，并且这个字符串值的**长度大于32字节**，那么字符串对象将使用一个简单动态字符串（SDS）来保存这个字符串值，并将对象的编码设置为raw

[![img](http://badwomen.asia/20201116153707.png)](https://nyimapicture.oss-cn-beijing.aliyuncs.com/img/20201116153707.png)

#### embstr

embstr编码是专门用于保存**短字符串**的一种优化编码方式。这种编码和raw编码一样，都使用redisObject结构和sdshdr结构来表示字符串对象，但raw编码会调用两次内存分配函数来分别创建redisObject结构和sdshdr结构，而**embstr编码则通过调用一次内存分配函数来分配一块连续的空间**，空间中依次包含redisObject和sdshdr两个结构

[![img](http://badwomen.asia/20201116154107.png)](https://nyimapicture.oss-cn-beijing.aliyuncs.com/img/20201116154107.png)

简单来说，raw和embstr都是用来保存字符串的。字符串长度较短时使用embstr，较长时使用raw

- raw的redisObject和sdshdr是分别分配空间的，通过redisObject的ptr指针联系起来
- embstr的redisObject和sdshdr则是一起分配空间的，在内存中是一段连续的区域

#### 浮点数的编码

浮点数在Redis中使用embstr或者raw进行编码

#### 编码的转换

int编码的字符串对象和embstr编码的字符串对象在条件满足的情况下，会被转换为raw编码的字符串对象（**int/embstr -> raw**）

**int转raw**

编码为int的字符串，在**进行append操作后**，编码会转换为raw

[![img](http://badwomen.asia/20201116155237.png)](https://nyimapicture.oss-cn-beijing.aliyuncs.com/img/20201116155237.png)

**embstr转raw**

编码为embstr的字符串，**进行append操作后**，编码会转换为raw

[![img](http://badwomen.asia/20201116155653.png)](https://nyimapicture.oss-cn-beijing.aliyuncs.com/img/20201116155653.png)

### 4、列表对象

列表对象的编码是quicklist，quicklist在上面部分已经介绍过了，在此不再赘述

[![img](http://badwomen.asia/20201117214036.png)](https://nyimapicture.oss-cn-beijing.aliyuncs.com/img/20201117214036.png)

### 5、哈希对象

哈希对象的编码可以是ziplist或者hashtable

#### ziplist

**ziplist编码**的哈希对象使用压缩列表作为底层实现，每当有新的键值对要加入到哈希对象时，程序会先将保存了**键**的压缩列表节点推入到压缩列表表尾，然后再将保存了**值**的压缩列表节点推入到压缩列表表尾，因此

- 保存了同一键值对的两个节点总是紧挨在一起，保存**键的节点在前**，保存**值的节点在后**
- **先**添加到哈希对象中的键值对会被放在压缩列表的表头方向，而**后**来添加到哈希对象中的键值对会被放在压缩列表的表尾方向

如果我们依次向哈希表中添加一下元素，那么哈希对象中的压缩列表结构如下图所示

[![img](http://badwomen.asia/20201117215013.png)](https://nyimapicture.oss-cn-beijing.aliyuncs.com/img/20201117215013.png)

[![img](http://badwomen.asia/20201117215302.png)](https://nyimapicture.oss-cn-beijing.aliyuncs.com/img/20201117215302.png)

#### hashtable

hashtable编码的哈希对象使用字典作为底层实现，哈希对象中的每个键值对都使用一个字典键值对来保存

- 字典的每个键都是一个**字符串对象**，对象中保存了键值对的键
- 字典的每个值都是一个**字符串对象**，对象中保存了键值对的值

如果前面profile键创建的不是ziplist编码的哈希对象，而是**hashtable编码**的哈希对象，结构则如下图所示

[![img](http://badwomen.asia/20201117215430.png)](https://nyimapicture.oss-cn-beijing.aliyuncs.com/img/20201117215430.png)

#### 编码转换

当哈希对象可以**同时满足**以下两个条件时，哈希对象使用**ziplist编码**

- 哈希对象保存的所有键值对的**键和值的字符串长度都小于64字节**
- 哈希对象保存的键值对**数量小于512个**

不能满足这两个条件的哈希对象需要使用hashtable编码

**使用ziplist**

[![img](http://badwomen.asia/20201117215556.png)](https://nyimapicture.oss-cn-beijing.aliyuncs.com/img/20201117215556.png)

**使用hashtable**

[![img](http://badwomen.asia/20201117215637.png)](https://nyimapicture.oss-cn-beijing.aliyuncs.com/img/20201117215637.png)

### 6、集合(set)对象

集合对象的编码可以是intset或者hashtable

#### intset

intset编码的集合对象使用整数集合作为底层实现，集合对象包含的所有元素都被保存在整数集合里面

[![img](http://badwomen.asia/20201118192800.png)](https://nyimapicture.oss-cn-beijing.aliyuncs.com/img/20201118192800.png)

其结构如下图所示

[![img](http://badwomen.asia/20201118192819.png)](https://nyimapicture.oss-cn-beijing.aliyuncs.com/img/20201118192819.png)

#### hashtable

hashtable编码的集合对象使用字典作为底层实现，字典的每个**键都是一个字符串对象**，每个字符串对象包含了一个集合元素，而字典的值则全部被设置为NULL

[![img](http://badwomen.asia/20201118192946.png)](https://nyimapicture.oss-cn-beijing.aliyuncs.com/img/20201118192946.png)

其结构如下图所示

[![img](http://badwomen.asia/20201118193015.png)](https://nyimapicture.oss-cn-beijing.aliyuncs.com/img/20201118193015.png)

#### 编码转换

当集合对象可以**同时满足**以下两个条件时，对象使用intset编码

- 集合对象保存的**所有元素都是整数值**
- 集合对象保存的元素数量不超过512个

不能满足这两个条件的集合对象需要使用hashtable编码

### 7、有序(sorted_set)集合

有序集合的编码可以是ziplist或者skiplist

#### ziplist

ziplist编码的压缩列表对象使用压缩列表作为底层实现，每个集合元素**使用两个紧挨在一起的压缩列表节点来保存**，第一个节点保存元素的成员（member），而第二个元素则保存元素的分值（score）

[![img](http://badwomen.asia/20201118194654.png)](https://nyimapicture.oss-cn-beijing.aliyuncs.com/img/20201118194654.png)

其结构如下图所示

[![img](http://badwomen.asia/20201118194930.png)](https://nyimapicture.oss-cn-beijing.aliyuncs.com/img/20201118194930.png)

#### skiplist

skiplist编码的有序集合对象使用zset结构作为底层实现，一个zset结构同时包含一个跳跃表和一个字典

```
typedef struct zset {
    // 跳跃表
    zskiplist *zsl;
    
    // 字典
    dict *dict
} zset;Copy
```

其结构如下图所示

[![img](http://badwomen.asia/20201118201719.png)](https://nyimapicture.oss-cn-beijing.aliyuncs.com/img/20201118201719.png)

**字典和跳表中的数据**

![img](http://badwomen.asia/20201118201736.png)]

**注意**：上图在字典和跳跃表中重复展示了各个元素的成员和分值，但在实际中，**字典和跳跃表会共享元素的成员和分值**，所以并不会造成任何数据重复，也不会因此而浪费任何内存

zset结构中的zsl跳跃表按分值(score)**从小到大保存了所有集合元素**，每个跳跃表节点都保存了一个集合元素：跳跃表节点的object属性保存了元素的成员，而跳跃表节点的score属性则保存了元素的分值

其结构如下图所示

![img](http://badwomen.asia/20201118200641.png)]

除此之外，zset结构中的**dict字典为有序集合创建了一个从成员到分值的映射**，字典中的每个键值对都保存了一个集合元素：字典的键保存了元素的成员，而字典的值则保存了元素的分值。通过这个字典，**程序可以用O（1）复杂度查找给定成员的分值**，ZSCORE命令就是根据这一特性实现的，而很多其他有序集合命令都在实现的内部用到了这一特性

**为何sorted_set同时使用字典和跳表来作为底层的数据结构**

**字典**可以保证查询效率为O(1)，但是对于范围查询就无能为力了

**跳表**可以保证数据是有序的，范围查询效率较高，但是单个查询效率较低

### 8、内存回收

因为C语言并不具备自动内存回收功能，所以Redis在自己的对象系统中构建了一个**引用计数**（reference counting）技术实现的内存回收机制，通过这一机制，程序可以通过跟踪对象的引用计数信息，在适当的时候自动释放对象并进行内存回收

每个对象的引用计数信息由redisObject结构的**refcount**属性记录

```C
typedef struct redisObject {
    unsigned type:4;
    unsigned encoding:4;
    unsigned lru:LRU_BITS; 
    
    // 引用计数
    int refcount;
    void *ptr;
} robj;
```

对象的引用计数信息会随着对象的使用状态而不断变化

- 在创建一个新对象时，引用计数的值会被初始化为1
- 当对象被一个新程序使用时，它的引用计数值会被增一
- 当对象不再被一个程序使用时，它的引用计数值会被减一
- 当对象的引用计数值变为0时，对象所占用的内存会被释放

# 来源

https://nyimac.gitee.io/2020/11/08/Redis%E8%AE%BE%E8%AE%A1%E4%B8%8E%E5%AE%9E%E7%8E%B0/#Redis%E8%AE%BE%E8%AE%A1%E4%B8%8E%E5%AE%9E%E7%8E%B0
