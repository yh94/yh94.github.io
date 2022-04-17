---
title: QPS、TPS、并发数
date: 2022-04-16 10:57:12
tags: 
 - java 
categories: 
 - java
 - 并发
---
# 1. 平均响应时长（每个用户请求）
- 公式：Time token for tests/（Complete requests/Concurrency Level）
- 用户平均请求等待时间 = 总耗时 /（总请求数 / 并发数）
>  结合1、2 ==> 用户平均请求等待时间 = 服务器平均请求等待时间 * 并发数

# 2. 服务器平均请求等待时间（每个请求）
- 公式：总耗时 / 总请求数

# 3. QPS（每秒能够相应的查询次数）
- 公式：QPS = 并发数 / 平均响应时长
> 结合1、3 ==> QPS = 总请求数 / 总耗时

# 4.TPS
- 每秒钟处理完的事务次数，一般TPS是对整个系统来讲的。一个应用系统1s能完成多少事务处理

# 5. 并发数（系统同时处理的request 或 事务数）
- 公式：并发数 = QPS * 平均响应时长
> 指系统可以同时承载的正常使用系统功能的用户的数量

# 6. 吞吐量
- F = VU * R / T
- 其中F为吞吐量，VU表示虚拟用户个数，R表示每个虚拟用户发出的请求数，T表示性能测试所用的时间
> 一个系统吞吐量通常由QPS(TPS)、并发数两个因素决定，在应用场景访问压力下，只要某一项达到系统最高值或超标，系统超负荷工作、上下文切换、内存等其他消耗导致系统性能下降，系统的吞吐量会下降反而会下降。

# 7.AbTest例子
```
[root@s1 ~]# ab -c 500 -n 10000 -p POST.json -T application/json  http://10.152.49.33:8080/scrm/searchCustomer
This is ApacheBench, Version 2.3 <$Revision: 1430300 $>
Copyright 1996 Adam Twiss, Zeus Technology Ltd, http://www.zeustech.net/
Licensed to The Apache Software Foundation, http://www.apache.org/

Benchmarking 10.152.49.33 (be patient)
Completed 1000 requests
Completed 2000 requests
Completed 3000 requests
Completed 4000 requests
Completed 5000 requests
Completed 6000 requests
Completed 7000 requests
Completed 8000 requests
Completed 9000 requests
Completed 10000 requests
Finished 10000 requests


Server Software:
Server Hostname:        10.152.49.33
Server Port:            8080

Document Path:          /scrm/searchCustomer
Document Length:        5257 bytes

Concurrency Level:      500 // 并发数
Time taken for tests:   151.283 seconds // 总耗时
Complete requests:      10000 // 总请求数
Failed requests:        0
Write errors:           0
Total transferred:      53620000 bytes
Total body sent:        8030000
HTML transferred:       52570000 bytes
Requests per second:    66.10 [#/sec] (mean)  // QPS
Time per request:       7564.153 [ms] (mean)  // 用户平均请求等待时间
Time per request:       15.128 [ms] (mean, across all concurrent requests) // 服务器平均请求等待时间
Transfer rate:          346.13 [Kbytes/sec] received
                        51.84 kb/s sent
                        397.96 kb/s total

Connection Times (ms)
              min  mean[+/-sd] median   max
Connect:        0   13 112.2      0    1003
Processing:   168 7447 5399.8   5613   34826
Waiting:      168 7447 5399.8   5613   34826
Total:        183 7461 5398.2   5618   34826

Percentage of the requests served within a certain time (ms)
  50%   5618
  66%   6345
  75%   7057
  80%   7383
  90%  13845
  95%  23104
  98%  25971
  99%  27224
 100%  34826 (longest request)
```