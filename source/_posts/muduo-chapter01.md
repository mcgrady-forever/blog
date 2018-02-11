---
title: <<Linux多线程服务端编程-使用muduo c++网络库>>笔记（一）
date: 2017-10-30 04:33:54
tags:
---

##c++主要内存问题及解决方法
1、缓冲区溢出
solution：使用vector<char>、string或自己编写的BufferClass来管理缓冲区，记录缓冲区的长度，并通过成员函数而不是裸指针修改缓冲区。

2、空悬指针/野指针
solution：shared_ptr/weak_ptr

3、重复释放
solution：scoped_ptr，只在对象析构时候释放一次

4、内存泄漏
solution：scoped_ptr，对象析构时候自动释放内存

5、不配对的new[]/delete
solution：把new[]替换为vector/scoped_array

6、内存碎片
solution：todo

##shared_ptr是否线程安全
shared_ptr计数操作是线程安全的，release 1.33.0后在大多数系统中采用无锁的原子操作实现；但对于对象本身的访问不是线程安全的。对于shared_ptr的线程安全问题，boost官方文档中作了详细说明, http://www.boost.org/doc/libs/1_65_1/libs/smart_ptr/doc/html/smart_ptr.html#shared_ptr
这里作了下总结：
1. 多个线程可以同时读一个shared_ptr实例
2. 不同的线程中可以对不同的shared_ptr实例进行“写操作”（包括operator=、reset、析构）
3. 一个shared_ptr实例被不同的线程同时读写是不安全的

代码例子：
Reading a shared_ptr from two threads
```c++
shared_ptr<int> p(new int(42));

// thread A
shared_ptr<int> p2(p); // reads p

// thread B
shared_ptr<int> p3(p); // OK, multiple reads are safe
```

Writing different shared_ptr instances from two threads
```c++
// thread A
p.reset(new int(1912)); // writes p

// thread B
p2.reset(); // OK, writes p2
```

Reading and writing a shared_ptr from two threads
```c++
// thread A
p = p3; // reads p3, writes p

// thread B
p3.reset(); // writes p3; undefined, simultaneous read/write
```

Reading and destroying a shared_ptr from two threads
```c++
// thread A
p3 = p2; // reads p2, writes p3

// thread B
// p2 goes out of scope: undefined, the destructor is considered a "write access"
```

Writing a shared_ptr from two threads
```c++
// thread A
p3.reset(new int(1));

// thread B
p3.reset(new int(2)); // undefined, multiple writes
```