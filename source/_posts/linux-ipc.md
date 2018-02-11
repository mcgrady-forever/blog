---
title: linux-ipc
date: 2017-02-23 19:42:40
tags:
categories: Linux环境编程
---

## 简介
linux中的两种共享内存。一种是我们的IPC通信System V版本的共享内存，另外的一种是存储映射I/O（mmap函数）

在说mmap之前我们先说一下普通的读写文件的原理，进程调用read或是write后会陷入内核，因为这两个函数都是系统调用，进入系统调用后，内核开始读写文件，假设内核在读取文件，内核首先把文件读入自己的内核空间，读完之后进程在内核回归用户态，内核把读入内核内存的数据再copy进入进程的用户态内存空间。实际上我们同一份文件内容相当于读了两次，先读入内核空间，再从内核空间读入用户空间。

Linux提供了内存映射函数mmap, 它把文件内容映射到一段内存上(准确说是虚拟内存上),通过对这段内存的读取和修改, 实现对文件的读取和修改,mmap()系统调用使得进程之间可以通过映射一个普通的文件实现共享内存。普通文件映射到进程地址空间后，进程可以向访问内存的方式对文件进行访问，不需要其他系统调用(read,write)去操作。


## 共享内存-System V
``` cpp
#include <sys/types.h>
#include <sys/ipc.h>
#include <sys/shm.h>
#include <cstdio>
#include <cstdlib>

#define ARRAY_SIZE   40000
#define MALLOC_SIZE  100000
#define SHM_SIZE  100000
#define SHM_MODE  (SHM_R | SHM_W)

char array[ARRAY_SIZE];

int main()
{
	int shmid;
	char *ptr, *shmptr;

	printf("array[] from %x to %x \n", &array[0], &array[ARRAY_SIZE]);
	printf("stack around %x \n", &shmid);

	if (NULL == (ptr = (char*)malloc(MALLOC_SIZE)))
	{
		printf("malloc error");
		exit(-1);
	}
	printf("malloced from %x to %x \n", ptr, ptr+MALLOC_SIZE);

	if ((shmid = shmget(IPC_PRIVATE, SHM_SIZE, SHM_MODE)) <0)
	{
		printf("shmget error");
		exit(-1);
	}
	if ( (shmptr = (char*)shmat(shmid, NULL, 0)) == (void*)-1 )
	{
		printf("shmat error");
		exit(-1);
	}
	printf("shared memory attached from %x to %x \n",
			shmptr, shmptr+SHM_SIZE);

	if (shmctl(shmid, IPC_RMID, 0) <0)
	{
		printf("shmctl error");
		exit(-1);
	}

	return 0;
}
```

内存布局
![程序内存布局](/2017/03/23/linux-ipc/share_mem01.png)

## mmap
####a. /dev/zero####
设备/dev/zero在读时，是0字节的无限资源。此设备也接收写向它的任何数据，但忽略此
数据。我们对此设备作为IPC的兴趣在于，当对其进行存储映射时，它具有一些特殊性质：
• 创建一个未名存储区，其长度是mmap的第二个参数，将其取整为系统上的最近页长。
• 存储区都初始化为 0。
• 如果多个进程的共同祖先进程对mmap指定了MAP_SHARED标志，则这些进程可共享此存储区。

使用/dev/zero的优点是：在调用mmap创建映射区之前，无需存在一个实际文件。映射/dev/zero自动创建一个指定长度的映射区。
这种技术的缺点是：它只能由相关进程使用。如果在无关进程之间需要使用共享存储区，则必须使用shmXXX函数。

example
``` cpp
#include<stdio.h>
#include<stdlib.h>
#include<unistd.h>
#include<fcntl.h>
#include<sys/types.h>
#include<sys/stat.h>
#include<sys/mman.h>
#include<string.h>
#include <iostream>

using namespace std;

struct STU
{
	int age;
	char name[20];
	char sex;
};

const int LOOP_TIMES = 10;

int update(uint64_t* p)
{
	return (*p)++;
}

int main(int argc,char *argv[]) //这个进程用于创建映射区进行写。
{
	int fd;
	struct STU *p = NULL;
	pid_t pid;
	uint64_t* incre = NULL;

	fd = open("/dev/zero", O_RDWR, 0644);
	if(fd < 0)
	{
		perror("open");
		exit(2);
	}
	
	incre = (uint64_t*)mmap(NULL, 
							  sizeof(uint64_t),
							  PROT_READ|PROT_WRITE,
							  MAP_SHARED,
							  fd,0
							  );
	if(incre == MAP_FAILED)
	{
		perror("mmap");
		exit(3);
	}

	close(fd); //关闭不用的文件描述符

	pid = fork();
	if (pid <  0)
	{
		perror("fork");
		exit(1);
	}
	else if (pid > 0)
	{
		for (int i = 0; i < LOOP_TIMES; ++i)
		{
			cout << "parent>>"
				 << " i=" << i
				 << " ts=" << time(NULL) 
				 << " pid=" << getpid() 
				 << " incre=" << update(incre)
				 << endl;
			//sleep(1);
		}
	}
	else
	{
		for (int i = 0; i < LOOP_TIMES; ++i)
		{
			cout << "child>>"
				 << " i=" << i
				 << " ts=" << time(NULL) 
				 << " pid=" << getpid() 
				 << " incre=" << update(incre)
				 <<endl;
			//sleep(1);
		}
	}

	return 0;
}
```

####b.匿名存储映射
4.3+BSD提供了一种类似于/dev/zero的施设，称为匿名存储映射。为了使用这种功能，在调用mmap时指定MAP_ A NON标志，并将描述符指定为－1。

example
``` cpp
#include<stdio.h>
#include<stdlib.h>
#include<unistd.h>
#include<fcntl.h>
#include<sys/types.h>
#include<sys/stat.h>
#include<sys/mman.h>
#include<string.h>
#include <iostream>

using namespace std;

struct STU
{
	int age;
	char name[20];
	char sex;
};

const int LOOP_TIMES = 10;

int update(uint64_t* p)
{
	return (*p)++;
}

int main(int argc,char *argv[]) //这个进程用于创建映射区进行写。
{
	struct STU *p = NULL;
	pid_t pid;
	uint64_t* incre = NULL;
	
	incre = (uint64_t*)mmap(NULL, 
							  sizeof(uint64_t),
							  PROT_READ|PROT_WRITE,
							  MAP_ANON|MAP_SHARED,
							  -1,0
							  );
	if(incre == MAP_FAILED)
	{
		perror("mmap");
		exit(3);
	}

	pid = fork();
	if (pid <  0)
	{
		perror("fork");
		exit(1);
	}
	else if (pid > 0)
	{
		for (int i = 0; i < LOOP_TIMES; ++i)
		{
			cout << "parent>>"
				 << " i=" << i
				 << " ts=" << time(NULL) 
				 << " pid=" << getpid() 
				 << " incre=" << update(incre)
				 << endl;
			//sleep(1);
		}
	}
	else
	{
		for (int i = 0; i < LOOP_TIMES; ++i)
		{
			cout << "child>>"
				 << " i=" << i
				 << " ts=" << time(NULL) 
				 << " pid=" << getpid() 
				 << " incre=" << update(incre)
				 <<endl;
			//sleep(1);
		}
	}

	return 0;
}
```