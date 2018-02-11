---
title: Redis源码剖析和注释（二）---简单动态字符串
date: 2018-02-04 06:02:51
tags:
---

## 1.介绍
Redis兼容传统的C语言字符串类型，但没有直接使用C语言的传统的字符串（以’\0’结尾的字符数组）表示，而是自己构建了一种名为简单动态字符串（simple dynamic string，SDS）的对象。简单动态字符串在Redis数据库中应用很广泛，例如：键值对在底层就是由SDS实现的。

在redis种，有一种数据类型叫string类型，而string类型简单的说就是SDS实现的（简单理解），先通过几个命令来感受一下string类型：
```
127.0.0.1:6379> SET str1 Redis  //设置key:value = str1:Redis
OK
127.0.0.1:6379> GET str1        //获取str1的value
"Redis"
127.0.0.1:6379> TYPE str1       //获取key的存储类型 string类型
string
127.0.0.1:6379> STRLEN str1     //str1的长度为5字节
(integer) 5
```
## 2.SDS的定义
SDS定义在redis源码根目录下的sds.h/sdshdr
```
typedef char *sds;  
//sds兼容传统C风格字符串，所以起了个别名叫sds，并且可以存放sdshdr结构buf成员的地址
```

SDS也有一个表头（header）用来存放sds的信息。
```
struct sdshdr {
    int len;        //buf中已占用空间的长度
    int free;       //buf中剩余可用空间的长度
    char buf[];     //初始化sds分配的数据空间，而且是柔性数组（Flexible array member）
};
```

根据这个结构体，我们用图大概表示一下str1，如下图：
![sdshdr](/2018/02/04/redis-analysis-s2s/redis-sds01.jpg)

这里写图片描述
* len为5，表示这个sds长度为5字节。
* free为2，表示这个sds还有2个字节未使用的空间。
* buf是一个char[]的数组，分配了（len+1+free）个字节的长度，前len个字节保存着’R’、’e’、’d’、’i’、’s’这5个字符，接下来的1个字节保存着’\0’，剩下的free个字节未使用

## 3. SDS的优点
SDS本质上就是char *，因为有了表头sdshdr结构的存在，所以SDS比传统C字符串在某些方面更加优秀，并且能够兼容传统C字符串。

## 3.1 兼容C的部分函数
因为SDS兼容传统的C字符串，采用以’\0’作为结尾，所以SDS就能够使用一部分

## 3.2 二进制安全（Binary Safe）
因为传统C字符串符合ASCII编码，这种编码的操作的特点就是：遇零则止 。即，当读一个字符串时，只要遇到’\0’结尾，就认为到达末尾，就忽略’\0’结尾以后的所有字符。因此，如果传统字符串保存图片，视频等二进制文件，操作文件时就被截断了。

而SDS表头的buf被定义为字节数组，因为判断是否到达字符串结尾的依据则是表头的len成员，这意味着它可以存放任何二进制的数据和文本数据，包括’\0’，如下图：
![buf](/2018/02/04/redis-analysis-s2s/redis-sds02.jpg)

## 3.3 获得字符串长度的操作复杂度为O(1)
传统的C字符串获得长度时的做法：遍历字符串的长度，遇零则止，复杂度为O(n)。

而SDS表头的len成员就保存着字符串长度，所以获得字符串长度的操作复杂度为O(1)。

## 3.4 杜绝缓冲区溢出
因为SDS表头的free成员记录着buf字符数组中未使用空间的字节数，所以，在进行APPEND命令向字符串后追加字符串时，如果不够用会先进行内存扩展，在进行追加。

总之，正是因为表头的存在，使得redis的字符串有这么多优点


## 4. SDS源码剖析
### 4.1 SDS内存分配策略—空间预分配
空间预分配策略用于优化SDS的字符串增长操作。

如果对SDS进行修改后，SDS表头的len成员小于1MB，那么就会分配和len长度相同的未使用空间。free和len成员大小相等。
如果对SDS进行修改后，SDS的长度大于等于1MB，那么就会分配1MB的未使用空间。
通过空间预分配策略，Redis可以减少连续执行字符串增长操作所需的内存重分配次数。

源代码如下：
```
sds sdsMakeRoomFor(sds s, size_t addlen) {      //对 sds 中 buf 的长度进行扩展
    struct sdshdr *sh, *newsh;
    size_t free = sdsavail(s);  //获得s的未使用空间长度
    size_t len, newlen;

    //free的长度够用不用扩展直接返回
    if (free >= addlen) return s;  

    //free长度不够用，需要扩展
    len = sdslen(s);    //获得s字符串的长度
    sh = (void*) (s-(sizeof(struct sdshdr)));       //获取表头地址
    newlen = (len+addlen);  //扩展后的新长度

    //空间预分配     
    //#define SDS_MAX_PREALLOC (1024*1024)  
    //预先分配内存的最大长度为 1MB
    if (newlen < SDS_MAX_PREALLOC)  //新长度小于“最大预分配长度”，就直接将扩展的新长度乘2
        newlen *= 2;
    else
        newlen += SDS_MAX_PREALLOC; //新长度大于“最大预分配长度”，就在加上一个“最大预分配长度”
    newsh = zrealloc(sh, sizeof(struct sdshdr)+newlen+1);   //获得新的扩展空间的地址
    if (newsh == NULL) return NULL;

    newsh->free = newlen - len; //更新新空间的未使用的空间free
    return newsh->buf;
}
```

### 4.2 SDS内存释放策略—惰性空间释放
惰性空间释放用于优化SDS的字符串缩短操作。

* 当要缩短SDS保存的字符串时，程序并不立即使用内存充分配来回收缩短后多出来的字节，而是使用表头的free成员将这些字节记录起来，并等待将来使用。
源代码如下：
```
void sdsclear(sds s) {  //重置sds的buf空间，懒惰释放
    struct sdshdr *sh = (void*) (s-(sizeof(struct sdshdr)));
    sh->free += sh->len;    //表头free成员+已使用空间的长度len = 新的free
    sh->len = 0;            //已使用空间变为0
    sh->buf[0] = '\0';         //字符串置空
}
```

### 4.3 Redis源码注释
1. 在sds.h文件中，有两个static inline的函数，分别是sdslen和sdsavail函数，你可以把它认为是一个static的函数，加上了inline的属性。而inline关键字仅仅是建议编译器做内联展开处理，而不是强制。

2. 在sds.c中，几乎所有的函数所传的参数都是sds类型，而非表头sdshdr的地址，但是使用了通过sds指针运算从而求得表头的地址的技巧，因为sds是指向sdshdr结构buf成员的。通过sds.h/sdslen函数，来分析：

这里的关键就是sds类型是指向sdshdr结构buf成员。
* struct sdshdr结构共有三个变量，其中sds指向的buf成员是一个柔性数组，它仅仅起到占位符的作用，并不占用该结构体的大小，因此sizeof(sizeof(struct sdshdr))大小为8字节。
* 由于一个SDS类型的内存是通过动态内存分配的，所以它的内存在堆区，堆由下往上增长，因此sds指针减区sizeof(struct sdshdr)的大小就得到了表头的地址，然后就可以通过”->”访问表头的成员。如下图：
![sds&sdshdr](/2018/02/04/redis-analysis-s2s/redis-sds03.jpg)
```
static inline size_t sdslen(const sds s) {      //计算buf中字符串的长度

    struct sdshdr *sh = (void*)(s-(sizeof(struct sdshdr))); //s指针地址减去结构体大小就是结构体的地址

   return sh->len;
}
```
通过这种技巧，将表头结构隐藏起来，只对外公开sds类型。


* sds.h文件注释
```
#ifndef __SDS_H
#define __SDS_H

#define SDS_MAX_PREALLOC (1024*1024)    //预先分配内存的最大长度为1MB

#include <sys/types.h>
#include <stdarg.h>

typedef char *sds;  //sds兼容传统C风格字符串，所以起了个别名叫sds，并且可以存放sdshdr结构buf成员的地址

struct sdshdr {
    unsigned int len;   //buf中已占用空间的长度
    unsigned int free;  //buf中剩余可用空间的长度
    char buf[];         //初始化sds分配的数据空间，而且是柔性数组（Flexible array member）
};

static inline size_t sdslen(const sds s) {      //计算buf中字符串的长度
    struct sdshdr *sh = (void*)(s-(sizeof(struct sdshdr)));
    return sh->len;
}

static inline size_t sdsavail(const sds s) {    //计算buf中的未使用空间的长度
    struct sdshdr *sh = (void*)(s-(sizeof(struct sdshdr)));
    return sh->free;
}

sds sdsnewlen(const void *init, size_t initlen);    //创建一个长度为initlen的字符串,并保存init字符串中的值
sds sdsnew(const char *init);       //创建一个默认长度的字符串
sds sdsempty(void);     //建立一个只有表头，字符串为空"\0"的sds
size_t sdslen(const sds s); //计算buf中字符串的长度
sds sdsdup(const sds s);    //拷贝一份s的副本
void sdsfree(sds s);     //释放s字符串和表头
size_t sdsavail(const sds s);   //计算buf中的未使用空间的长度
sds sdsgrowzero(sds s, size_t len); //将sds扩展制定长度并赋值为0
sds sdscatlen(sds s, const void *t, size_t len);    //将字符串t追加到s表头的buf末尾，追加len个字节
sds sdscat(sds s, const char *t);       //将t字符串拼接到s的末尾
sds sdscatsds(sds s, const sds t);      //将sds追加到s末尾
sds sdscpylen(sds s, const char *t, size_t len);    //将字符串t覆盖到s表头的buf中，拷贝len个字节
sds sdscpy(sds s, const char *t);       //将字符串覆盖到s表头的buf中

sds sdscatvprintf(sds s, const char *fmt, va_list ap);  //打印函数，被 sdscatprintf 所调用
#ifdef __GNUC__
sds sdscatprintf(sds s, const char *fmt, ...)   //打印任意数量个字符串，并将这些字符串追加到给定 sds 的末尾
    __attribute__((format(printf, 2, 3)));
#else
sds sdscatprintf(sds s, const char *fmt, ...);  //打印任意数量个字符串，并将这些字符串追加到给定 sds 的末尾
#endif

sds sdscatfmt(sds s, char const *fmt, ...); //格式化打印多个字符串，并将这些字符串追加到给定 sds 的末尾
sds sdstrim(sds s, const char *cset);   //去除sds中包含有 cset字符串出现字符 的字符
void sdsrange(sds s, int start, int end);   //根据start和end区间截取字符串
void sdsupdatelen(sds s);           //更新字符串s的长度
void sdsclear(sds s);               //将字符串重置保存空间，懒惰释放
int sdscmp(const sds s1, const sds s2);     //比较两个sds的大小，相等返回0
sds *sdssplitlen(const char *s, int len, const char *sep, int seplen, int *count);  //使用长度为seplen的sep分隔符对长度为len的s进行分割，返回一个sds数组的地址，*count被设置为数组元素数量
void sdsfreesplitres(sds *tokens, int count); //释放tokens中的count个sds元素
void sdstolower(sds s);     //将sds字符串所有字符转换为小写
void sdstoupper(sds s);     //将sds字符串所有字符转换为大写
sds sdsfromlonglong(long long value);       //根据long long value创建一个SDS
sds sdscatrepr(sds s, const char *p, size_t len);   //将长度为len的字符串p以带引号""的格式追加到s末尾
sds *sdssplitargs(const char *line, int *argc); //参数拆分,主要用于 config.c 中对配置文件进行分析。
sds sdsmapchars(sds s, const char *from, const char *to, size_t setlen);    //将s中所有在 from 中的字符串，替换成 to 中的字符串
sds sdsjoin(char **argv, int argc, char *sep);  //以分隔符连接字符串子数组构成新的字符串

/* Low level functions exposed to the user API */
sds sdsMakeRoomFor(sds s, size_t addlen);   //对 sds 中 buf 的长度进行扩展
void sdsIncrLen(sds s, int incr);       //根据incr的正负，移动字符串末尾的'\0'标志
sds sdsRemoveFreeSpace(sds s);      //回收sds中的未使用空间
size_t sdsAllocSize(sds s);      //获得sds所有分配的空间

#endif
```