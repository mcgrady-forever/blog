---
title: 谈谈Java并发（二）
date: 2018-01-03 01:50:15
tags:
---

http://www.cnblogs.com/shamo89/p/6684960.html

```
public class Concurrent {

    public final static int THREAD_POOL_SIZE = 5;

    public static Map<String, Integer> crunchifyHashTableObject = null;
    public static Map<String, Integer> crunchifySynchronizedMapObject = null;
    public static Map<String, Integer> crunchifyConcurrentHashMapObject = null;

    public static void main(String[] args) throws InterruptedException {

        // Test with Hashtable Object
        crunchifyHashTableObject = new Hashtable<String, Integer>();
        crunchifyPerformTest(crunchifyHashTableObject);

        // Test with synchronizedMap Object
        crunchifySynchronizedMapObject = Collections.synchronizedMap(new HashMap<String, Integer>());
        crunchifyPerformTest(crunchifySynchronizedMapObject);

        // Test with ConcurrentHashMap Object
        crunchifyConcurrentHashMapObject = new ConcurrentHashMap<String, Integer>();
        crunchifyPerformTest(crunchifyConcurrentHashMapObject);

    }

    public static void crunchifyPerformTest(final Map<String, Integer> crunchifyThreads) throws InterruptedException {

        System.out.println("Test started for: " + crunchifyThreads.getClass());
        long averageTime = 0;
        for (int i = 0; i < 5; i++) {

            long startTime = System.nanoTime();
            ExecutorService crunchifyExServer = Executors.newFixedThreadPool(THREAD_POOL_SIZE);

            for (int j = 0; j < THREAD_POOL_SIZE; j++) {
                crunchifyExServer.execute(new Runnable() {
                    @SuppressWarnings("unused")
                    public void run() {

                        for (int i = 0; i < 500000; i++) {
                            Integer crunchifyRandomNumber = (int) Math.ceil(Math.random() * 550000);

                            // Retrieve value. We are not using it anywhere
                            Integer crunchifyValue = crunchifyThreads.get(String.valueOf(crunchifyRandomNumber));

                            // Put value
                            crunchifyThreads.put(String.valueOf(crunchifyRandomNumber), crunchifyRandomNumber);
                        }
                    }
                });
            }

            // Make sure executor stops
            crunchifyExServer.shutdown();

            // Blocks until all tasks have completed execution after a shutdown request
            crunchifyExServer.awaitTermination(Long.MAX_VALUE, TimeUnit.DAYS);

            long entTime = System.nanoTime();
            long totalTime = (entTime - startTime) / 1000000L;
            averageTime += totalTime;
            System.out.println("2500K entried added/retrieved in " + totalTime + " ms");
        }
        System.out.println("For " + crunchifyThreads.getClass() + " the average time is " + averageTime / 5 + " ms\n");
    }
}
```

执行结果
```
Test started for: class java.util.Hashtable
2500K entried added/retrieved in 2585 ms
2500K entried added/retrieved in 1645 ms
2500K entried added/retrieved in 1612 ms
2500K entried added/retrieved in 1615 ms
2500K entried added/retrieved in 1611 ms
For class java.util.Hashtable the average time is 1813 ms

Test started for: class java.util.Collections$SynchronizedMap
2500K entried added/retrieved in 2186 ms
2500K entried added/retrieved in 1730 ms
2500K entried added/retrieved in 1735 ms
2500K entried added/retrieved in 1693 ms
2500K entried added/retrieved in 1435 ms
For class java.util.Collections$SynchronizedMap the average time is 1755 ms

Test started for: class java.util.concurrent.ConcurrentHashMap
2500K entried added/retrieved in 941 ms
2500K entried added/retrieved in 2475 ms
2500K entried added/retrieved in 768 ms
2500K entried added/retrieved in 641 ms
2500K entried added/retrieved in 770 ms
For class java.util.concurrent.ConcurrentHashMap the average time is 1119 ms
```

性能表现：ConcurrentHashMap > SynchronizedMap > Hashtable