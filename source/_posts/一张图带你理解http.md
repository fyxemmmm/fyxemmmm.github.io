---
title: 图解http协议
date: 2021-08-19 21:33:03
author: yuxuan
img: https://image.fyxemmmm.cn/blog/fj/images/fj-12.jpg
top: false
hide: false
cover: false
coverImg: 
toc: true
mathjax: false
summary: 加强对http协议的理解
categories: 协议
tags:
  - http
---

## 一张图带你弄懂http协议

![](https://image.fyxemmmm.cn/blog/images/http.jpg)



> http优化点：  
> **初始拥塞窗口(慢启动)** **-- 10个MMS，一开始一次性发10个，初始带宽就起来了**
> rto **一来一回，ping一下，这个rto比我们ping的值大一点，是定时器来采样的**
> rto超时 就会重新经历慢启动
> 因为慢启动 tls
> 接收端 16kb收到了之后，openssl才能解压， 2个rtt， 体验很差 --> 所以要使用fastopen
> 广播报文都只支持udp的



 ### http缓存的工作原理

缓存都有指纹 叫etag或者时间

**然后get发送请求的时候带上etag，服务端判断etag有没有变化，没有变化，就把资源返回304（只有描述信息，节约了带宽）**

 

请求里带了指纹 if-none-match，包含了文件的长度和修改的时间，做了16进制的编码

服务器通过指纹比较没问题，又发(etag)回来了，304返回

No-cache 要和服务端协商一下，得到服务端304可以使用






![缓存工作原理](https://image.fyxemmmm.cn/blog/images/etag.png)