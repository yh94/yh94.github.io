---
title: CAS简单了解
date: 2020-05-10
tags:
 - java
 - 锁 
categories:
 - java
 - 锁
---

## CAS
> CAS：compare and swap 
> 是一个原子性操作，对应cpu指令 cmpxchg
> CAS 有三个操作数：
 - 当前值A
 - 内存值V
 - 要修改的新值B
 ```
 update A -> B
 if(A == V) { //判断当前值和内存值是否相等
    V == B
 }else{
    //重试，或者放弃更新
 }
 ```
```
public class AtomicDemo { 
public static int NUMBER = 0; 
public static void increase() { 
    NUMBER++; 
} 
public static void main(String[] args) throws InterruptedException { 
AtomicDemo test = new AtomicDemo(); 
for (int i = 0; i < 10; i++) 
{ new Thread(() -> { for (int j = 0; j < 1000; j++) test.increase(); }).start(); } 
Thread.sleep(200); 
System.out.println(test.NUMBER); 
} }
```
这里很明显得出的结果不会是我们那想要的。
```
public static AtomicInteger NUMBER = new AtomicInteger(0); 
public static void increase() { 
    NUMBER.getAndIncrement(); 
} 
public static void main(String[] args) throws InterruptedException { 
AtomicDemo test = new AtomicDemo(); 
for (int i = 0; i < 10; i++) { new Thread(() -> { for (int j = 0; j < 1000; j++) test.increase(); }).start(); } 
Thread.sleep(200); 
System.out.println(test.NUMBER); }
```
运行main方法，程序输出的就是我们想要的值，也就是10000
为什么要使用CAS？
 - ok，就不得不说synchorized，它是java锁、jvm锁。这里还有一个知识点就是对象信息（下次再说吧）。
   synchorized锁每次只会让一个线程去操作共享资源，而CAS相当于没有加锁，多个线程都可以直接去操作共享资源。
在实际修改再判断是否修改成功。

synchorized效率要 > CAS 
##CAS缺点？
> ABA问题。
 - ABA：假设线程A读到当前值是10，可能线程B修改成100，然后C又修改成10。

当线程A拿到执行权时，因为当前值和内存值是一样的，所以是可以修改的。
其实我觉得没有什么太大问题，但是也是可以解决的，有一个**AtomicStampedReference**类提供给我们使用
说白了就是加个version，对比一下是否一致。
## 但是推荐使用 LongAdder对象
> AutoMic做累加的时候实际上是多个线程操作同一个资源，其他线程不断自旋，效率和性能就不说了吧。
而LongAdder的思想就是要把操作的目标资源分散到数组cell中。
每个线程对自己的cell进行原子操作，大大降低了失败的次数，有点像volidate哈，下课～！