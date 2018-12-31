---
layout: post
title:  "macOS spring项目启动速度慢问题解决"
date:   2018-12-31 21:40:00 +0800
categories: 后端开发
excerpt_separator: <!--more-->
---

同样的spring项目，同事启动项目的速度比我的快20秒左右，我们使用的是相同的电脑，环境配置也几乎相同，但是启动速度差这么多，严重影响工作效率。抱着这样的心思，我一定要解决这个问题。
<!--more-->

> 环境：
macOS mojave version 10.14.2  
spring boot 1.5.17.RELEASE  
spring boot 2.1.1.RELEASE  

### 问题复盘：
项目使用spring boot 1.5.17.RELEASE,项目本身属于一个中小型项目，在同事的电脑上启动住需要10秒左右，但是在我的电脑启动需要27秒。
有一次意外发现把电脑拿回家后，连上家里的WIFI启动速度竟然也很快。


### 解决思路：
肯定和网络有关系，起初我怀疑跟公司的maven私服，kafaka日志收集服务有关系，后来使用Intellij idea完全创建了一个全新的项目，里面什么都没有，启动速度居然需要16秒，但是连上家里的网络启动速度瞬间变成了1秒。这样就完全排除了和公司特有服务的关系。这个问题一直困扰着我，经过多次google，意外的发现了解决方案。

竟然是因为spring boot启动的时候需要解析localhost，主要的代码是`java.net.InetAddress.getLocalHost()`，但是不知道我的电脑是什么原因解析居然需要5秒，启动的时候还解析了好几次，于是20秒时间白白的花费在域名解析上。解决方案就是在/etc/hosts文件中添加

```
127.0.0.1    Jeroens-MacBook-Pro.local
::1          Jeroens-MacBook-Pro.local
```

后面的名字是主机名，可以通过hostname命令获得。

文章的解决思路来源于:
<https://amsterdam.luminis.eu/2017/05/10/fixing-slow-startup-time-java-application-running-macos-sierra/>