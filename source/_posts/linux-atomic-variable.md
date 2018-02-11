---
title: atomic variable
date: 2017-09-26 04:59:06
tags:
---

http://www.alexonlinux.com/multithreaded-simple-data-type-access-and-atomic-variables#table_of_contents

##How atomic variables workBACK TO TOC

Intel的x86和x86_64架构当访问内存时，有锁住前端总线（FSB）的指令，前端总线是处理器和RAM通信的通道，锁住FSB可以阻止其它处理器或进程在当前处理器运行，也可以阻止访问RAM。

##原子变量的字节数限制
Intel的专家建议并不用每次访问内存都要锁住FSB，Intel处理器允许memcpy() and memcmp()允许在一个处理器上执行，但锁住大块内存的代价太大。实际使用中，当访问1, 2, 4, 8字节的整数时可以锁住FSB，gcc支持int、long、long long的原子操作。

##什么情况下不能使用原子变量
``` cpp
decrement_atomic_value();
if (atomic_value() == 0)
    fire_a_gun();
```

##gcc支持的原子操作函数
``` cpp
type __sync_fetch_and_add (type *ptr, type value);
type __sync_fetch_and_sub (type *ptr, type value);
type __sync_fetch_and_or (type *ptr, type value);
type __sync_fetch_and_and (type *ptr, type value);
type __sync_fetch_and_xor (type *ptr, type value);
type __sync_fetch_and_nand (type *ptr, type value);

type __sync_add_and_fetch (type *ptr, type value);
type __sync_sub_and_fetch (type *ptr, type value);
type __sync_or_and_fetch (type *ptr, type value);
type __sync_and_and_fetch (type *ptr, type value);
type __sync_xor_and_fetch (type *ptr, type value);
type __sync_nand_and_fetch (type *ptr, type value);
```

使用原子操作的例子
``` cpp
#include <stdio.h>
#include <pthread.h>
#include <unistd.h>
#include <stdlib.h>
#include <sched.h>
#include <linux/unistd.h>
#include <sys/syscall.h>
#include <errno.h>
#include <iostream>

#define INC_TO 1000000 // one million...

int global_int = 0;

pid_t gettid( void )
{
	return syscall( __NR_gettid );
}

void *thread_routine( void *arg )
{
	int i;
	int proc_num = (int)(long)arg;
	cpu_set_t set;

	CPU_ZERO( &set );
	CPU_SET( proc_num, &set );

	if (sched_setaffinity( gettid(), sizeof( cpu_set_t ), &set ))
	{
		perror( "sched_setaffinity" );
		return NULL;
	}

	for (i = 0; i < INC_TO; i++)
	{
		global_int++;
		//__sync_fetch_and_add( &global_int, 1 );
	}

	return NULL;
}

int main()
{
	int procs = 0;
	int i;
	pthread_t *thrs;

	// Getting number of CPUs
	procs = (int)sysconf( _SC_NPROCESSORS_ONLN );
	if (procs < 0)
	{
		perror( "sysconf" );
		return -1;
	}
	else
	{
		std::cout << "cpu nums:" << procs << std::endl;
	}

	thrs = (pthread_t*)malloc( sizeof( pthread_t ) * procs );
	if (thrs == NULL)
	{
		perror( "malloc" );
		return -1;
	}

	printf( "Starting %d threads...\n", procs );

	for (i = 0; i < procs; i++)
	{
		if (pthread_create( &thrs[i], NULL, thread_routine,
			(void *)(long)i ))
		{
			perror( "pthread_create" );
			procs = i;
			break;
		}
	}

	for (i = 0; i < procs; i++)
		pthread_join( thrs[i], NULL );

	free( thrs );

	printf( "After doing all the math, global_int value is: %d\n",
		global_int );
	printf( "Expected value is: %d\n", INC_TO * procs );

	return 0;
}
``` 
平均耗时90ms


同样的逻辑使用互斥锁的例子
``` cpp
#include <stdio.h>
#include <pthread.h>
#include <unistd.h>
#include <stdlib.h>
#include <sched.h>
#include <linux/unistd.h>
#include <sys/syscall.h>
#include <errno.h>
#include <iostream>
#include <unistd.h>  
#include <sys/types.h> 
#include <time.h>
#include <sys/time.h>

#define INC_TO 1000000 // one million...

int global_int = 0;
pthread_mutex_t mutex;

pid_t gettid( void )
{
	return syscall( __NR_gettid );
}

uint64_t get_time()
{
	struct timeval tv;
    gettimeofday(&tv, NULL);
    return tv.tv_sec*1000 + tv.tv_usec/1000;
}

void *thread_routine( void *arg )
{
	int i;
	int proc_num = (int)(long)arg;
	cpu_set_t set;

	CPU_ZERO( &set );
	CPU_SET( proc_num, &set );

	if (sched_setaffinity( gettid(), sizeof( cpu_set_t ), &set ))
	{
		perror( "sched_setaffinity" );
		return NULL;
	}

	for (i = 0; i < INC_TO; i++)
	{
		pthread_mutex_lock(&mutex);
		global_int++;
		pthread_mutex_unlock(&mutex);
	}

	return NULL;
}

int main()
{
	int procs = 0;
	int i;
	pthread_t *thrs;

	// Getting number of CPUs
	procs = (int)sysconf( _SC_NPROCESSORS_ONLN );
	if (procs < 0)
	{
		perror( "sysconf" );
		return -1;
	}
	else
	{
		std::cout << "cpu nums:" << procs << std::endl;
	}

	thrs = (pthread_t*)malloc( sizeof( pthread_t ) * procs );
	if (thrs == NULL)
	{
		perror( "malloc" );
		return -1;
	}

	printf( "Starting %d threads...\n", procs );
	uint64_t begin_ts = get_time();
	pthread_mutex_init(&mutex, NULL);

	for (i = 0; i < procs; i++)
	{
		if (pthread_create( &thrs[i], NULL, thread_routine,
			(void *)(long)i ))
		{
			perror( "pthread_create" );
			procs = i;
			break;
		}
	}

	for (i = 0; i < procs; i++)
		pthread_join( thrs[i], NULL );

	pthread_mutex_destroy(&mutex);
	uint64_t end_ts = get_time();

	printf("time costs:%lld\n", (long long int)end_ts-begin_ts);
	free( thrs );

	printf( "After doing all the math, global_int value is: %d\n",
		global_int );
	printf( "Expected value is: %d\n", INC_TO * procs );

	return 0;
}
``` 
平均耗时400ms,可以看出原子操作相比互斥锁有很大的性能优势.