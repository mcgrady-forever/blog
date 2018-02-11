---
title: IO模型介绍
date: 2017-03-22 20:07:55
tags:
categories: network
---

## 阻塞io模型
![阻塞io模型](/2017/03/22/network-io-model/阻塞io模型.png)

## 非阻塞io模型
![非阻塞io模型](/2017/03/22/network-io-model/非阻塞io模型.png)

## io复用模型
![io复用模型](/2017/03/22/network-io-model/io复用模型.png)

## 信号驱动io模型
![信号驱动io模型](/2017/03/22/network-io-model/信号驱动io模型.png)

## 异步io模型
![异步io模型](/2017/03/22/network-io-model/异步io.png)

多啰嗦几句：
a. **阻塞和非阻塞**描述的对象是函数，指调用这个函数后是否会block进程/线程。
b. **同步/异步**描述的是执行IO操作的主体是谁，同步是由用户进程自己去执行最终的IO操作。异步是用户进程自己不关系实际IO操作的过程，只需要由内核在IO完成后通知它既可，由内核进程来执行最终的IO操作。
