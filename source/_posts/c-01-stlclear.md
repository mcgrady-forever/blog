---
title: 回顾C++之一-STL中的clear
date: 2017-04-17 02:50:00
tags:
categories: C++
---

通常我们可能会写下这样的代码
``` cpp
	std::vector<int> v1;
	for (int i = 0; i < 100; ++i)
	{
		v1.push_back(i);
	}

	cout<<"v1 before clear"<<endl;
	cout<< "v1.size() = "<<v1.size()<<endl;
	cout<< "v1.capacity() = "<<v1.capacity()<<endl;

	v1.clear();
	cout<<"v1 after clear"<<endl;
	cout<< "v1.size() = "<<v1.size()<<endl;
	cout<< "v1.capacity() = "<<v1.capacity()<<endl;

	std::vector<int> v2;
	for (int i = 0; i < 100; ++i)
	{
		v2.push_back(i);
	}
```
运行结果
``` cpp
v1 before clear--------------
v1.size() = 100
v1.capacity() = 128
v1 after clear--------------
v1.size() = 0
v1.capacity() = 128
```
clear并没有释放掉内存，而仅仅是将size置为0，如果需要立即释放掉内存，可以用一个空的容器和其交换，修改后的代码如下：
``` cpp
	cout<<"v2 before swap"<<endl;
	cout<< "v2.size() = "<<v2.size()<<endl;
	cout<< "v2.capacity() = "<<v2.capacity()<<endl;

	vector<int>().swap(v2);
	cout<<"v2 after swap--------------"<<endl;
	cout<< "v2.size() = "<<v2.size()<<endl;
	cout<< "v2.capacity() = "<<v2.capacity()<<endl;
```
运行结果
``` cpp
v2 before swap--------------
v2.size() = 100
v2.capacity() = 128
v2 after swap--------------
v2.size() = 0
v2.capacity() = 0
```


c++11中增加了新的方法shrink_to_fit，可以释放掉多余的内存
```	cpp
	std::vector<int> v3;
	for (int i = 0; i < 100; ++i)
	{
		v3.push_back(i);
	}

	cout<<"v3 before shrink_to_fit--------------"<<endl;
	cout<< "v3.size() = "<<v3.size()<<endl;
	cout<< "v3.capacity() = "<<v3.capacity()<<endl;

	v3.shrink_to_fit();
	cout<<"v3 after shrink_to_fit--------------"<<endl;
	cout<< "v3.size() = "<<v3.size()<<endl;
	cout<< "v3.capacity() = "<<v3.capacity()<<endl;
```
运行结果
``` cpp
v3 before shrink_to_fit--------------
v3.size() = 100
v3.capacity() = 128
v3 after shrink_to_fit--------------
v3.size() = 100
v3.capacity() = 100
```
*Question?*
不理解这个方法的作用，设置capacity的本意是预留内存，下次插入元素时不用涉及耗时的内存分配，这样下次插入元素肯定会触发重新分配内存。