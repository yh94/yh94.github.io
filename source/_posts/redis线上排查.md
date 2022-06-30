---
title: redis问题排查
date: 2021-07-05 10:57:12
tags: 
 - java
 - redis
categories: 
 - java
 - 性能
---
# Redis问题排查
- 在系统缓慢的情况下，查出是内网带宽暂满的原因导致。通过网络工具查出Redis在高分期使用内网带宽
  达到300M/S，占用总内网带宽的30%，于是对Redis问题进行排查。
# 通过RDB工具分析RDB中大Key的情况
- RDB是redis数据库的快照文件，存储在硬盘上，用于持久化缓存。带宽过高，肯定是大Key的频繁读取
  导致的，所以要找出Redis中排在前几的大key进行分析，于是乎从网上找到了一款基于Go语言编写的一
  款RDB分析工具--RDR（https://github.com/xueqiu/rdr）。
  通过RDR加载RDB后，分析结果如下：
  ![avatar](https://note.youdao.com/yws/res/177/619F82B355884E9EBE84E06C327AD989)
- 可以看到Redis的内存大小、key数量及大小、每个类型的数量、占比等，发现排在前几名的大Key分别
  是dataDicCache、configCache、sysPage、USER_PERMISSION。存储类型都是Hash，其中
  dataDicCache、configCache、sysPage在hash表中的数据都1条，但是大小却特别大，其中dataDicCache达到了惊人的918kb。
  通过Redis的MONITOR日志查看大Key的读取情况
  语法
  redis Monitor 命令基本语法如下：
>redis 127.0.0.1:6379> MONITOR

- 监控后得到的日志如下：
  ![image](https://note.youdao.com/yws/res/180/C4CDD337ED994CF5BA66CAF004805D76)
- 通过统计30秒左右的MONITOR日志，发现dataDicCache缓存读取数量巨大。一次900kb，一秒就有上百次。优化掉频繁读取的大Key，减少I/O量
  找到获取缓存的地方，以静态变量来存储，在调用缓存时直接获取变量，不走Redis来获取，刷新缓存时同时刷新静态变量。
### 总结:
不要将频繁读取的大表存储到一个key中，不管类型是hash还是string，可以将大表拆分成多个key来存储，一次获取只获取一条数据。可以减少大量的IO和反序列化所消耗的时间