---
title: Redis源码剖析和注释（六）--- 压缩列表(ziplist)
date: 2018-02-08 05:42:34
tags:
categories: Redis
---

## 1. 介绍
压缩列表(ziplist)是哈希键的底层实现之一。它是经过特殊编码的双向链表，和整数集合(intset)一样，是为了提高内存的存储效率而设计的。当保存的对象是小整数值，或者是长度较短的字符串，那么redis就会使用压缩列表来作为哈希键的实现。
```
127.0.0.1:6379> HMSET hash name mike age 28 sex male
OK
127.0.0.1:6379> HGETALL hash
1) "name"
2) "mike"
3) "age"
4) "28"
5) "sex"
6) "male"
127.0.0.1:6379> OBJECT ENCODING hash    //编码格式为ziplist
"ziplist"
```
注：redis 3.2以后，quicklist作为列表键的实现底层实现之一，代替了压缩列表。

通过命令来查看一下：
```
127.0.0.1:6379> RPUSH list 1 2
(integer) 2
127.0.0.1:6379> LRANGE list 0 -1
1) "1"
2) "2"
127.0.0.1:6379> OBJECT ENCODING list    //是quicklist而非ziplist
"quicklist"
```

## 2. 压缩列表的结构
压缩列表是一系列特殊编码的连续内存块组成的顺序序列数据结构，可以包含任意多个节点(entry)，每一个节点可以保存一个字节数组或者一个整数值。

空间中的结构组成如下图所示：
[url01]

这里写图片描述
* zlbytes：占4个字节，记录整个压缩列表占用的内存字节数。
* zltail_offset：占4个字节，记录压缩列表尾节点entryN距离压缩列表的起始地址的字节数。
* zllength：占2个字节，记录了压缩列表的节点数量。
* entry[1-N]：长度不定，保存数据。
* zlend：占1个字节，保存一个常数255(0xFF)，标记压缩列表的末端。
redis没有提供一个结构体来保存压缩列表的信息，而是提供了一组宏来定位每个成员的地址，定义在ziplist.c文件中：
```
由于压缩列表对数据的信息访问都是以字节为单位的，所以参数zl的类型是char *类型的，因此对zl指针进行一系列的强制类型转换，以便对不用长度成员的访问。
/* Utility macros */
//  ziplist的成员宏定义
//  (*((uint32_t*)(zl))) 先对char *类型的zl进行强制类型转换成uint32_t *类型，
//  然后在用*运算符进行取内容运算，此时zl能访问的内存大小为4个字节。

#define ZIPLIST_BYTES(zl)       (*((uint32_t*)(zl)))
//将zl定位到前4个字节的bytes成员，记录这整个压缩列表的内存字节数

#define ZIPLIST_TAIL_OFFSET(zl) (*((uint32_t*)((zl)+sizeof(uint32_t))))
//将zl定位到4字节到8字节的tail_offset成员，记录着压缩列表尾节点距离列表的起始地址的偏移字节量

#define ZIPLIST_LENGTH(zl)      (*((uint16_t*)((zl)+sizeof(uint32_t)*2)))
//将zl定位到8字节到10字节的length成员，记录着压缩列表的节点数量

#define ZIPLIST_HEADER_SIZE     (sizeof(uint32_t)*2+sizeof(uint16_t))
//压缩列表表头（以上三个属性）的大小10个字节

#define ZIPLIST_ENTRY_HEAD(zl)  ((zl)+ZIPLIST_HEADER_SIZE)
//返回压缩列表首节点的地址

#define ZIPLIST_ENTRY_TAIL(zl)  ((zl)+intrev32ifbe(ZIPLIST_TAIL_OFFSET(zl)))
//返回压缩列表尾节点的地址

#define ZIPLIST_ENTRY_END(zl)   ((zl)+intrev32ifbe(ZIPLIST_BYTES(zl))-1)
//返回end成员的地址，一个字节。
```
* intrev32ifbe()是封装的宏，用来根据主机的字节序按需要进行字节大小端的转换。

## 3. 创建一个空的压缩列表
空的压缩列表就是没有节点的列表。
```
/* Create a new empty ziplist. */
unsigned char *ziplistNew(void) {   //创建并返回一个新的压缩列表
    //ZIPLIST_HEADER_SIZE是压缩列表的表头大小，1字节是末端的end大小
    unsigned int bytes = ZIPLIST_HEADER_SIZE+1;

    unsigned char *zl = zmalloc(bytes); //为表头和表尾end成员分配空间
    ZIPLIST_BYTES(zl) = intrev32ifbe(bytes);    //将bytes成员初始化为bytes=11
    ZIPLIST_TAIL_OFFSET(zl) = intrev32ifbe(ZIPLIST_HEADER_SIZE);    //空列表的tail_offset成员为表头大小为10
    ZIPLIST_LENGTH(zl) = 0;     //节点数量为0
    zl[bytes-1] = ZIP_END;      //将表尾end成员设置成默认的255
    return zl;
}
```
如下图所示：
[url02]

## 4. 压缩列表节点结构
redis对于压缩列表节点定义了一个zlentry的结构，用来管理节点的所有信息。
```
typedef struct zlentry {
    //prevrawlen 前驱节点的长度
    //prevrawlensize 编码前驱节点的长度prevrawlen所需要的字节大小
    unsigned int prevrawlensize, prevrawlen;

    //len 当前节点值长度
    //lensize 编码当前节点长度len所需的字节数
    unsigned int lensize, len;

    //当前节点header的大小 = lensize + prevrawlensize
    unsigned int headersize;

    //当前节点的编码格式
    unsigned char encoding;

    //指向当前节点的指针，以char *类型保存
    unsigned char *p;
} zlentry;                  //压缩列表节点信息的结构
```

虽然定义了这个结构体，但是根本就没有使用zlentry结构来作为压缩列表中用来存储数据节点中的结构，但是。因为，这个结构存小整数或短字符串实在是太浪费空间了。这个结构总共在32位机占用了28个字节(32位机)，在64位机占用了32个字节。这不符合压缩列表的设计目的：提高内存的利用率。因此，在redis中，并没有定义结构体来进行操作，也是定义了一些宏，压缩列表的节点真正的结构如下图所示：
[url03]

* prev_entry_len：记录前驱节点的长度。
* encoding：记录当前节点的value成员的数据类型以及长度。
* value：根据encoding来保存字节数组或整数。
接下来就分别讨论这三个成员：

接下来就分别讨论这三个成员：

### 4.1 prev_entry_len成员
prev_entry_len成员实际上就是zlentry结构中prevrawlensize,和prevrawlen这两个成员的压缩版。

prevrawlen：记录着上一个节点的长度。
prevrawlensize：记录编码prevrawlen值的所需的字节个数。
而这两个成员都是int类型，因此将两者压缩为一个成员prev_entry_len，而且分别对不同长度的前驱节点使用不同的字节数来表示。

当前驱节点的长度小于254字节，那么prev_entry_len使用1字节表示。
当前驱节点的长度大于等于255字节，那么prev_entry_len使用5个字节表示。并且用5个字节中的最高8位(最高1个字节)用 0xFE 标示prev_entry_len占用了5个字节，后四个字节才是真正保存前驱节点的长度值。
因为，对于访问的指针都是char 类型，它能访问的范围1个字节，如果这个字节的大小等于0xFE，那么就会继续访问四个字节来获取前驱节点的长度，如果该字节的大小小于0xFE，那么该字节就是要获取的前驱节点的长度。因此这样就使prev_entry_len同时具有了prevrawlen和prevrawlensize的功能，而且更加节约内存。*

redis中的代码这样描述，定义在ziplist.c中：
```
#define ZIP_BIGLEN 254 

//对前驱节点的长度len进行编码，并写入p中，如果p为空，则仅仅返回编码len所需要的字节数
static unsigned int zipPrevEncodeLength(unsigned char *p, unsigned int len) {
    if (p == NULL) {
        return (len < ZIP_BIGLEN) ? 1 : sizeof(len)+1;  //如果前驱节点的长度len字节小于254则返回1个字节，否则返回5个
    } else {
        if (len < ZIP_BIGLEN) { //如果前驱节点的长度len字节小于254
            p[0] = len;         //将len保存在p[0]中
            return 1;           //返回所需的编码数1字节
        } else {                //前驱节点的长度len大于等于255字节
            p[0] = ZIP_BIGLEN;  //添加5字节的标示，0xFE
            memcpy(p+1,&len,sizeof(len));   //从p+1的起始地址开始拷贝len，拷贝四个字节
            memrev32ifbe(p+1);
            return 1+sizeof(len);   //返回所需的编码数5字节
        }
    }
}
```

### 4.2 encoding成员
和prev_entry_len一样，encoding成员同样可以看做成zlentry结构中lensize和len的压缩版。

len：当前节点值长度。
lensize： 编码当前节点长度len所需的字节数。
同样的lensize和len都是占4个字节的，因此将两者压缩为一个成员encoding，只要encoding能够同时具有lensize和len成员的功能，而且对当前节点保存的是字节数组还是整数分别编码。

redis对字节数组和整数编码提供了一组宏定义，定义在ziplist.c中：
```
/* Different encoding/length possibilities */
#define ZIP_STR_MASK 0xc0               //1100 0000     字节数组的掩码
#define ZIP_STR_06B (0 << 6)            //0000 0000
#define ZIP_STR_14B (1 << 6)            //0100 0000
#define ZIP_STR_32B (2 << 6)            //1000 0000

#define ZIP_INT_MASK 0x30               //0011 0000     整数的掩码
#define ZIP_INT_16B (0xc0 | 0<<4)       //1100 0000
#define ZIP_INT_32B (0xc0 | 1<<4)       //1101 0000
#define ZIP_INT_64B (0xc0 | 2<<4)       //1110 0000
#define ZIP_INT_24B (0xc0 | 3<<4)       //1111 0000
#define ZIP_INT_8B 0xfe                 //1111 1110

//掩码个功能就是区分一个encoding是字节数组编码还是整数编码
//如果这个宏返回 1 就代表该enc是字节数组，如果是 0 就代表是整数的编码
#define ZIP_IS_STR(enc) (((enc) & ZIP_STR_MASK) < ZIP_STR_MASK)
```