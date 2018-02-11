---
title: java基础（一）-编译时类型和运行时类型
date: 2017-10-09 20:18:41
tags:
categories: Java
---
                                                                                          
Java的引用变量有两个类型，一个是编译时类型，一个是运行时类型，编译时类型由声明该变量时使用的类型决定，运行时类型由实际赋给该变量的对象决定。如果编译时类型和运行时类型不一致，会出现所谓的多态。因为子类其实是一种特殊的父类，因此java允许把一个子类对象直接赋值给一个父类引用变量，无须任何类型转换，或者被称为向上转型，由系统自动完成。

``` Java
class Base
{
    int i = 1;
    Base() {
        System.out.println(this.i);
        System.out.println(this.getClass());
        this.print();
    }

    void print() {
        System.out.println("Base print:"+i);
    }
}

class Derived extends Base
{
    int i = 2;
    void print() {
        System.out.println("Derived print:"+i);
    }
}

public class Test02 {
    public static void main(String[] args) {
        Derived d = new Derived();
    }
}
``` 

运行结果
``` 
1(Base.i)
class com.java_basic.test01.Derived
Derived print:0(Derived.print)
``` 

Java这里与c++不同之处是，c++的运行时多态依赖virtual关键字，c++中若在Base构造函数中调用print方法，是调用Base的print方法。有这样一条规则，**对象调用编译时类型的属性和运行时类型的方法**。