---
title: java-timer
date: 2017-08-21 15:28:45
tags:
categories: Java
---

## 使用Timer类实现
``` Java
public class TimerTest {
    public static String getStringDate() {
        Date currentTime = new Date();
        SimpleDateFormat formatter = new SimpleDateFormat("yyyy-MM-dd HH:mm:ss");
        String dateString = formatter.format(currentTime);
        return dateString;
    }

    public static void main(String[] args) throws InterruptedException {
        final TimerTask timerTask1 = new TimerTask() {
            @Override
            public void run() {
                try {
                    System.out.println("task2 invoked!"+Thread.currentThread().getId());
                    System.out.println("begin: "+getStringDate());
                    //Thread.sleep(5000L);
                } catch (Exception e) {
                    e.printStackTrace();
                }
            }
        };

        final TimerTask timerTask2 = new TimerTask() {
            @Override
            public void run() {
                try {
                    System.out.println("task2 invoked!"+Thread.currentThread().getId());
                    System.out.println("begin: "+getStringDate());
                    //Thread.sleep(5000L);
                } catch (Exception e) {
                    e.printStackTrace();
                }
            }
        };

        Timer timer = new Timer();// 实例化Timer
        timer.schedule(timerTask1, 8000);
        timer.schedule(timerTask2, 10000);
    }
}
```

Timer
```

```