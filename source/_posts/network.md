---
title: TCP状态机分析（一）
date: 2017-03-09 03:49:00
tags:
categories: network
---


## tcp状态机
![tcp状态机](/2017/03/08/network/tcp_state_machine.png)

## 建立连接过程

流程图
![三次握手流程图](/2017/03/08/network/connect.png)

tcpdump抓包(listen port 1234)
```
 sudo tcpdump -i lo tcp port 1234 and host 127.0.0.1
```

客户端代码
```
       if(connect(sockfd,(struct sockaddr *)&server,sizeof(server))==-1){
              printf("connect()error\n");
              exit(1);
       }
```

服务端代码
```
       if((connectfd = accept(listenfd,(struct sockaddr*)&client,&addrlen))==-1) {
              perror("accept()error\n");
              exit(1);
       }
       else
       {
              printf("Yougot a connection from cient's ip is %s, prot is %d\n",inet_ntoa(client.sin_addr),htons(client.sin_port));
       }
       sleep(30);
```

tcpdump抓包
![tcpdump三次握手](/2017/03/08/network/tcpdump_connect.png)


netstat查看连接状态
```
netstat -apn |grep 1234 
```
![established status](/2017/03/08/network/established.png)



## 关闭连接过程

为验证CLOSE_WAIT状态，服务端accpet后sleep，客户单立即调用close

流程图
![四次挥手流程图](/2017/03/08/network/close.png)

客户端代码
```
	close(sockfd);
```

服务端代码
```
	sleep(30);
```

tcpdump抓包
![tcpdump四次挥手](/2017/03/08/network/tcpdump_close.png)

netstat查看连接状态
```
netstat -apn |grep 1234 
```
![established status](/2017/03/08/network/close_wait.png)

服务端sleep时间到后，此时客户端已close，read返回0后，服务端也调用close，进入TIME_WAIT状态。


为什么需要TIME_WAIT?
a. 当发起关闭一方的最后一个ack丢失后，对方会重传FIN，如果没有直接关闭连接，发起发就收不到重传FIN。
b. 当被动关闭一方的最后一个FIN包超时重传，如果没有TIME_WAIT状态而且此时发起方用相同的ip和port建立了新的连接，这时候会收到这个重传的包，并认为他是新连接的包，就会导致严重错误。