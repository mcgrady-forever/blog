---
title: 谈谈Java并发（一）
date: 2017-12-15 03:25:53
tags:
---

SimpleDateFormat是一个常用的类日期格式化类，这个类不是线程安全的，在多线程环境下调用 format() 和 parse() 方法应该使用同步代码来避免问题。也借由这个问题，进一步来探讨下如何在多线程并发执行的程序中写出安全、高效的代码。

##发现问题
例如我们要把时间格式化后再使用，很容易我们可以写出这样一个DateUtil，每次处理一个请求的时候，就需要创建一个SimpleDateFormat实例对象，然后再丢弃这个对象。大量的对象就这样被创建出来，占用大量的内存和jvm空间。
```
public class DateUtil {
    
    public static  String formatDate(Date date)throws ParseException{
         SimpleDateFormat sdf = new SimpleDateFormat("yyyy-MM-dd HH:mm:ss");
        return sdf.format(date);
    }
    
    public static Date parse(String strDate) throws ParseException{
         SimpleDateFormat sdf = new SimpleDateFormat("yyyy-MM-dd HH:mm:ss");
        return sdf.parse(strDate);
    }
}
```

你也许会说，OK，那我就创建一个静态的simpleDateFormat实例，然后放到一个DateUtil类（如下）
中，在使用时直接使用这个实例进行操作，这样问题就解决了。改进后的代码如下：
```
package com.peidasoft.dateformat;

import java.text.ParseException;
import java.text.SimpleDateFormat;
import java.util.Date;

public class DateUtil {
    private static final  SimpleDateFormat sdf = new SimpleDateFormat("yyyy-MM-dd HH:mm:ss");
    
    public static  String formatDate(Date date)throws ParseException{
        return sdf.format(date);
    }
    
    public static Date parse(String strDate) throws ParseException{

        return sdf.parse(strDate);
    }
}
```

当然，这个方法的确很不错，在大部分的时间里面都会工作得很好。但当你在生产环境中使用一段时间之后，你就会发现这么一个事实：它不是线程安全的。在正常的测试情况之下，都没有问题，但一旦在生产环境中一定负载情况下时，这个问题就出来了。他会出现各种不同的情况，比如转化的时间不正确，比如报错，比如线程被挂死等等。我们看下面的测试用例，那事实说话：
```
public class DateUtilTest {
    
    public static class TestSimpleDateFormatThreadSafe extends Thread {
        @Override
        public void run() {
            while(true) {
                try {
                    this.join(2000);
                } catch (InterruptedException e1) {
                    e1.printStackTrace();
                }
                try {
                    System.out.println(this.getName()+":"+DateUtil.parse("2013-05-24 06:02:20"));
                } catch (ParseException e) {
                    e.printStackTrace();
                }
            }
        }    
    }
    
    
    public static void main(String[] args) {
        for(int i = 0; i < 3; i++){
            new TestSimpleDateFormatThreadSafe().start();
        }
            
    }
}
```
    执行输出如下：
```
Exception in thread "Thread-1" java.lang.NumberFormatException: multiple points
    at sun.misc.FloatingDecimal.readJavaFormatString(FloatingDecimal.java:1082)
    at java.lang.Double.parseDouble(Double.java:510)
    at java.text.DigitList.getDouble(DigitList.java:151)
    at java.text.DecimalFormat.parse(DecimalFormat.java:1302)
    at java.text.SimpleDateFormat.subParse(SimpleDateFormat.java:1589)
    at java.text.SimpleDateFormat.parse(SimpleDateFormat.java:1311)
    at java.text.DateFormat.parse(DateFormat.java:335)
    at com.peidasoft.orm.dateformat.DateNoStaticUtil.parse(DateNoStaticUtil.java:17)
    at com.peidasoft.orm.dateformat.DateUtilTest$TestSimpleDateFormatThreadSafe.run(DateUtilTest.java:20)
Exception in thread "Thread-0" java.lang.NumberFormatException: multiple points
    at sun.misc.FloatingDecimal.readJavaFormatString(FloatingDecimal.java:1082)
    at java.lang.Double.parseDouble(Double.java:510)
    at java.text.DigitList.getDouble(DigitList.java:151)
    at java.text.DecimalFormat.parse(DecimalFormat.java:1302)
    at java.text.SimpleDateFormat.subParse(SimpleDateFormat.java:1589)
    at java.text.SimpleDateFormat.parse(SimpleDateFormat.java:1311)
    at java.text.DateFormat.parse(DateFormat.java:335)
    at com.peidasoft.orm.dateformat.DateNoStaticUtil.parse(DateNoStaticUtil.java:17)
    at com.peidasoft.orm.dateformat.DateUtilTest$TestSimpleDateFormatThreadSafe.run(DateUtilTest.java:20)
Thread-2:Mon May 24 06:02:20 CST 2021
Thread-2:Fri May 24 06:02:20 CST 2013
Thread-2:Fri May 24 06:02:20 CST 2013
Thread-2:Fri May 24 06:02:20 CST 2013
```

##分析问题
SimpleDateFormat继承了DateFormat,在DateFormat中定义了一个protected属性的 Calendar类的对象：calendar。只是因为Calendar累的概念复杂，牵扯到时区与本地化等等，Jdk的实现中使用了成员变量来传递参数，这就造成在多线程的时候会出现错误。

##解决问题
1. 需要的时候创建新实例：

```
public class DateUtil {
    
    public static  String formatDate(Date date)throws ParseException{
         SimpleDateFormat sdf = new SimpleDateFormat("yyyy-MM-dd HH:mm:ss");
        return sdf.format(date);
    }
    
    public static Date parse(String strDate) throws ParseException{
         SimpleDateFormat sdf = new SimpleDateFormat("yyyy-MM-dd HH:mm:ss");
        return sdf.parse(strDate);
    }
}
```
说明：在需要用到SimpleDateFormat 的地方新建一个实例，不管什么时候，将有线程安全问题的对象由共享变为局部私有都能避免多线程问题，不过也加重了创建对象的负担。在一般情况下，这样其实对性能影响比不是很明显的。

2. 使用同步：同步SimpleDateFormat对象
```
public class DateSyncUtil {

    private static SimpleDateFormat sdf = new SimpleDateFormat("yyyy-MM-dd HH:mm:ss");
      
    public static String formatDate(Date date)throws ParseException{
        synchronized(sdf){
            return sdf.format(date);
        }  
    }
    
    public static Date parse(String strDate) throws ParseException{
        synchronized(sdf){
            return sdf.parse(strDate);
        }
    } 
}
```
说明：当线程较多时，当一个线程调用该方法时，其他想要调用此方法的线程就要block，多线程并发量大的时候会对性能有一定的影响。

3. 使用ThreadLocal：　
```
public class ConcurrentDateUtil {

    private static ThreadLocal<DateFormat> threadLocal = new ThreadLocal<DateFormat>() {
        @Override
        protected DateFormat initialValue() {
            return new SimpleDateFormat("yyyy-MM-dd HH:mm:ss");
        }
    };

    public static Date parse(String dateStr) throws ParseException {
        return threadLocal.get().parse(dateStr);
    }

    public static String format(Date date) {
        return threadLocal.get().format(date);
    }
}
```
  另外一种写法：
```
public class ThreadLocalDateUtil {
    private static final String date_format = "yyyy-MM-dd HH:mm:ss";
    private static ThreadLocal<DateFormat> threadLocal = new ThreadLocal<DateFormat>(); 
 
    public static DateFormat getDateFormat()   
    {  
        DateFormat df = threadLocal.get();  
        if(df==null){  
            df = new SimpleDateFormat(date_format);  
            threadLocal.set(df);  
        }  
        return df;  
    }  

    public static String formatDate(Date date) throws ParseException {
        return getDateFormat().format(date);
    }

    public static Date parse(String strDate) throws ParseException {
        return getDateFormat().parse(strDate);
    }   
}
```
说明：使用ThreadLocal, 也是将共享变量变为独享，线程独享肯定能比方法独享在并发环境中能减少不少创建对象的开销。如果对性能要求比较高的情况下，一般推荐使用这种方法。

4.抛弃JDK，使用其他类库中的时间格式化类：
a. 使用Apache commons 里的FastDateFormat，宣称是既快又线程安全的SimpleDateFormat, 可惜它只能对日期进行format, 不能对日期串进行解析。
b. 使用Joda-Time类库来处理时间相关问题

##总结
```
由SimpleDateFormat的线程安全问题，我们在遇到多线程并发问题是可以得到如下的解决方案：
1. 函数内单独创建对象
2. 采用同步代码
3. 用threadlocal变量
4. 采用线程安全的第三方库
```