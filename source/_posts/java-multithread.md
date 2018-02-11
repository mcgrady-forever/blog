---
title: java-multithread
date: 2017-04-25 05:11:45
tags:
categories: Java
---

## 
``` Java
public class test01 {
    public static void main(String[] args) throws InterruptedException {
        class Counter {
            private int count = 0;

            public void incre() {
                count += 1;
            }

            public int getCount() {
                return count;
            }
        }

        final Counter counter = new Counter();
        class CountingThread extends Thread {
            public void run() {
                for (int i = 0; i < 200000; ++i) {
                    System.out.println("thread id:"+this.getId());
                    counter.incre();
                }
            }
        }

        Thread thread1 = new CountingThread();
        Thread thread2 = new CountingThread();

        thread2.start();
        thread1.start();
        thread1.join();
        thread2.join();

        System.out.println("counter: "+counter.getCount());
    }
}
```

```
counter: 399998
```

使用synchronized修饰方法
```
    public synchronized void incre() {
        count += 1;
    }
```

使用synchronized修饰"关键区域"
```
    public void incre() {
        synchronized (this) {
            count += 1;
        }
    }
```
synchronized不能继承——也就是说，假如一个方法在基础类中是“ 同步”的，那么在衍生类过载版本中，它不会自动进入“同步”状态。