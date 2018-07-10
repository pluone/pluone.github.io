---
layout: post
title:  "TCP协议三次握手及四次握手"
date:   2018-07-09 18:00:00 +0800
categories: devOps
---

tcp协议是一套保证信息可靠传输的协议，这是与UDP协议的主要区别。
那么如何在不可靠的网络上保证信息的可靠传输呢？TCP协议是面向连接的，这就是解决之道。
通俗的讲TCP协议在传输信息前,通信的双方需要首先建立可靠的信道,建立连接时需要进行三次握手,而断开连接时需要进行四次握手.

## 三次握手和四次握手

<img src="/assets/tcp-connection.png" alt="TCP建立连接和断开连接" style="width: 500px;"/>
<img src="/assets/tcp-state.png" alt="TCP建立连接和断开连接过程中的状态转变" style="width: 500px;"/>
<img src="/assets/tcp-state-transition.png" alt="TCP状态转变图" style="width: 500px;"/>

## TIME_WAIT状态下客户端为什么要等待2MSL才进入CLOSED状态

 1. 如果time_wait状态下直接进入closed状态，由于网络原因，服务端可能会收不到ack报文，此时服务端会重发fin报文，但是client端已经找不到了，应用报错
 2. 如果time_wait状态下直接进入closed状态，此时系统恰好又重启了一个一摸一样的socket连接，滞留在网络中的报文就会误认为当前连接是之前的链接。

## 为什么是三次握手，而不是两次

建立连接的双方需要确认对方的ISN(initial sequence number),第一次握手请求client带着自己的ISN到server,server进行确认并且附带上自己的ISN,
此时client需要再进行一次确认,表示自己收到了server的ISN.

## 为什么是四次挥手，而不是三次

因为TCP是全双工的工作模式，连接的双方可以同时发送和接收信息，所以一方发起关闭请求，另一方进行确认后，表示一方的信道已经关闭，此时处于半关闭(half-close)状态，另一方也需要发起关闭请求。所以TCP断开连接需要四步操作。

参考:  
《TCP/IP Illustrated Volume 1, The Protocols, Second Edition》第13章