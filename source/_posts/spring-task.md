---
title: Spring Summary 1：Quartz定时任务为什么会被阻塞
date: 2017-02-22 19:54:05
tags:
categories: Spring
---

1. 写了两个job
``` Java
@Service
public class TestTask1 {
    private final static Logger logger = LoggerFactory.getLogger(AuthCheckTask.class);
    private long count = 0;
    private AtomicInteger number = new AtomicInteger(0);

    public   void  execute(){
        logger.info("execute TestTask1(" + new Date()+ ") begin number={}", number.get());
        try  {
            Thread.sleep(1000000);
        }catch (InterruptedException ire) {

        }
        logger.info("execute TestTask1(" + new Date()+ ") end number={}", number.get());
        number.incrementAndGet();
        //++count;
    }
}

@Service
public class TestTask2 {
    private final static Logger logger = LoggerFactory.getLogger(AuthCheckTask.class);
    private AtomicInteger number = new AtomicInteger(0);
    private static final int nCount = 5;
    private long count = 0;


    public   void  execute(){
        logger.info("execute TestTask2(" + new Date()+ ") begin number={}", number.get());
        try  {
            Thread.sleep(10000);
        }catch (InterruptedException ire) {

        }
        logger.info("execute TestTask2(" + new Date()+ ") end number={}", number.get());
        number.incrementAndGet();
        //++count;
    }
}
```

2. 任务并发执行时的配置
``` Java
<bean id="startQuertz" lazy-init="false" autowire="no" class="org.springframework.scheduling.quartz.SchedulerFactoryBean">
    <property name="triggers">
        <list>
            <ref bean="testTask1Job" />
            <ref bean="testTask2Job" />
        </list>
    </property>
</bean>


<bean id="testTask1Task" class="org.springframework.scheduling.quartz.MethodInvokingJobDetailFactoryBean">
    <!-- 调用的类 -->
    <property name="targetObject">
        <ref bean="testTask1" />
    </property>
    <!-- 调用类中的方法 -->
    <property name="targetMethod">
        <value>execute</value>
    </property>
    <property name="concurrent" value = "false"/>
</bean>
<!-- vip 订阅统计job定时 -->
<bean id="testTask1Job" class="org.springframework.scheduling.quartz.CronTriggerBean">
    <property name="jobDetail">
        <ref bean="testTask1Task" />
    </property>
    <!-- cron表达式 -->
    <property name="cronExpression">
        <value>0/5 * * * * ?</value>
    </property>
</bean>

<bean id="testTask2Task" class="org.springframework.scheduling.quartz.MethodInvokingJobDetailFactoryBean">
    <!-- 调用的类 -->
    <property name="targetObject">
        <ref bean="testTask2" />
    </property>
    <!-- 调用类中的方法 -->
    <property name="targetMethod">
        <value>execute</value>
    </property>
</bean>
<!-- vip 订阅统计job定时 -->
<bean id="testTask2Job" class="org.springframework.scheduling.quartz.CronTriggerBean">
    <property name="jobDetail">
        <ref bean="testTask2Task" />
    </property>
    <!-- cron表达式 -->
    <property name="cronExpression">
        <value>0/5 * * * * ?</value>
    </property>
</bean>
```

3. 任务并发执行结果
``` Java
[DEBUG 2017-02-23 14:43:50.001 startQuertz_Worker-3] org.quartz.core.JobRunShell.run(JobRunShell.java:201) (Calling execute on job DEFAULT.testTask2Task)
[INFO  2017-02-23 14:43:50.002 startQuertz_Worker-3] com.huyu.zhibo.task.TestTask2.execute(TestTask2.java:22) (execute TestTask2(Thu Feb 23 14:43:50 CST 2017) begin number=0)
[DEBUG 2017-02-23 14:43:55.000 startQuertz_Worker-4] org.quartz.core.JobRunShell.run(JobRunShell.java:201) (Calling execute on job DEFAULT.testTask2Task)
[INFO  2017-02-23 14:43:55.006 startQuertz_Worker-4] com.huyu.zhibo.task.TestTask2.execute(TestTask2.java:22) (execute TestTask2(Thu Feb 23 14:43:55 CST 2017) begin number=0)
[DEBUG 2017-02-23 14:43:57.462 main-SendThread(zc-jm-zookeeper04.bj:2181)] org.apache.zookeeper.ClientCnxn$SendThread.readResponse(ClientCnxn.java:714) (Got ping response for sessionid: 0x25939635b71730c after 1ms)
[DEBUG 2017-02-23 14:44:00.000 startQuertz_Worker-5] org.quartz.core.JobRunShell.run(JobRunShell.java:201) (Calling execute on job DEFAULT.testTask2Task)
[INFO  2017-02-23 14:44:00.005 startQuertz_Worker-5] com.huyu.zhibo.task.TestTask2.execute(TestTask2.java:22) (execute TestTask2(Thu Feb 23 14:44:00 CST 2017) begin number=0)
[DEBUG 2017-02-23 14:44:05.000 startQuertz_Worker-6] org.quartz.core.JobRunShell.run(JobRunShell.java:201) (Calling execute on job DEFAULT.testTask2Task)
[INFO  2017-02-23 14:44:05.001 startQuertz_Worker-6] com.huyu.zhibo.task.TestTask2.execute(TestTask2.java:22) (execute TestTask2(Thu Feb 23 14:44:05 CST 2017) begin number=0)
[DEBUG 2017-02-23 14:44:07.462 main-SendThread(zc-jm-zookeeper04.bj:2181)] org.apache.zookeeper.ClientCnxn$SendThread.readResponse(ClientCnxn.java:714) (Got ping response for sessionid: 0x25939635b71730c after 0ms)
[DEBUG 2017-02-23 14:44:10.001 startQuertz_Worker-7] org.quartz.core.JobRunShell.run(JobRunShell.java:201) (Calling execute on job DEFAULT.testTask2Task)
[INFO  2017-02-23 14:44:10.004 startQuertz_Worker-7] com.huyu.zhibo.task.TestTask2.execute(TestTask2.java:22) (execute TestTask2(Thu Feb 23 14:44:10 CST 2017) begin number=0)
[DEBUG 2017-02-23 14:44:15.000 startQuertz_Worker-8] org.quartz.core.JobRunShell.run(JobRunShell.java:201) (Calling execute on job DEFAULT.testTask2Task)
[INFO  2017-02-23 14:44:15.001 startQuertz_Worker-8] com.huyu.zhibo.task.TestTask2.execute(TestTask2.java:22) (execute TestTask2(Thu Feb 23 14:44:15 CST 2017) begin number=0)
[DEBUG 2017-02-23 14:44:17.463 main-SendThread(zc-jm-zookeeper04.bj:2181)] org.apache.zookeeper.ClientCnxn$SendThread.readResponse(ClientCnxn.java:714) (Got ping response for sessionid: 0x25939635b71730c after 0ms)
[DEBUG 2017-02-23 14:44:20.000 startQuertz_Worker-9] org.quartz.core.JobRunShell.run(JobRunShell.java:201) (Calling execute on job DEFAULT.testTask2Task)
[INFO  2017-02-23 14:44:20.001 startQuertz_Worker-9] com.huyu.zhibo.task.TestTask2.execute(TestTask2.java:22) (execute TestTask2(Thu Feb 23 14:44:20 CST 2017) begin number=0)
[DEBUG 2017-02-23 14:44:25.001 startQuertz_Worker-10] org.quartz.core.JobRunShell.run(JobRunShell.java:201) (Calling execute on job DEFAULT.testTask2Task)
[INFO  2017-02-23 14:44:25.002 startQuertz_Worker-10] com.huyu.zhibo.task.TestTask2.execute(TestTask2.java:22) (execute TestTask2(Thu Feb 23 14:44:25 CST 2017) begin number=0)
```

4. task1修改为同步执行
``` Java
<property name="concurrent" value = "false"/>
```

5. task1修改为同步执行结果
``` Java
[INFO] Started Jetty Server
[DEBUG 2017-02-23 14:48:25.000 startQuertz_Worker-3] org.quartz.core.JobRunShell.run(JobRunShell.java:201) (Calling execute on job DEFAULT.testTask1Task)
[INFO  2017-02-23 14:48:25.001 startQuertz_Worker-3] com.huyu.zhibo.task.TestTask1.execute(TestTask1.java:20) (execute TestTask1(Thu Feb 23 14:48:25 CST 2017) begin number=0)
[DEBUG 2017-02-23 14:48:25.001 startQuertz_Worker-4] org.quartz.core.JobRunShell.run(JobRunShell.java:201) (Calling execute on job DEFAULT.testTask2Task)
[INFO  2017-02-23 14:48:25.002 startQuertz_Worker-4] com.huyu.zhibo.task.TestTask2.execute(TestTask2.java:22) (execute TestTask2(Thu Feb 23 14:48:25 CST 2017) begin number=0)
[DEBUG 2017-02-23 14:48:30.001 startQuertz_Worker-5] org.quartz.core.JobRunShell.run(JobRunShell.java:201) (Calling execute on job DEFAULT.testTask1Task)
[INFO  2017-02-23 14:48:30.002 startQuertz_Worker-5] com.huyu.zhibo.task.TestTask1.execute(TestTask1.java:20) (execute TestTask1(Thu Feb 23 14:48:30 CST 2017) begin number=0)
[DEBUG 2017-02-23 14:48:30.002 startQuertz_Worker-6] org.quartz.core.JobRunShell.run(JobRunShell.java:201) (Calling execute on job DEFAULT.testTask2Task)
[INFO  2017-02-23 14:48:30.002 startQuertz_Worker-6] com.huyu.zhibo.task.TestTask2.execute(TestTask2.java:22) (execute TestTask2(Thu Feb 23 14:48:30 CST 2017) begin number=0)
[DEBUG 2017-02-23 14:48:31.196 main-SendThread(sq-jm-stag03.bj:2181)] org.apache.zookeeper.ClientCnxn$SendThread.readResponse(ClientCnxn.java:714) (Got ping response for sessionid: 0x559396358edd1cb after 0ms)
[DEBUG 2017-02-23 14:48:35.000 startQuertz_Worker-7] org.quartz.core.JobRunShell.run(JobRunShell.java:201) (Calling execute on job DEFAULT.testTask1Task)
[INFO  2017-02-23 14:48:35.002 startQuertz_Worker-7] com.huyu.zhibo.task.TestTask1.execute(TestTask1.java:20) (execute TestTask1(Thu Feb 23 14:48:35 CST 2017) begin number=0)
[DEBUG 2017-02-23 14:48:35.002 startQuertz_Worker-8] org.quartz.core.JobRunShell.run(JobRunShell.java:201) (Calling execute on job DEFAULT.testTask2Task)
[INFO  2017-02-23 14:48:35.002 startQuertz_Worker-8] com.huyu.zhibo.task.TestTask2.execute(TestTask2.java:22) (execute TestTask2(Thu Feb 23 14:48:35 CST 2017) begin number=0)
[DEBUG 2017-02-23 14:48:40.000 startQuertz_Worker-9] org.quartz.core.JobRunShell.run(JobRunShell.java:201) (Calling execute on job DEFAULT.testTask1Task)
[INFO  2017-02-23 14:48:40.001 startQuertz_Worker-9] com.huyu.zhibo.task.TestTask1.execute(TestTask1.java:20) (execute TestTask1(Thu Feb 23 14:48:40 CST 2017) begin number=0)
[DEBUG 2017-02-23 14:48:40.001 startQuertz_Worker-10] org.quartz.core.JobRunShell.run(JobRunShell.java:201) (Calling execute on job DEFAULT.testTask2Task)
[INFO  2017-02-23 14:48:40.002 startQuertz_Worker-10] com.huyu.zhibo.task.TestTask2.execute(TestTask2.java:22) (execute TestTask2(Thu Feb 23 14:48:40 CST 2017) begin number=0)
```