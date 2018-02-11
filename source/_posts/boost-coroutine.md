---
title: boost_coroutine
date: 2017-03-23 02:32:48
tags:
categories: 协程
---

## 两个函数如何并发执行？
![函数并发执行](/2017/03/22/boost-coroutine/coroutine01.png)

## 执行转移机制
![执行转移机制](/2017/03/22/boost-coroutine/coroutine02.png)

``` cpp
#include <iostream>
#include <boost/coroutine/all.hpp>

typedef boost::coroutines::asymmetric_coroutine< void >::pull_type pull_coro_t;
typedef boost::coroutines::asymmetric_coroutine< void >::push_type push_coro_t;


void foo(push_coro_t & sink)
{
    std::cout << "1";
    sink();
    std::cout << "2";
    sink();
    std::cout << "3";
    sink();
    std::cout << "4";
}

int main(int argc, char * argv[])
{
    {
        pull_coro_t source(foo);
        while (source)
        {
            std::cout << "-";
            source();
        }
    }

    std::cout << "\nDone" << std::endl;

    return 0;
}
```
运行输出
```
1-2-3-4

```


``` cpp
#include <iostream>
#include <boost/coroutine/all.hpp>

using namespace std;

typedef boost::coroutines::asymmetric_coroutine< int >::pull_type pull_coro_t1;
typedef boost::coroutines::asymmetric_coroutine< int >::push_type push_coro_t1;

void foo1(push_coro_t1& sink1)
{
	cout<<"1";
	sink1(10);
	cout<<"2";
	sink1(20);
	cout<<"3";
	sink1(30);
	cout<<"4";
	sink1(40);
}

int main(int argc, char const *argv[])
{
	{
		pull_coro_t1 source1(foo1);
		while (source1)
		{
			int ret = source1.get();
			cout<<"ret: "<<ret<<endl;
			source1();
		}
	}

	cout<<"\nDone"<<endl;

	return 0;
}
```
运行输出
```
1ret: 10
2ret: 20
3ret: 30
4ret: 40
```

``` cpp
#include <iostream>
#include <boost/coroutine/all.hpp>

using namespace std;

typedef boost::coroutines::asymmetric_coroutine< int >::pull_type pull_coro_t1;
typedef boost::coroutines::asymmetric_coroutine< int >::push_type push_coro_t1;


void foo1(pull_coro_t1& sink1)
{
	cout<<"1 "<<source1.get();
	sink1();
	cout<<"2 "<<source1.get();
	sink1();
	cout<<"3 "<<source1.get();
	sink1();
	cout<<"4 "<<source1.get();
	sink1();
}

int main(int argc, char const *argv[])
{
	{
		push_coro_t1 source1(foo1);
		int c = 0;
		while (source1)
		{
			++c;
			source1();
		}
	}

	cout<<"\nDone"<<endl;

	return 0;
}
```