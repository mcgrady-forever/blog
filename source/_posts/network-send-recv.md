---
title: 数据发送过程
date: 2017-03-20 00:15:31
tags: network
categories: network
---

## tcp发送
![tcp发送过程](/2017/03/19/network-send-recv/tcp_send.png)

从写一个TCP套接字的write调用成功返回仅表示可以使用原来的应用进程缓冲区，并不表明对端的TCP或应用进程已接受到数据

## udp发送
![udp发送过程](/2017/03/19/network-send-recv/udp_send.png)

udp是不可靠的，不会保存应用进程数据的一个副本，因此没有真正的发送缓冲区（数据被发送后，这个副本就被数据链路层丢弃）。

udp的write调用成功返回表示所写的数据报或其所有片段已加入数据链路层的输出队列（如果空间不够，应用进程也不会知道）。
