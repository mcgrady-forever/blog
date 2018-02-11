---
title: Protocal Buffer trap 1-CopyFrom
date: 2017-06-22 04:22:38
tags:
categories: C++
---

最近在使用pb时遇到了这么一个问题，还是废了不少时间去定位，出现coredump的场景类似下面的代码
```
Student stu;
Teacher teacher;

stu.CopyFrom(teacher);
```

奇怪的是这样居然可以编译通过，查了pb的源码发现类都是共同的基类Message，而有两个overload版本
```
void CopyFrom(const ::google::protobuf::Message& from);
void CopyFrom(const Student& from);
```

终于恍然大悟了！！