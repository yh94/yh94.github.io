---
title: 浅谈ArrayList
date: 2020-06-10
tags:
- java
categories:
- java
---


浅谈ArrayList
===========

**ArrayList类**又称动态数组，同时实现了Collection和List接口，其内部数据结构由数组实现，因此可对容器内元素实现快速随机访问。但因为ArrayList中插入或删除一个元素需要移动其他元素，所以不适合在插入和删除操作频繁的场景下使用。

ArrayList的容量可以随着元素的增加而自动增加，因此不用担心ArrayList容量不足的问题。

ArrayList是非线程安全的。

接下来，我们将解析ArrayList的构造方法，在看构造方法之前，我们先来明确一下ArrayList源码中的一些概念。这些变量和对象大家可能有疑惑，先记住就好了，后面会看到它们的用途。

```
// 默认的容量大小（常量）
private static final int DEFAULT\_CAPACITY = 10;

// 定义的空数组（final修饰，大小固定为0）
private static final Object\[\] EMPTY\_ELEMENTDATA = {};

// 定义的默认空容量的数组（final修饰，大小固定为0）
private static final Object\[\] DEFAULTCAPACITY\_EMPTY\_ELEMENTDATA = {};

// 定义的不可被序列化的数组，实际存储元素的数组
transient Object\[\] elementData; 

// 数组中元素的个数
private int size;
```

ArrayList有三种构造方法:

1.无参的构造方法

2.根据传入的数值大小，创建指定长度的数组

3.通过传入Collection元素列表进行生成

1.无参的构造方法
---------

```
// 无参的构造方法
public ArrayList() {
    this.elementData = DEFAULTCAPACITY\_EMPTY\_ELEMENTDATA;
}
```

可以看出来，当我们直接创建ArrayList时，elementData被赋予了默认空容量的数组。注意，因为默认空容量数组是被final修饰的，此时ArrayList数组是空的、固定长度的，也就是说其容量此时是0，元素个数size为默认值0。

2.根据传入的数值大小，创建指定长度的数组
---------------------

```
// 传容量的构造方法
public ArrayList(int initialCapacity) {
    if (initialCapacity > 0) {
        this.elementData = new Object\[initialCapacity\];
    } else if (initialCapacity == 0) {
        this.elementData = EMPTY\_ELEMENTDATA;
    } else {
        throw new IllegalArgumentException("Illegal Capacity: "+
                                           initialCapacity);
    }
}
```

当initialCapacity > 0时，会在堆上new一个大小为initialCapacity的数组，然后将其引用赋给elementData，此时ArrayList的容量为initialCapacity，元素个数size为默认值0。

当initialCapacity = 0时，elementData被赋予了默认空数组，因为其被final修饰了，所以此时ArrayList的容量为0，元素个数size为默认值0。

当initialCapacity < 0时，会抛出异常。

3.通过传入Collection元素列表进行生成
------------------------

```
// 传入Collection元素列表的构造方法
public ArrayList(Collection<? extends E> c) {
    // 将列表转化为对应的数组
    elementData = c.toArray();
    if ((size = elementData.length) != 0) {
        // 此处见下面详细解析
        if (elementData.getClass() != Object\[\].class)
            elementData = Arrays.copyOf(elementData, size, Object\[\].class);
    } else {
        // 赋予空数组
        this.elementData = EMPTY\_ELEMENTDATA;
    }
}
```

传入Collection元素列表后，构造方法首先会将其转化为数组，将其索引赋给elementData。

如果此数组的长度为0，会重新赋予elementData为空数组，此时ArrayList的容量是0，元素个数size为0。

如果此数组的长度大于0，会更新size的大小为其长度，也就是元素的个数，然后执行里面的程序。大家对里面的代码可能不理解，让我们等会看下面解析。执行完后此时ArrayList的容量为传入序列的长度，也就是size的大小，同时元素个数也为size，也就是说，此时ArrayList是满的。

让我们来看看下面的代码，然后再去理解上面 if 语句的代码：

```
public class Test {
    public static void main(String\[\] args) {
        // 1.创建Student对象数组
        Student\[\] students = new Student\[\] {
                new Student("小明", 18),
                new Student("小李", 19),
                new Student("小张", 21)
        };
        
        // 2.将其赋值给Object对象数组
        Object\[\] objects = students;
        
        // 3.执行if语句前，打印数组的class
        System.out.println("执行前：" + objects.getClass());
        
        // 4.执行上面的代码
        if (objects.getClass() != Object\[\].class) {
            objects \= Arrays.copyOf(objects, objects.length, Object\[\].class);
        }
        
        // 5.执行if语句后，打印数组的class
        System.out.println("执行后：" + objects.getClass());
    }
}
```

程序的运行结果如下：

![](https://img2020.cnblogs.com/blog/2123988/202009/2123988-20200922191354245-68689372.png)

可以看到，对象数组也是有.class的，其中含有所存储元素的类型，而上面的那段代码的作用就是将原对象数组的数组类型转化为Object对象数组的数组类型，以便更好的存储。

ArrayList的扩容机制
==============

当我们探讨扩容时，肯定要从ArrayList的add方法走起，让我们来看看吧。

```
public boolean add(E e) {
    modCount++;
    add(e, elementData, size);
    return true;
}
```

这是最基本的add方法，当然，也是可以说明问题的。可以看到，此add方法的参数就是一个被加元素，moCount是记录ArrayList被修改次数的，可以不用管。然后是另一个add方法，所传的值是被加元素、当前数组和当前数组的元素个数，让我们来看看这个add方法吧。

```
private void add(E e, Object\[\] elementData, int s) {
    // 判断元素个数是否等于当前容量
    if (s == elementData.length)
        elementData \= grow();
    elementData\[s\] \= e;
    size \= s + 1;
}
```

首先，它判断了元素个数是否等于当前数组的容量，也就是判断当前数组是不是满的，如果当前空间是满的，就需要扩容了，grow函数就是扩容函数了，扩容后再将被加元素加到数组中。

下面我们来看看grow函数是什么样子的：

```
private Object\[\] grow() {
    return grow(size + 1);
}
```

它里面又调用了一个带参的grow函数，参数是当前元素个数+1，也就是当前容量+1。返回的是这个函数的返回值，让我们进一步研究这个带参的函数。

```
private Object\[\] grow(int minCapacity) {
    // 获取老容量，也就是当前容量
    int oldCapacity = elementData.length;
    // 如果当前容量大于0 或者 数组不是DEFAULTCAPACITY\_EMPTY\_ELEMENTDATA
    if (oldCapacity > 0 || elementData != DEFAULTCAPACITY\_EMPTY\_ELEMENTDATA) {
        int newCapacity = ArraysSupport.newLength(oldCapacity,
                minCapacity \- oldCapacity, /\* minimum growth \*/
                oldCapacity \>> 1           /\* preferred growth \*/);
        return elementData = Arrays.copyOf(elementData, newCapacity);
    // 如果 数组是DEFAULTCAPACITY\_EMPTY\_ELEMENTDATA（容量等于0的话，只剩这一种情况了）
    } else {
        return elementData = new Object\[Math.max(DEFAULT\_CAPACITY, minCapacity)\];
    }
}
```

首先，它是记录了一下老容量的大小，然后再进行下面的操作。

如果当前容量大于0，或者当前数组不是DEFAULTCAPACITY\_EMPTY\_ELEMENTDATA，前面说明过，DEFAULTCAPACITY\_EMPTY\_ELEMENTDATA是一个被final修饰的空数组，在三个构造方法中，只有无参构造方法中elementData被赋予了DEFAULTCAPACITY\_EMPTY\_ELEMENTDATA。也就是说，这个if语句中不会处理用默认无参构造方法创建的数组的初始扩容情况，那么其余的扩容情况都是由此if语句来处理的。

我们来看一下if里面的操作，先创建一个新的数组，然后将旧数组拷贝到新数组并赋给elementData返回。ArraysSupport.newLength函数的作用是创建一个大小为oldCapacity + max（minimum growth， preferred growth）的数组。

minCapacity是传入的参数，我们上面看过，它的值是当前容量（老容量）+1，那么minCapacity - oldCapacity的值就恒为1，minimum growth的值也就恒为1。

oldCapacity >> 1的功能是将oldCapacity 进行位运算，右移一位，也就是减半，preferred growth的值即为oldCapacity大小的一半。

扩容分析：

当oldCapacity为0时，右移后还是0，也就是说此时扩容的大小为0+max（1,0）=1，容量从0扩展到1。那么什么时候是这种情况呢？

（1）传容量的构造方法传入的是0时，elementData被赋予的是EMPTY\_ELEMENTDATA，此时数组容量为0，添加元素时，符合if的条件，会进入此扩容情况，容量从0扩展到1。

（2）传Collection元素列表的构造方法被传入空列表时，elementData被赋予的是EMPTY\_ELEMENTDATA，数组容量为0，此时添加元素时，符合if的条件，会进入此扩容情况，容量从0扩展到1。

当oldCapacity大于0时，新创建的数组大小是老容量+老容量的一半，也就是老容量的1.5倍，每次扩容到原来的1.5倍。

else就剩一种情况了，也就是用默认无参构造方法创建的数组的初始扩容情况。此时的容量为0，添加一个元素时会创建一个新的数组，其大小为max(DEFAULT\_CAPACITY, minCapacity)。

我们从上面的源码变量信息中可得知DEFAULT\_CAPACITY是一个常量，其值为10，而minCapacity的值为（0+1），所以添加一个元素时，max(DEFAULT\_CAPACITY, minCapacity)的值必为10。也就是说，当我们用默认无参构造方法创建的数组在添加元素前，ArrayList的容量为0，添加一个元素后，ArrayList的容量就变为10了。

总结一下
====

**ArrayList的特点：**

**1.ArrayList的底层数据结构是数组，所以查找遍历快，增删慢。**

**2.ArrayList可随着元素的增长而自动扩容，正常扩容的话，每次扩容到原来的1.5倍。**

**3.ArrayList的线程是不安全的。**

**ArrayList的扩容：**

**扩容可分为两种情况：**

**第一种情况，当ArrayList的容量为0时，此时添加元素的话，需要扩容，三种构造方法创建的ArrayList在扩容时略有不同：**

1.无参构造，创建ArrayList后容量为0，添加第一个元素后，容量变为10，此后若需要扩容，则正常扩容。

2.传容量构造，当参数为0时，创建ArrayList后容量为0，添加第一个元素后，容量为1，此时ArrayList是满的，下次添加元素时需正常扩容。

3.传列表构造，当列表为空时，创建ArrayList后容量为0，添加第一个元素后，容量为1，此时ArrayList是满的，下次添加元素时需正常扩容。

**第二种情况，当ArrayList的容量大于0，并且ArrayList是满的时，此时添加元素的话，进行正常扩容，每次扩容到原来的1.5倍。**



本文转自 [https://www.cnblogs.com/ruoli-0/p/13714389.html](https://www.cnblogs.com/ruoli-0/p/13714389.html)，如有侵权，请联系删除。
