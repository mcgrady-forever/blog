---
title: TCP状态机分析（二）CLOSE_WAIT&TIME_WAIT
date: 2017-11-29 01:46:17
tags:
---

##CLOSE_WAIT 
在被动关闭连接(对方关闭)情况下，在已经接收到FIN，但是还没有发送自己的FIN的时刻，连接处于CLOSE_WAIT状态。出现大量close_wait的现象，主要原因是某种情况下对方关闭了socket链接，但是我方忙与读或者写，没有关闭连接。代码需要判断socket，一旦读到0，断开连接，read返回负，检查一下errno，如果不是AGAIN，就断开连接。

解决方法:
基本的思想就是要检测出对方已经关闭的socket，然后关闭它。
1. 代码需要判断socket，一旦read返回0，断开连接，read返回负，检查一下errno，如果不是AGAIN，也断开连接。(注:在UNP 7.5节的图7.6中，可以看到使用select能够检测出对方发送了FIN，再根据这条规则就可以处理CLOSE_WAIT的连接)
2. 给每一个socket设置一个时间戳last_update，每接收或者是发送成功数据，就用当前时间更新这个时间戳。定期检查所有的时间戳，如果时间戳与当前时间差值超过一定的阈值，就关闭这个socket。
3. 使用一个Heart-Beat线程，定期向socket发送指定格式的心跳数据包，如果接收到对方的RST报文，说明对方已经关闭了socket，那么我们也关闭这个socket。
4. 设置SO_KEEPALIVE选项，并修改内核参数,下面会详细介绍
```
a. 参数设置
查看相关的参数
sysctl -a|grep tcp_keepalive
net.ipv4.tcp_keepalive_intvl = 30
net.ipv4.tcp_keepalive_probes = 2
net.ipv4.tcp_keepalive_time = 160

参数的含义
tcp_keepalive_intvl:探测消息发送的频率
tcp_keepalive_probes:TCP发送keepalive探测以确定该连接已经断开的次数
tcp_keepalive_time:当keepalive打开的情况下，TCP发送keepalive消息的频率

设置相关的参数
sysctl -w net.ipv4.tcp_keepalive_time = 7500
也可以直接打开/etc/sysctl.conf
加入net.ipv4.tcp_keepalive_time = 7500，然后保存退出

让参数生效
sysctl -p

b. 开启keepalive属性 
int keepAlive = 1; 
setsockopt(client_fd, SOL_SOCKET, SO_KEEPALIVE, (void*)&keepAlive, sizeof(keepAlive)); 

c. 系统调用设置
这样只会影响单个连接，上面修改内核参数会影响所有设置keepalive属性的连接
#include <sys/types.h>  
#include <sys/socket.h>  
#include <netinet/tcp.h>  
  
int keepAlive = 1;          // 开启keepalive属性  
int keepIdle = 1800;        // 如该连接在1800秒内没有任何数据往来,则进行探测   
int keepInterval = 3;       // 探测时发包的时间间隔为3秒  
int keepCount = 2;          // 探测尝试的次数.如果第1次探测包就收到响应了,则后几次的不再发.  
setsockopt(client_fd, SOL_SOCKET, SO_KEEPALIVE, (void*)&keepAlive, sizeof(keepAlive));  
setsockopt(client_fd, SOL_TCP, TCP_KEEPIDLE, (void *)&keepIdle, sizeof(keepIdle));  
setsockopt(client_fd, SOL_TCP,TCP_KEEPINTVL, (void *)&keepInterval, sizeof(keepInterval));  
setsockopt(client_fd, SOL_TCP, TCP_KEEPCNT, (void *)&keepCount, sizeof(keepCount)); 
```

这篇文章对于一次生产环境遇到的CLOSE_WAIT问题分析很到位
转载：https://mp.weixin.qq.com/s?__biz=MzI4MjA4ODU0Ng==&mid=402163560&idx=1&sn=5269044286ce1d142cca1b5fed3efab1&3rd=MzA3MDU4NTYzMw==&scene=6#rd

##TIME_WAIT