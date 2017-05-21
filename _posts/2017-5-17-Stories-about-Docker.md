---
layout: post
title: Docker爬坑记
---

我们在生产环境使用Docker部署已经有段日子了，Docker为我们的部署工作提供了诸多的便利，不过同时因为经验不足我们也遇到了很多坑，因此写篇日志记录下。

### Docker踩坑故事一
年初某项目收到用户反馈，打开首页登录时*偶尔*会提示504错误。

我们使用了OpenId进行登录，我观察了一下，请求正常的时候后台是会输出这么3行日志的：

```
INFO -- : WARNING: making https request to https://login.xxx.com/openid/ without verifying server certificate; no CA path was specified.
INFO -- : WARNING: making https request to https://login.xxx.com/openid/xrds/ without verifying server certificate; no CA path was specified.
INFO -- : Generated checkid_setup request to https://login.xxx.com/openid/ using stateless mode.
```
当我们遇到504的时候，后台只能输出1~2行日志后就卡住了，随后我在后端代码中打印了OpenId的认证结果：是missing。

因为是偶现的问题，所以排查起来比较吃力。经过几次定位，发现在Docker内curl OpenId的服务器有时会卡住，而在Docker外则从未发生过这种情况，从而将问题初步定位在Docker上。

联系SA抓包后发现TCP报错*Out of order*：
![抓包截图]({{ site.baseurl }}/images/docker-1.png "抓包截图")

这时回想起OpenId的负责同学曾提醒过可能是MSS/MTU设置有问题。SA检查发现docker内MTU设置大于宿主机的MTU，将Docker MTU改小一些后异常消失。

> 最大传输单元（英语：Maximum Transmission Unit，缩写MTU）是指一种通信协议的某一层上面所能通过的最大数据包大小（以字节为单位）。最大传输单元这个参数通常与通信接口有关（网络接口卡、串口等）。
>
> 因特网协议允许IP分片，这样就可以将数据报包分成足够小的片段以通过那些最大传输单元小于该数据报原始大小的链路了。这一分片过程发生在 IP 层（OSI模型的第三层，即网络层），它使用的是将分组发送到链路上的网络接口的最大传输单元的值。原始分组的分片都被加上了标记，这样目的主机的 IP 层就能将分组重组成原始的数据报了。

### Docker踩坑故事二
我们在企业易信上开发了一个基于Spring boot的移动小程序，使用Docker部署后发现*偶尔*会出现首次进入应用后白屏的问题。

经过排查后发现在Docker容器外部署，一切回归正常。

自己尝试在客户端抓包：
![抓包截图2]({{ site.baseurl }}/images/docker-2.png "抓包截图2")

可以看到TCP 3次握手建立连接后，服务端忽然返回了一个RST报文，于是出现了白屏，可以初步判断网络出异常了。

为了不影响线上用户使用，我们计划暂时将Docker内的服务停止，改部署在Docker外，这时我们发现了一个奇怪的进程：

```
root  14360 99.9  0.0  20036   272 ?   R    May08 9583:04 /bin/bash -c nohup java -jar etest-0.0.1-SNAPSHOT.jar &
```

我们已经将Docker内的服务停止了，怎么还有这么一个进程存在，而且是root权限的。

这个进程即使SA使用root权限`kill -9`也杀不掉，并且将CPU的一个内核占用了100%：
![CPU截图]({{ site.baseurl }}/images/docker-3.png "CPU截图")

我们联系SA将云主机重启后，这个奇怪的进程消失了，偶现的白屏问题也消失了。

回想一下，这个杀不掉的进程是有天晚上我们将几条Docker命令连起来跑后出现的，包括Docker暂停、Docker启动、端口映射等，没有跑成功，当时也没去管，没想到当时潜在的隐患导致了后来线上的问题。