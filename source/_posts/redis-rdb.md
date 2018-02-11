---
title: redis-rdb
date: 2017-10-17 05:20:47
tags:
---
Reids是一个内存型数据库，所有的数据都存放在内存中。这种模式的缺点就是一旦服务器关闭后会立刻丢失所有存储的数据，Redis当然要避免这种情况的发生，于是其提供了两种持久化机制：RDB和AOF。它们的功能都是将内存中存放的数据保存到磁盘文件上，等到服务器下次开启时能重载数据，以免数据丢失。今天，我们先来剖析一下RDB持久化机制。

##RDB概述
开启一个redis-cli，执行添加数据操作如下
```
127.0.0.1:6379> flushdb
OK
127.0.0.1:6379> set name chris
OK
127.0.0.1:6379> save
OK
```

我先开启了一个Redis客户端，清空数据，然后依次添加了一个键值对到数据库，最后通过SAVE文件将数据库中的数据保存到rdb文件中，实现数据的持久化，服务器会显示数据已经存放在磁盘文件上。
```
6228:M 17 Oct 05:15:16.846 * DB saved on disk
```

保存到磁盘的文件名为dump.rdb，利用od命令就能查看里面的数据
```
chris@ubuntu:~/software/redis-3.2.6/src$ od -c dump.rdb 
0000000   R   E   D   I   S   0   0   0   7 372  \t   r   e   d   i   s
0000020   -   v   e   r 005   3   .   2   .   6 372  \n   r   e   d   i
0000040   s   -   b   i   t   s 300   @ 372 005   c   t   i   m   e 302
0000060   T 364 345   Y 372  \b   u   s   e   d   -   m   e   m 302 350
0000100   H  \b  \0 376  \0 373 001  \0  \0 004   n   a   m   e 005   c
0000120   h   r   i   s 376 002 373 001  \0  \0 004   n   a   m   e 005
0000140   c   h   r   i   s 377 326 354   a   i 227   . 350   %
0000156
```
*RDB文件标识和版本号：REDIS0007
*Redis版本：redis-ver 3.2.3
*Redis系统位数（32位或64位）：redis-bits
*系统时间：ctime
*内存使用量：used-mem
*一组键值对：name-chris


##RDB文件结构
| REDIS | db_version | databases | EOF | checksum | 
