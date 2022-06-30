---
title: redis底层数据结构
date: 2021-03-10 
tags:
 - java
 - redis 
categories:
 - java
 - redis
---
### 文章目录

*   [一、简单动态字符串](about:blank#_2)
*   *   [1、SDS的结构定义](about:blank#1SDS_4)
*   [2、SDS和c字符串的区别](about:blank#2SDSc_11)
*   *   [1）SDS获取字符串长度复杂度为常数](about:blank#1SDS_12)
*   [2）SDS杜绝了缓冲区溢出](about:blank#2SDS_15)
*   [3）减少内存重分配次数](about:blank#3_18)
*   [4）二进制安全](about:blank#4_26)
*   [5）SDS兼容部分c字符串函数](about:blank#5SDSc_28)
*   [二、双向链表](about:blank#_31)
*   *   [1、双向链表结构：](about:blank#1_32)
*   [2、链表节点结构：](about:blank#2_34)
*   [3、Redis的链表实现的特性：](about:blank#3Redis_36)
*   [三、字典](about:blank#_42)
*   *   [1、字典的实现结构](about:blank#1_46)
*   *   [1）哈希表节点](about:blank#1_49)
*   [2）哈希表](about:blank#2_55)
*   [3）字典](about:blank#3_61)
*   [2、哈希算法](about:blank#2_70)
*   *   [MurmurHash2算法](about:blank#MurmurHash2_73)
*   [3、哈希表的重新散列（rehash）](about:blank#3rehash_76)
*   [四、跳跃表](about:blank#_93)
*   *   [1、跳跃表节点的结构（zskiplistNode）](about:blank#1zskiplistNode_97)
*   [2、跳跃表的结构（zskiplist）](about:blank#2zskiplist_108)
*   [五、整数集合](about:blank#_117)
*   *   [1、整数集合的结构](about:blank#1_119)
*   [2、整数集合的升级](about:blank#2_125)
*   [六、压缩列表](about:blank#_135)
*   *   [1、压缩列表的构成](about:blank#1_137)
*   [2、压缩列表节点的结构](about:blank#2_147)
*   [3、连锁更新](about:blank#3_153)


本文将详细介绍Redis的六种底层数据结构：简单动态字符串、双向链表、字典、跳跃表、整数集合和压缩列表。

[](https://blog.csdn.net/Abel_Liujinquan/article/details/89339599)一、简单动态字符串
===========================================================================

Redis没有直接使用c语言传统的字符串表示，而是自己构建了一种名为简单动态字符串的可以被修改的抽象类型，并将SDS用作Redis的默认字符串表示。

[](https://blog.csdn.net/Abel_Liujinquan/article/details/89339599)1、SDS的结构定义
----------------------------------------------------------------------------

SDS结构定义如下：  
![在这里插入图片描述](https://img-blog.csdnimg.cn/20190416200836643.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L0FiZWxfTGl1amlucXVhbg==,size_16,color_FFFFFF,t_70)
注：  
free属性的值为0，表示这个SDS没有分配任何未使用空间。  
len属性的值为5，表示这个SDS保存了一个五字节长的字符串。  
buf属性是一个char类型的数组，数组最后保存了空字符‘\\0’。

[](https://blog.csdn.net/Abel_Liujinquan/article/details/89339599)2、SDS和c字符串的区别
-------------------------------------------------------------------------------

### [](https://blog.csdn.net/Abel_Liujinquan/article/details/89339599)1）SDS获取字符串长度复杂度为常数

SDS通过获取len属性就可以得到字符串的长度，时间复杂度为： O ( 1 ) O(1) O(1)  
c字符串需要遍历字符串，时间复杂度为： O ( N ) O(N) O(N)

### [](https://blog.csdn.net/Abel_Liujinquan/article/details/89339599)2）SDS杜绝了缓冲区溢出

c字符串如果没有重新分配空间，直接修改字符串的话，可能会造成数据溢出。  
当SDS的API需要对SDS进行修改时，API会先检查SDS的空间是否满足修改所需的需求，如果不满足，则自动将SDS空间扩展至所需大小。

### [](https://blog.csdn.net/Abel_Liujinquan/article/details/89339599)3）减少内存重分配次数

SDS通过空间预分配和惰性空间释放两种优化策略来减少内存重分配次数。

*   空间预分配  
    Redis通过额外分配未使用的空间，优化了SDS的字符串增长操作，减少了连续执行字符串增长操作所需的内存分配次数。
*   惰性空间释放  
    惰性空间释放用于优化SDS的字符串缩短操作。当SDS缩短时，程序并不会立即回收缩短后多出来的空间，而是使用free属性将这些字节的数量记录起来，等待将来使用  
    注：如果需要真正地释放SDS的未使用空间，我们可以使用相应的API。

### [](https://blog.csdn.net/Abel_Liujinquan/article/details/89339599)4）二进制安全

SDS API会以处理二进制的方式来处理SDS存放在buf数组里的数据，程序不会对其中的数据做任何的限制、过滤或者假设，所以SDS的API都是二进制安全的。

### [](https://blog.csdn.net/Abel_Liujinquan/article/details/89339599)5）SDS兼容部分c字符串函数

SDS之所以在末尾保存一个空字符’\\0’，是为了使用一些c字符串<string.h>函数库，避免不必要的代码重复。  
例如 字符串对比函数：<string.h>/strcasecmp函数

[](https://blog.csdn.net/Abel_Liujinquan/article/details/89339599)二、双向链表
========================================================================

[](https://blog.csdn.net/Abel_Liujinquan/article/details/89339599)1、双向链表结构：
---------------------------------------------------------------------------

![在这里插入图片描述](https://img-blog.csdnimg.cn/2019041620100646.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L0FiZWxfTGl1amlucXVhbg==,size_16,color_FFFFFF,t_70)

[](https://blog.csdn.net/Abel_Liujinquan/article/details/89339599)2、链表节点结构：
---------------------------------------------------------------------------

![在这里插入图片描述](https://img-blog.csdnimg.cn/20190416201057683.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L0FiZWxfTGl1amlucXVhbg==,size_16,color_FFFFFF,t_70)

[](https://blog.csdn.net/Abel_Liujinquan/article/details/89339599)3、Redis的链表实现的特性：
----------------------------------------------------------------------------------

*   双端
*   无环
*   带表头指针和表尾指针
*   带链表长度计数器
*   多态（保存各种不同类型的值）

[](https://blog.csdn.net/Abel_Liujinquan/article/details/89339599)三、字典
======================================================================

字典，又被称为符号表、关联数组或映射，是一种用于保存键值对的抽象数据结构。  
字典中的每个键都是独一无二的，程序可以在字典中根据键查找与之关联的值，或者通过键来更新值，又或者根据键来删除整个键值对，等等。  
字典用途：Redis数据库底层实现、哈希键底层实现。

[](https://blog.csdn.net/Abel_Liujinquan/article/details/89339599)1、字典的实现结构
---------------------------------------------------------------------------

Redis的字典使用哈希表作为底层实现，一个哈希表里面可以有多个哈希表节点，而每个哈希表节点就保存了字典中的一个键值对。  
接下来的三个小节将分别介绍Redis的哈希表节点、哈希表以及字典的实现。

### [](https://blog.csdn.net/Abel_Liujinquan/article/details/89339599)1）哈希表节点

![在这里插入图片描述](https://img-blog.csdnimg.cn/20190416204526782.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L0FiZWxfTGl1amlucXVhbg==,size_16,color_FFFFFF,t_70)

*   key属性保存着键值对中的键；
*   v属性保存着键值对中的值，其中值用union定义，支持三种数据类型。
*   next属性是指向另一个哈希表节点的指针，这个指针可以将多个哈希值相同的键值对连接在一起，以此来解决键冲突的问题。

### [](https://blog.csdn.net/Abel_Liujinquan/article/details/89339599)2）哈希表

![在这里插入图片描述](https://img-blog.csdnimg.cn/2019041620504380.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L0FiZWxfTGl1amlucXVhbg==,size_16,color_FFFFFF,t_70)

*   table属性是一个数组，数组中的每个元素都是一个指向哈希表节点的指针，每个节点都保存着一个键值对；
*   size属性记录了哈希表的大小，也就是table数组的大小；
*   sizemask属性的值总是等于size-1，这个属性和哈希值一起决定一个键应该被放到table数组的那个索引上面；
*   used属性记录了哈希表目前已有节点的数量。

### [](https://blog.csdn.net/Abel_Liujinquan/article/details/89339599)3）字典

![在这里插入图片描述](https://img-blog.csdnimg.cn/20190416205910583.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L0FiZWxfTGl1amlucXVhbg==,size_16,color_FFFFFF,t_70)

*   type属性是一个指向dictType结构的指针，每个dictType结构保存了一簇用于操作特定类型键值对的函数；
*   privdata属性保存了需要传给那些类型特定函数的可选参数；
*   ht属性是一个包含两个项的数组，数组中的每个项都是一个dictht哈希表，ht\[1\]只有在对ht\[0\]哈希表进行rehash操作时使用；
*   trehashidx属性是rehash索引，没有进行rehash操作时值都为-1.

字典、哈希表和哈希表节点关系图：  
![在这里插入图片描述](https://img-blog.csdnimg.cn/20190416211628239.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L0FiZWxfTGl1amlucXVhbg==,size_16,color_FFFFFF,t_70)

[](https://blog.csdn.net/Abel_Liujinquan/article/details/89339599)2、哈希算法
------------------------------------------------------------------------

当要将一个新的键值对添加到字典里面时，程序需要先根据键值对的键计算出哈希值和索引值，然后再根据索引值，将包含新键值对的哈希表节点放到哈希表数组的指定索引上。  
若果字典被用做数据库的底层实现，或者哈希键的底层实现时，Redis使用MurmurHash2算法来计算键的哈希值。

### [](https://blog.csdn.net/Abel_Liujinquan/article/details/89339599)MurmurHash2算法

MurmurHash2算法用来计算键的哈希值，特点是运算性能高，碰撞率低。  
详细参考：[https://blog.csdn.net/thinkmo/article/details/26833565](https://blog.csdn.net/thinkmo/article/details/26833565)

[](https://blog.csdn.net/Abel_Liujinquan/article/details/89339599)3、哈希表的重新散列（rehash）
------------------------------------------------------------------------------------

随着操作的不断执行，哈希表保存的键值对会主键增多或者减少，当哈希表保存的键值对数量太多或者太少时，程序需要对哈希表的大小进行相应的扩展或者收缩。  
扩展和收缩哈希表的工作可以通过执行rehash（重新散列）操作来完成，步骤如下：

**1）为字典的ht\[1\]哈希表分配空间，空间大小根据实际情况而定；**

**2）将ht\[0\]中所有键值对rehash到ht\[1\]中**  
注意：rehash指的是重新计算键的哈希值和索引值，然后将键值对放置到ht\[1\]哈希表的指定位置上

**3）释放ht\[0\]，将ht\[1\]设置为ht\[0\]，并在ht\[1\]新建一个空表，为下次rehash做准备**  
rehash操作的结构变化如下：

![在这里插入图片描述](https://img-blog.csdnimg.cn/20190417091847145.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L0FiZWxfTGl1amlucXVhbg==,size_16,color_FFFFFF,t_70)  
注：  
rehash操作是渐进式的。  
渐进式的rehash将rehash键值对所需的计算工作均摊到对字典的每个添加、删除、查找和更新操作上。  
之所以这样做，是考虑到如果哈希表保存的键值对的数量是百万级甚至千万级时，一次性进行rehash可能会导致服务器停止服务，渐进式地rehash避免了对服务器性能造成影响。

[](https://blog.csdn.net/Abel_Liujinquan/article/details/89339599)四、跳跃表
=======================================================================

跳跃表是一种有序数据结构，它通过在每个节点中维持多个指向其他节点的指针，从而达到快速访问节点的目的。  
Redis使用跳跃表作为有序集合键的底层实现之一，如果一个有序集合包含的元素数量比较多，又或者有序集合中元素的成员是比较长的字符串时，Redis就会使用跳跃表来作为有序集合键的底层实现。  
Redis只在两个地方用到了跳跃表，一个是实现有序集合键，另一个是在集群结点中用作内部数据结构，除此之外，跳跃表没有其他用途。

[](https://blog.csdn.net/Abel_Liujinquan/article/details/89339599)1、跳跃表节点的结构（zskiplistNode）
-------------------------------------------------------------------------------------------

![在这里插入图片描述](https://img-blog.csdnimg.cn/20190417212726757.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L0FiZWxfTGl1amlucXVhbg==,size_16,color_FFFFFF,t_70)

*   **层**：跳跃表节点的 level\[\] 数组可以包含多个元素，每个元素都包含一个指向其他节点的指针 和 跨度，下标从0开始为第一层；
*   **前进指针**：每个层都有一个指向表尾方向的前进指针，用于从表头向表尾方向访问节点；
*   **跨度**：层的跨度用于记录两个节点之间的距离，指向NULL的所有前进指针的跨度为0；跨度用来计算节点的排位：在查找某个节点的过程中，将沿途访问过的所有层的跨度累计起来，得到的结果就是目标节点在跳跃表中的排位。
*   **后退指针**：后退指针用于从表尾向表头方向访问节点，每次只能后退一个节点；
*   **分值和成员**：  
    分值：一个double类型的浮点数，所有节点都按照分值从小到大排序，多个节点可以包含相同的分值；  
    成员：一个指针，指向一个字符串对象，该字符串对象保存着一个SDS值，成员对象必须是唯一的。

[](https://blog.csdn.net/Abel_Liujinquan/article/details/89339599)2、跳跃表的结构（zskiplist）
-------------------------------------------------------------------------------------

仅靠过个跳跃表节点就可以组成一个跳跃表，但通过使用一个zskiplist结构来持有这些节点，就可以很方便地对整个跳跃表进行处理。  
zskiplist结构如图：![在这里插入图片描述](https://img-blog.csdnimg.cn/20190417215602606.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L0FiZWxfTGl1amlucXVhbg==,size_16,color_FFFFFF,t_70)

*   header和tail指针分别指向跳跃表的表头和表尾节点；
*   length属性记录节点的数量；
*   level属性记录层数最高的几点的层数量；

下图分别展示了完整的跳跃表和单个节点的详细结构图：  
![在这里插入图片描述](https://img-blog.csdnimg.cn/20190417222432637.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L0FiZWxfTGl1amlucXVhbg==,size_16,color_FFFFFF,t_70)

[](https://blog.csdn.net/Abel_Liujinquan/article/details/89339599)五、整数集合
========================================================================

整数集合是Redis用于保存整数值的集合抽象数据结构，它可以保存类型为 int16\_t 、int32\_t 或者 int64\_t 的整数值，并且保证集合中不出现重复值。

[](https://blog.csdn.net/Abel_Liujinquan/article/details/89339599)1、整数集合的结构
---------------------------------------------------------------------------

![在这里插入图片描述](https://img-blog.csdnimg.cn/20190417224045599.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L0FiZWxfTGl1amlucXVhbg==,size_16,color_FFFFFF,t_70)

*   **encoding**：contents数组中元素的类型，有 INTSET\_ENC\_INT16、INTSET\_ENC\_INT32 和 INTSET\_ENC\_INT64 三种，分别表示contents数组中元素类型为 int16\_t（16位二进制）、int32\_t 和 int64\_t 类型。
*   **contents**：整数集合的每个元素都是contents数组的一个数组项，各个项在数组中按值的大小从小到大有序地排列，数组中不包含重复项。
*   **length**：记录了整数集合包含的元素数量。  
    ![在这里插入图片描述](https://img-blog.csdnimg.cn/20190418131716416.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L0FiZWxfTGl1amlucXVhbg==,size_16,color_FFFFFF,t_70)

[](https://blog.csdn.net/Abel_Liujinquan/article/details/89339599)2、整数集合的升级
---------------------------------------------------------------------------

如果我们想要添加一个新元素到整数集合里面，但是新元素的类型比整数集合原有的元素类型都要长时，我们就要对整数集合进行升级，然后才能将新元素添加到整数集合里面。  
另外，还需注意，Redis的整数集合不支持降级。

**升级整数集合并添加新元素共分为三步进行：**  
1）根据新元素的类型，扩展整数集合底层数组的空间大小，并为新元素分配空间。  
2）将底层数组现有的所有元素都转换成与新元素相同的类型，并将类型转换后的元素继续维持底层数组的有序性质不变。  
3）将新元素添加到底层数组里面。

升级步骤图解：![在这里插入图片描述](https://img-blog.csdnimg.cn/20190418133651854.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L0FiZWxfTGl1amlucXVhbg==,size_16,color_FFFFFF,t_70)

[](https://blog.csdn.net/Abel_Liujinquan/article/details/89339599)六、压缩列表
========================================================================

压缩列表是列表键和哈希键的底层实现之一。当一个列表键值包含少量列表键，并且每个列表项要么就是小整数值，要么就是长度比较短的字符串，那么Redis就会使用压缩列表来做列表键的底层实现。

[](https://blog.csdn.net/Abel_Liujinquan/article/details/89339599)1、压缩列表的构成
---------------------------------------------------------------------------

压缩列表是由一系列特殊编码的连续内存块组成的顺序型数据结构，一个压缩列表可以包含任意多个节点 （entry），每个节点可以保存一个字节数组或者一个整数值。

压缩列表结构：![在这里插入图片描述](https://img-blog.csdnimg.cn/20190419083726519.png)

*   zlbytes属性：表示压缩列表的总字节长度；
*   zltail属性：记录压缩列表表尾节点距离压缩列表的起始地址有多少字节；
*   zllen属性：记录了压缩列表包含的节点数量；
*   entryX属性：压缩列表包含的各个节点；
*   zlend属性：用于标记压缩列表的末端。

[](https://blog.csdn.net/Abel_Liujinquan/article/details/89339599)2、压缩列表节点的结构
-----------------------------------------------------------------------------

压缩列表节点的结构如图：![在这里插入图片描述](https://img-blog.csdnimg.cn/20190419085114429.png)

*   **previous\_entry\_length属性**：以字节为单位，记录压缩列表中前一个节点的长度。  
    程序可以通过指针运算，根据当前节点的起始地址来计算出前一个节点的起始地址，以此实现遍历操作。
*   **encoding属性**：记录了节点的content属性所保存数据的类型和长度；
*   **content属性**：保存节点的值，可以是一个字节数组或者整数，值的类型和长度由节点的encoding属性决定。

[](https://blog.csdn.net/Abel_Liujinquan/article/details/89339599)3、连锁更新
------------------------------------------------------------------------

假设压缩列表中所有节点的previous\_entry\_length属性都是用1字节来保存，那么节点的长度只要小于等于253字节previous\_entry\_length都可以记录，但是，如果添加一个长度大于253字节的节点，那么下一个节点的previous\_entry\_length就无法保存该长度的值，同样的，下下个节点也无法保存上个节点的长度，由此将导致连续多次空间扩展操作。

添加节点和删除节点都可能导致连锁更新，但是这种操作出现的几率很低。





本文转自 [https://blog.csdn.net/Abel\_Liujinquan/article/details/89339599](https://blog.csdn.net/Abel_Liujinquan/article/details/89339599)，如有侵权，请联系删除。