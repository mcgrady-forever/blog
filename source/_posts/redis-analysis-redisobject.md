---
title: Redis源码剖析和注释（八）--- redis对象(redisObject)
date: 2018-02-10 18:39:02
tags:
categories: Redis
---

## 1. 介绍
redis中基于双端链表、简单动态字符串(sds)、字典、跳跃表、整数集合、压缩列表、快速列表等等数据结构实现了一个对象系统，并且实现了5种不同的对象，每种对象都使用了至少一种前面的数据结构，优化对象在不同场合下的使用效率。

## 2. 对象的系统的实现
### 2.1 对象的结构
对象结构robj功能：
* 为5种不同的对象类型提供同一的表示形式。
* 为不同的对象适用于不同的场景，支持同一种对象类型采用多种的数据结构方式。
* 支持引用计数，实现对象共享机制。
* 记录对象的访问时间，便于删除对象。

对象结构定义在redis 3.2版本的server.h
```
#define LRU_BITS 24
#define LRU_CLOCK_MAX ((1<<LRU_BITS)-1) /* Max value of obj->lru */
#define LRU_CLOCK_RESOLUTION 1000 /* LRU clock resolution in ms */

typedef struct redisObject {
    //对象的数据类型，占4bits，共5种类型
    unsigned type:4;        
    //对象的编码类型，占4bits，共10种类型
    unsigned encoding:4;

    //least recently used
    //实用LRU算法计算相对server.lruclock的LRU时间
    unsigned lru:LRU_BITS; /* lru time (relative to server.lruclock) */

    //引用计数
    int refcount;

    //指向底层数据实现的指针
    void *ptr;
} robj;

//type的占5种类型：
/* Object types */
#define OBJ_STRING 0    //字符串对象
#define OBJ_LIST 1      //列表对象
#define OBJ_SET 2       //集合对象
#define OBJ_ZSET 3      //有序集合对象
#define OBJ_HASH 4      //哈希对象

/* Objects encoding. Some kind of objects like Strings and Hashes can be
 * internally represented in multiple ways. The 'encoding' field of the object
 * is set to one of this fields for this object. */
// encoding 的10种类型
#define OBJ_ENCODING_RAW 0     /* Raw representation */     //原始表示方式，字符串对象是简单动态字符串
#define OBJ_ENCODING_INT 1     /* Encoded as integer */         //long类型的整数
#define OBJ_ENCODING_HT 2      /* Encoded as hash table */      //字典
#define OBJ_ENCODING_ZIPMAP 3  /* Encoded as zipmap */          //不在使用
#define OBJ_ENCODING_LINKEDLIST 4 /* Encoded as regular linked list */  //双端链表,不在使用
#define OBJ_ENCODING_ZIPLIST 5 /* Encoded as ziplist */         //压缩列表
#define OBJ_ENCODING_INTSET 6  /* Encoded as intset */          //整数集合
#define OBJ_ENCODING_SKIPLIST 7  /* Encoded as skiplist */      //跳跃表和字典
#define OBJ_ENCODING_EMBSTR 8  /* Embedded sds string encoding */   //embstr编码的简单动态字符串
#define OBJ_ENCODING_QUICKLIST 9 /* Encoded as linked list of ziplists */   //由压缩列表组成
```

### 2.2 字符串对象的底层实现类型
| 编码—encoding | 对象—ptr |
| ------------- | ------------- |
| OBJ_ENCODING_RAW | 简单动态字符串实现的字符串对象 |
| OBJ_ENCODING_INT | 整数值实现的字符串对象 |
| OBJ_ENCODING_EMBSTR |	embstr编码的简单动态字符串实现的字符串对象 |
### 2.3 列表对象的底层实现类型
| 编码—encoding	| 对象—ptr |
| ------------- | ------------- |
| OBJ_ENCODING_QUICKLIST | 快速列表实现的列表对象 |
| OBJ_ENCODING_ZIPLIST | 压缩列表实现的列表对象 |
### 2.4 集合对象的底层实现类型
| 编码—encoding | 对象—ptr |
| ------------- | ------------- |
| OBJ_ENCODING_HT | 字典实现的集合对象 |
| OBJ_ENCODING_INTSET | 整数集合实现的集合对象 |
### 2.5 哈希对象的底层实现类型
| 编码—encoding | 对象—ptr |
| ------------- | ------------- |
| OBJ_ENCODING_ZIPLIST | 压缩列表实现的哈希对象 |
| OBJ_ENCODING_HT | 字典实现的哈希对象 |
### 2.6 有序集合对象的底层实现类型
| 编码—encoding | 对象—ptr |
| ------------- | ------------- |
| OBJ_ENCODING_SKIPLIST | 跳跃表和字典实现的有序集合对象 |
| OBJ_ENCODING_ZIPLIST | 压缩列表实现的有序集合对象 |

## 3. 对象系统的重要操作
### 3.1创建一个字符串对象
* 编码为OBJ_ENCODING_RAW
```
robj *createObject(int type, void *ptr) {   //创建一个对象
    robj *o = zmalloc(sizeof(*o));          //分配空间
    o->type = type;                         //设置对象类型
    o->encoding = OBJ_ENCODING_RAW;         //设置编码方式为OBJ_ENCODING_RAW
    o->ptr = ptr;                           //设置
    o->refcount = 1;                        //引用计数为1

    /* Set the LRU to the current lruclock (minutes resolution). */
    o->lru = LRU_CLOCK();                   //计算设置当前LRU时间
    return o;
}
```

* 编码为OBJ_ENCODING_EMBSTR
```
/* Create a string object with encoding OBJ_ENCODING_EMBSTR, that is
 * an object where the sds string is actually an unmodifiable string
 * allocated in the same chunk as the object itself. */
//创建一个embstr编码的字符串对象
robj *createEmbeddedStringObject(const char *ptr, size_t len) {
    robj *o = zmalloc(sizeof(robj)+sizeof(struct sdshdr8)+len+1);   //分配空间
    struct sdshdr8 *sh = (void*)(o+1);  //o+1刚好就是struct sdshdr8的地址

    o->type = OBJ_STRING;               //类型为字符串对象
    o->encoding = OBJ_ENCODING_EMBSTR;  //设置编码类型OBJ_ENCODING_EMBSTR
    o->ptr = sh+1;                      //指向分配的sds对象，分配的len+1的空间首地址
    o->refcount = 1;                    //设置引用计数
    o->lru = LRU_CLOCK();               //计算设置当前LRU时间

    sh->len = len;                      //设置字符串长度
    sh->alloc = len;                    //设置最大容量
    sh->flags = SDS_TYPE_8;             //设置sds的类型
    if (ptr) {                          //如果传了字符串参数
        memcpy(sh->buf,ptr,len);        //将传进来的ptr保存到对象中
        sh->buf[len] = '\0';            //结束符标志
    } else {
        memset(sh->buf,0,len+1);        //否则将对象的空间初始化为0
    }
    return o;
}
```

* 两种字符串对象编码方式的区别
```
/* Create a string object with EMBSTR encoding if it is smaller than
 * REIDS_ENCODING_EMBSTR_SIZE_LIMIT, otherwise the RAW encoding is
 * used.
 *
 * The current limit of 39 is chosen so that the biggest string object
 * we allocate as EMBSTR will still fit into the 64 byte arena of jemalloc. */

//sdshdr8的大小为3个字节，加上1个结束符共4个字节
//redisObject的大小为16个字节
//redis使用jemalloc内存分配器，且jemalloc会分配8，16，32，64等字节的内存
//一个embstr固定的大小为16+3+1 = 20个字节，因此一个最大的embstr字符串为64-20 = 44字节
#define OBJ_ENCODING_EMBSTR_SIZE_LIMIT 44

// 创建字符串对象，根据长度使用不同的编码类型
// createRawStringObject和createEmbeddedStringObject的区别是：
// createRawStringObject是当字符串长度大于44字节时，robj结构和sdshdr结构在内存上是分开的
// createEmbeddedStringObject是当字符串长度小于等于44字节时，robj结构和sdshdr结构在内存上是连续的
robj *createStringObject(const char *ptr, size_t len) {
    if (len <= OBJ_ENCODING_EMBSTR_SIZE_LIMIT)
        return createEmbeddedStringObject(ptr,len);
    else
        return createRawStringObject(ptr,len);
}
```

### 3.2 字符串对象编码的优化
```
/* Try to encode a string object in order to save space */
//尝试优化字符串对象的编码方式以节约空间
robj *tryObjectEncoding(robj *o) {
    long value;
    sds s = o->ptr;
    size_t len;

    /* Make sure this is a string object, the only type we encode
     * in this function. Other types use encoded memory efficient
     * representations but are handled by the commands implementing
     * the type. */
    serverAssertWithInfo(NULL,o,o->type == OBJ_STRING);

    /* We try some specialized encoding only for objects that are
     * RAW or EMBSTR encoded, in other words objects that are still
     * in represented by an actually array of chars. */
    //如果字符串对象的编码类型为RAW或EMBSTR时，才对其重新编码
    if (!sdsEncodedObject(o)) return o;

    /* It's not safe to encode shared objects: shared objects can be shared
     * everywhere in the "object space" of Redis and may end in places where
     * they are not handled. We handle them only as values in the keyspace. */
    //如果refcount大于1，则说明对象的ptr指向的值是共享的，不对共享对象进行编码
     if (o->refcount > 1) return o;

    /* Check if we can represent this string as a long integer.
     * Note that we are sure that a string larger than 20 chars is not
     * representable as a 32 nor 64 bit integer. */
    len = sdslen(s);            //获得字符串s的长度

    //如果len小于等于20，表示符合long long可以表示的范围，且可以转换为long类型的字符串进行编码
    if (len <= 20 && string2l(s,len,&value)) {
        /* This object is encodable as a long. Try to use a shared object.
         * Note that we avoid using shared integers when maxmemory is used
         * because every object needs to have a private LRU field for the LRU
         * algorithm to work well. */
        if ((server.maxmemory == 0 ||
             (server.maxmemory_policy != MAXMEMORY_VOLATILE_LRU &&
              server.maxmemory_policy != MAXMEMORY_ALLKEYS_LRU)) &&
            value >= 0 &&
            value < OBJ_SHARED_INTEGERS)    //如果value处于共享整数的范围内
        {
            decrRefCount(o);                //原对象的引用计数减1，释放对象
            incrRefCount(shared.integers[value]); //增加共享对象的引用计数
            return shared.integers[value];      //返回一个编码为整数的字符串对象
        } else {        //如果不处于共享整数的范围
            if (o->encoding == OBJ_ENCODING_RAW) sdsfree(o->ptr);   //释放编码为OBJ_ENCODING_RAW的对象
            o->encoding = OBJ_ENCODING_INT;     //转换为OBJ_ENCODING_INT编码
            o->ptr = (void*) value;             //指针ptr指向value对象
            return o;
        }
    }

    /* If the string is small and is still RAW encoded,
     * try the EMBSTR encoding which is more efficient.
     * In this representation the object and the SDS string are allocated
     * in the same chunk of memory to save space and cache misses. */
    //如果len小于44，44是最大的编码为EMBSTR类型的字符串对象长度
    if (len <= OBJ_ENCODING_EMBSTR_SIZE_LIMIT) {
        robj *emb;

        if (o->encoding == OBJ_ENCODING_EMBSTR) return o;   //将RAW对象转换为OBJ_ENCODING_EMBSTR编码类型
        emb = createEmbeddedStringObject(s,sdslen(s)); //创建一个编码类型为OBJ_ENCODING_EMBSTR的字符串对象
        decrRefCount(o);    //释放之前的对象
        return emb;
    }

    /* We can't encode the object...
     *
     * Do the last try, and at least optimize the SDS string inside
     * the string object to require little space, in case there
     * is more than 10% of free space at the end of the SDS string.
     *
     * We do that only for relatively large strings as this branch
     * is only entered if the length of the string is greater than
     * OBJ_ENCODING_EMBSTR_SIZE_LIMIT. */
    //无法进行编码，但是如果s的未使用的空间大于使用空间的10分之1
    if (o->encoding == OBJ_ENCODING_RAW &&
        sdsavail(s) > len/10)
    {
        o->ptr = sdsRemoveFreeSpace(o->ptr);    //释放所有的未使用空间
    }

    /* Return the original object. */
    return o;
}
```

### 3.3 引用计数管理对象
```
//引用计数加1
void incrRefCount(robj *o) {
    o->refcount++;
}

//引用计数减1
void decrRefCount(robj *o) {
    if (o->refcount <= 0) serverPanic("decrRefCount against refcount <= 0");

    //当引用对象等于1时，在操作引用计数减1，直接释放对象的ptr和对象空间
    if (o->refcount == 1) {
        switch(o->type) {
        case OBJ_STRING: freeStringObject(o); break;
        case OBJ_LIST: freeListObject(o); break;
        case OBJ_SET: freeSetObject(o); break;
        case OBJ_ZSET: freeZsetObject(o); break;
        case OBJ_HASH: freeHashObject(o); break;
        default: serverPanic("Unknown object type"); break;
        }
        zfree(o);
    } else {
        o->refcount--;  //否则减1
    }
}
```

### 3.4 对象的复制，创建的对象非共享
```
//返回 复制的o对象的副本的地址，且创建的对象非共享
robj *dupStringObject(robj *o) {
    robj *d;

    serverAssert(o->type == OBJ_STRING);    //一定是OBJ_STRING类型

    switch(o->encoding) {                   //根据不同的编码类型
    case OBJ_ENCODING_RAW:
        return createRawStringObject(o->ptr,sdslen(o->ptr));        //创建的对象非共享
    case OBJ_ENCODING_EMBSTR:
        return createEmbeddedStringObject(o->ptr,sdslen(o->ptr));   //创建的对象非共享
    case OBJ_ENCODING_INT:                  //整数编码类型
        d = createObject(OBJ_STRING, NULL); //即使是共享整数范围内的整数，创建的对象也是非共享的
        d->encoding = OBJ_ENCODING_INT;
        d->ptr = o->ptr;
        return d;
    default:
        serverPanic("Wrong encoding.");
        break;
    }
}
```

### 3.5 对象的解码操作
将保存的整数值解码成字符串对象返回回来。
```
/* Get a decoded version of an encoded object (returned as a new object).
 * If the object is already raw-encoded just increment the ref count. */
//将对象是整型的解码为字符串并返回，如果是字符串编码则直接返回输入对象，只需增加引用计数
robj *getDecodedObject(robj *o) {
    robj *dec;

    if (sdsEncodedObject(o)) {  //如果是OBJ_ENCODING_RAW或OBJ_ENCODING_EMBSTR类型的对象
        incrRefCount(o);        //增加引用计数，返回一个共享的对象
        return o;
    }
    if (o->type == OBJ_STRING && o->encoding == OBJ_ENCODING_INT) { //如果是整数对象
        char buf[32];

        ll2string(buf,32,(long)o->ptr); //将整数转换为字符串
        dec = createStringObject(buf,strlen(buf));  //创建一个字符串对象
        return dec;
    } else {
        serverPanic("Unknown encoding type");
    }
}
```