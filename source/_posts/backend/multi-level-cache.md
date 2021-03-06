title: 多级缓存的设计与实现
date: 2018-10-30 01:43:29
tags: [Java,Concurrent,Backend]
categories: Backend
toc: true
---

## Overview 总览

在高并发量的后端业务中，多级缓存是一个非常常见的设计。
事实上，多级缓存策略在计算机硬件也是普遍存在的。

![overview](/uploads/persister-multi-level-cache-overview-cache-memory-4113.jpg)

为什么会有多级缓存？
通常缓存都是比原数据要小的，设立缓存的目的很简单：某些数据经常用到，我们不想每次都去最深的地方（这里一般是指数据库）去找，所以我们就在更方便取到数据的地方把这些数据保存起来。

那么，为什么要用多级缓存呢？
对于所有的数据来说，并不是每一份数据使用的频率都一样。对于使用频率非常高的数据，我们称为热点数据。
例如，某个大V发了一条微博，告诉大家，他结婚了，那么这一条微博就是一个热点数据，热到不行的数据，我们的服务器平均下来每一毫秒要访问这条数据一万次（不夸张。。）
我前一篇文章也说了，分布式缓存 100us 的读取已经是正常成绩了，但是这个成绩完全不够支撑啊。我们必须有更快的缓存来保存这条数据，以减轻分布式缓存的压力。

对于我们业务开发者来说，显式使用 CPU 的 L1，L2，L3 显然是不现实的。那么，我们只能指望着静静插在主板上的内存条了。

现在的内存性能已经非常牛逼了，10000 mb/s 的读取速度已经非常普遍，至于延迟我还没有太多的概念。
内存那么牛逼，我们就要尽量利用好它，所以内存一般是我们多级缓存的最顶一层。

那么在这篇文章中，我就尝试着用 Java 来实现一下多级缓存。

各个语言都大同小异的，我这一年里基本都是写 Java 了，所以就直接用 Java 来写了。

## Preconditions 预设计

对于一个合格的后端服务来说，多级缓存的设计至少要包括以下几个功能：

- 支持过期清理
- 支持容量限制及逐出策略
- 支持回源及回写机制（后面我会说下回源和回写这两个概念）

### Back-to-Source 回源

一般我们说回源一般是指 CDN 层面上的回源，意思是在这个 CDN 节点上没找到相关的资源，就去到「源站」上去获取这个资源。
回到我们的缓存设计中，在缓存层面上，回源也是类似的意思：在这一层的缓存中没找到这个数据，就到下一个「数据源」去获取资源。

### Write-Back 回写

回写依托在回源的基础上。
当某一层的缓存没找到这个资源时，到下一层去找，发现找到了，然后这一层的缓存就把这个数据缓存起来。
到下一次再取这个数据的时候，就能马上取到了。

## Code 搞起

简单写了下，轻喷 [https://github.com/XhinLiang/multi-level-cache](https://github.com/XhinLiang/multi-level-cache)。

