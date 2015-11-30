---
layout: post
title: 使用Apache Benchmark测试网页性能
---

### 写在前面
>不要总是“我觉得”，你应该编写Benchmark测试对比

上面这句话是我观看RubyConf China 2015时听到的，旨在号召社区内的同学不要仅仅凭感觉，就认为这种写法比较慢，那种写法比较快。

恰好今天将应用的Ruby版本从`2.0.0`升级到了`2.2.3`，于是就想测试一下同一个网页使用不同的ruby版本，速度会相差多少。

### 简单安装使用
我选择了Apache Benchmark来做测评，因为看起来使用挺简单的。

Ubuntu下可以这么安装：

```
sudo apt-get install apache2-utils
```

装好后我们简单试用一下：

```
ab -n 10 -c 10 http://www.163.com/
```

然后我们就会看到长长的输出结果啦：

```
This is ApacheBench, Version 2.3 <$Revision: 1528965 $>
Copyright 1996 Adam Twiss, Zeus Technology Ltd, http://www.zeustech.net/
Licensed to The Apache Software Foundation, http://www.apache.org/

Benchmarking www.163.com (be patient).....done


Server Software:        nginx
Server Hostname:        www.163.com
Server Port:            80

Document Path:          /
Document Length:        729047 bytes

Concurrency Level:      10
// 完成这项测试总共耗时
Time taken for tests:   6.304 seconds
Complete requests:      10
Failed requests:        0
Total transferred:      7293684 bytes
HTML transferred:       7290470 bytes
// 每秒执行了多少个请求
Requests per second:    1.59 [#/sec] (mean)
// 这一组请求所消耗的时间
Time per request:       6304.459 [ms] (mean)
// 平均下来每个请求所消耗的时间
Time per request:       630.446 [ms] (mean, across all concurrent requests)
Transfer rate:          1129.79 [Kbytes/sec] received

// 以下这段数据标志了一个请求从连接、发送数据、接收数据这三个阶段所消耗时间的最小值、平均值、中位数以及最大值
Connection Times (ms)
              min  mean[+/-sd] median   max
Connect:        8   90  87.5    150     185
Processing:  1309 3252 1599.6   3359    6155
Waiting:      238  532 222.2    619     794
Total:       1317 3342 1607.9   3544    6304

// 所有请求的消耗时间分布区间
// 50%的用户响应时间小于3544s
// 66%的用户响应时间小于3768s
// 最大响应时间小于6304s
Percentage of the requests served within a certain time (ms)
  50%   3544
  66%   3768
  75%   3941
  80%   5227
  90%   6304
  95%   6304
  98%   6304
  99%   6304
 100%   6304 (longest request)
```

### 如果应用有登陆认证
像我需要测试的web 应用，是需要登陆认证的：要检查session中有没有用户信息。

Rails默认的session策略是[cookie-based session](http://guides.rubyonrails.org/security.html#session-storage)。

对于Rails应用来说，我们的session通常是这么命名的： `_your_site_session`

我们可以在Chrome中按F12，在Resources这个tab标签页的Cookies下找到我们的session。（听起来有些拗口～）
![Session示意]({{ site.baseurl }}/images/cookie-based-session.png "Session示意图")

找到了session我们就可以顺利通过登陆认证啦：

```
ab -C "_quality_metric_session=BAh7CkkiD3Nlc3Npb25faWQGOgZFVEkiJWRmYmE5NjY5N2Y4YzM4ODgyMGU3YzQzNGVlYWQ1YzYwBjsAVEkiEF9jc3JmX3Rva2VuBjsARkkiMTZFS0RCak1lTWlTSDJZTnNCOXhwOEpPbzV3dGx0Mm96V0lGelEzSlV1V289BjsARkkiCXVzZXIGOwBGSSIlZGZjN2U4Mjk0ZmRiMTE2ZGNiOWExYWZiODkxMmQxNzIGOwBUSSINZnVsbG5hbWUGOwBGSSIO5p2o5r6N55GeBjsAVEkiCmxvZ2luBjsARkkiEWh6eWFuZ3NodXJ1aQY7AFQ%3D--e64692bcd315f83dc5b2c103b7cdc0c7e8dd713b" -n 20 -c 2 localhost:3000/versions/596
```

-C参数代表了设定cookie值，格式是`cookie-name=value`这种形式。

### 参考
[Apache Benchmark文档](https://httpd.apache.org/docs/2.2/programs/ab.html)