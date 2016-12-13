---
layout: post
title: 分布式部署静态资源预编译后fingerprint不一致问题的解决思路
---

### 问题描述
为了实现线上服务高可用的目标，我们对Rails应用做了分布式部署，结果在部署服务的a和b两台服务器上，静态资源预编译后fingerprint不一致。这样的后果是我们的样式、前端js效果时好时坏，非常头疼。临时的解决方案是将一台服务器上`public/`目录下的文件手动拷贝到另外一台服务器上，但显然这并非长久之道，于是我们开始着力排查原因。

### 问题排查
为什么需要fingerprint:

>In production, Rails inserts an MD5 fingerprint into each filename so that the file is cached by the web browser

并且根据[Rails Guides](http://guides.rubyonrails.org/asset_pipeline.html#what-is-fingerprinting-and-why-should-i-care)中的介绍，fingerprint的生成是基于文件内容的，文件内容一致，则理论上来说，每次预编译后的fingerprint也应当一致。

然而在排查过程中我发现，a服务器和b服务器的`app/assets`目录下的文件内容完全一致，那么显然不是两边文件内容不一致的原因。

那么换一个角度思考，fingerprint是由[Sprockets](https://github.com/rails/sprockets)这个gem生成的，会不会是gem的原因？

我对比了两台服务器上的`Gemfile.lock`文件，发现部分gem的版本号不一致，这其中就包括了Sprockets、[Tilt](https://github.com/rtomayko/tilt)这些gem。

当我们将两台服务器上的`Gemfile.lock`文件统一后，重新预编译，结果发现，大部分文件的fingerprint一致了，但仍旧有少部分不一致的文件。

这至少说明了两台服务器上gem的版本不同是导致预编译后fingerprint不一致问题的原因之一。

我分析了剩下的一小部分fingerprint仍旧不一致的文件，大多是jQuery相关的，属于第三方的js文件。这部分文件是什么原因导致fingerprint不一致呢？

思考半天无果，无奈之下将整个工程整体打包，打算从a服务器拷贝到b服务器上试着预编译下看结果。结果在打包的过程中看到了`tmp/cache/assets`目录，瞬间想到，会不会是cache的原因？

> By default, Sprockets caches assets in tmp/cache/assets in development and production environments.

于是停止打包，将工程的tmp目录下的所有cache清空，重新预编译，至此所有文件的fingerprint完全一致，问题解决。

### 总结回顾
首先`Gemfile.lock`文件没有纳入代码仓库应当是fingerprint不一致的根本原因，关于是否应当将这个文件纳入代码仓库，这个问题在社区中被[详细讨论过](https://ruby-china.org/topics/9018)。

其次`tmp/cache/assets`目录下的缓存文件存在是fingerprint不一致的次要原因，如果文件没有修改，Rails默认就不会去重新生成fingerprint。
