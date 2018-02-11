---
title: java-cmd01-jps
date: 2017-05-18 19:58:12
tags:
categories: Java
---

## 位置
我们知道，很多Java命令都在jdk的JAVA_HOME/bin目录下面，jps也不例外，他就在bin目录下，所以，他是java自带的一个命令。

## 功能
jps(Java Virtual Machine Process Status Tool)是JDK1.5提供的一个显示当前所有java进程pid的命令，简单实用，非常适合在linux/unix平台上简单察看当前java进程的一些简单情况。

## 原理
jdk中的jps命令可以显示当前运行的java进程以及相关参数，它的实现机制如下：
java程序在启动以后，会在java.io.tmpdir指定的目录下，就是临时文件夹里，生成一个类似于hsperfdata_User的文件夹，这个文件夹里（在Linux中为/tmp/hsperfdata_{userName}/），有几个文件，名字就是java进程的pid，因此列出当前运行的java进程，只是把这个目录里的文件名列一下而已。至于系统的参数什么，就可以解析这几个文件获得。

## 使用
-q 只显示pid，不显示class名称,jar文件名和传递给main 方法的参数
-m 输出传递给main 方法的参数，在嵌入式jvm上可能是null
-l 输出应用程序main class的完整package名 或者 应用程序的jar文件完整路径名
-v 输出传递给JVM的参数

PS:jps命令有个地方很不好，似乎只能显示当前用户的java进程，要显示其他用户的还是只能用unix/linux的ps命令