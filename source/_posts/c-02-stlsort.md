---
title: 回顾C++之一-STL中的sort
date: 2017-04-17 02:50:00
tags:
categories: C++
---


## 相等 or 等价
STL充满了比较对象是否有同样的值。比如，当你用find来定位区间中第一个有特定值的对象的位置，find必须可以比较两个对象，看看一个的值是否与另一个相等。同样，当你尝试向set中插入一个新元素时，set::insert必须可以判断那个元素的值是否已经在set中了。

find算法和set的insert成员函数是很多必须判断两个值是否相同的函数的代表。但它们以不同的方式完成，find对“相同”的定义是相等，基于operator==。set::insert对“相同”的定义是等价，通常基于operator<。等价是基于在一个有序区间中对象值的相对位置。等价一般在每种标准关联容器（比如，set、multiset、map和multimap）的一部分——排序顺序方面有意义。两个对象x和y如果在关联容器c的排序顺序中没有哪个排在另一个之前，那么它们关于c使用的排序顺序有等价的值。set<Widget>的默认比较函数是less<Widget>，而默认的less<Widget>简单地对Widget调用operator<，所以w1和w2关于operator<有等价的值如果下面表达式为真：
```
(w1 < w2) // w1 < w2时它非真
&&        // 而且
(w2<w1)   // w2 < w1时它非真
```