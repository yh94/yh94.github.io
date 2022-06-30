---
title: HashMap
date: 2021-9-10
tags:
 - java
 - 数据结构 
categories:
 - java
---
Java8 对 HashMap 进行了一些修改，最大的不同就是利用了红黑树，所以其由 数组+链表+红黑
树 组成。
根据 Java7 HashMap 的介绍，我们知道，查找的时候，根据 hash 值我们能够快速定位到数组的
具体下标，但是之后的话，需要顺着链表一个个比较下去才能找到我们需要的，时间复杂度取决
于链表的长度，为 O(n)。为了降低这部分的开销，在 Java8 中，当链表中的元素超过了 8 个以后，
会将链表转换为红黑树，在这些位置进行查找的时候可以降低时间复杂度为 O(logN)。


# hashMap 底层分析
> 1.7版本 HashMap 底层是由数组和链表组成。数组里面存放的是Entry<K,V>的引用。初始容量为16，负载因子是0.75。扩容发生在当hashMap中元素个数大于arr.length * 负载因子 开始进行扩容。
- put具体实现：
```
HashMap<String.Object> map = new HashMap();
map.put(key,value);
```
对key.hashCode() 进行 & 操作。
为什么不是 % 呢？ 因为位运算符是运行最快的。
我们要的无非就是满足两点。
1、保证随机性散列在数组上。
2、不能越界
&操作是如何保证的呢？
首先key.hashCode() 会得到一个数字转换为二进制h
h & (length - 1) //有为1则为1
因为length都要保证是2的幂次方所以 lengh - 1 的 后四位都为1
0000 abcd
0000 1111
0000 abcd  其实就是hashCode 的后四位 ，保证了随机性。
为了更乱hashMap还进行了一些位运算，讲前面的数字参与进来。
扩容保证2的幂次函数：
Integer.highestOneBit( (number - 1) << 1 )
highestOneBit 方法 是获取到小于等于参数的最大2的幂。
实现自己去看。
想要拿到比他小的最大2的幂
只需要把第一个1后面的所有数字都变成0就可以了
对number进行翻倍,-1是为了刚好卡在2的幂方的数。

 
