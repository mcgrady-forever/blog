---
title: Redis源码剖析和注释（六）--- 压缩列表(ziplist)
date: 2018-02-05 08:43:29
tags:
categories: Redis
---

## 1. 跳跃表（skiplist）介绍
定义：跳跃表是一个有序链表，其中每个节点包含不定数量的链接，节点中的第i个链接构成的单向链表跳过含有少于i个链接的节点。
* 跳跃表支持平均O(logN)，最坏O(N)
* 复杂度的节点查找，大部分情况下，跳跃表的效率可以和平衡树相媲美。
* 跳跃表在redis中当数据较多时作为有序集合键的实现方式之一。

接下来，还是举个有序集合键的例子:
```
127.0.0.1:6379> ZADD score 95.5 Mike 98 Li 96 Wang  //socre是一个有序集合键
(integer) 3
127.0.0.1:6379> ZRANGE score 0 -1 WITHSCORES//所有分数按从小到大排列，每一个成员都保存了一个分数
1) "Mike"
2) "95.5"
3) "Wang"
4) "96" 
5) "Li"
6) "98"
127.0.0.1:6379> ZSCORE score Mike       //查询Mike的分值
"95.5"
```

## 2. 跳跃表的实现
redis 3.0版本将跳跃表定义在redis.h文件中，而3.2版本定义在server.h文件中
```
跳跃表节点 zskiplistNode
typedef struct zskiplistNode {
    robj *obj;                          //保存成员对象的地址
    double score;                       //分值
    struct zskiplistNode *backward;     //后退指针
    struct zskiplistLevel {
        struct zskiplistNode *forward;  //前进指针
        unsigned int span;              //跨度
    } level[];                          //层级，柔型数组
} zskiplistNode;
```

```
跳跃表表头 zskiplist（记录跳跃表信息）
typedef struct zskiplist {
    struct zskiplistNode *header, *tail;//header指向跳跃表的表头节点，tail指向跳跃表的表尾节点
    unsigned long length;       //跳跃表的长度或跳跃表节点数量计数器，除去第一个节点
    int level;                  //跳跃表中节点的最大层数，除了第一个节点
} zskiplist;
```

## 3. 幂次定律
在redis中，返回一个随机层数值，随机算法所使用的幂次定律。
* 含义是：如果某件事的发生频率和它的某个属性成幂关系，那么这个频率就可以称之为符合幂次定律。
* 表现是：少数几个事件的发生频率占了整个发生频率的大部分， 而其余的大多数事件只占整个发生频率的一个小部分。

在文件t_set.c中，zslRandomLevel函数的定义为：
```
int zslRandomLevel(void) {          //返回一个随机层数值
    int level = 1;
    //(random()&0xFFFF)只保留低两个字节的位值，其他高位全部清零，所以该值范围为0到0xFFFF
    while ((random()&0xFFFF) < (ZSKIPLIST_P * 0xFFFF))  //ZSKIPLIST_P（0.25）所以level+1的概率为0.25
        level += 1;         //返回一个1到ZSKIPLIST_MAXLEVEL（32）之间的值
    return (level<ZSKIPLIST_MAXLEVEL) ? level : ZSKIPLIST_MAXLEVEL;
}

#define ZSKIPLIST_MAXLEVEL 32 /* Should be enough for 2^32 elements */
#define ZSKIPLIST_P 0.25      /* Skiplist P = 1/4 */
```
算法性能分析：

层数至少为1，所以层数恰好等于1（不执行while循环体）的概率为 1−p.
* 层数恰好等于2的概率为 p(1−p)（执行1次while循环体）。
* 层数恰好等于3的概率为 p^2(1−p)（执行2次while循环体）。
* 层数恰好等于4的概率为 p^3(1−p)（执行3次while循环体）。
* 层数恰好等于k的概率为 p^k−1(1−p)（执行k-1次while循环体）。（k <= ZSKIPLIST_MAXLEVEL）
因此，一个节点的平均层数，或平均指针数为：

​ 1×(1−p)+2p(1−p)+3p^2(1−p)+...+kp^(k−1)(1−p)
​ =1/(1−p)

因此，
当 p = 1/2 时，每个节点的平均指针为2；
当 p = 1/4 时，每个节点的平均指针为1.33；
而redis的概率 ZSKIPLIST_P 取值就为0.25，所以跳跃表的指针开销为1.33

## 4. 跳跃表与哈希表和平衡树的比较
跳跃表和平衡树的元素都是有序排列，而哈希表不是有序的。因此在哈希表上的查找只能是单个key的查找，不适合做范围查找。
* 跳跃表和平衡树做范围查找时，跳跃表算法简单，实现方便，而平衡树逻辑复杂。
* 查找单个key，跳跃表和平衡树的平均时间复杂度都为O(logN)，而哈希表的时间复杂度为O(1)。
* 跳跃表平均每个节点包含1.33个指针，而平衡树每个节点包含2个指针，更加节约内存。

因此，在redis中实现有序集合的办法是：跳跃表+哈希表
* 跳跃表元素有序，而且可以范围查找，且比平衡树简单。
* 哈希表查找单个key时间复杂度性能高。

## 5 跳跃表基本操作
redis关于跳跃表的API都定义在t_zset.c文件中。

### 5.1 创建跳跃表 zslCreate()
```
zskiplist *zslCreate(void) {        //创建返回一个跳跃表 表头zskiplist
    int j;
    zskiplist *zsl;

    zsl = zmalloc(sizeof(*zsl));       //分配空间
    zsl->level = 1;                     //设置默认层数
    zsl->length = 0;                    //设置跳跃表长度
    //创建一个层数为32，分数为0，没有obj的跳跃表头节点
    zsl->header = zslCreateNode(ZSKIPLIST_MAXLEVEL,0,NULL);

    //跳跃表头节点初始化
    for (j = 0; j < ZSKIPLIST_MAXLEVEL; j++) {
        zsl->header->level[j].forward = NULL;   //将跳跃表头节点的所有前进指针forward设置为NULL
        zsl->header->level[j].span = 0;         //将跳跃表头节点的所有跨度span设置为0
    }
    zsl->header->backward = NULL;           //跳跃表头节点的后退指针backward置为NULL

    zsl->tail = NULL;                       //表头指向跳跃表尾节点的指针置为NULL
    return zsl;
}
```

### 5.2 插入节点 zslInsert()
//创建一个节点，分数为score，对象为obj，插入到zsl表头管理的跳跃表中，并返回新节点的地址
```
zskiplistNode *zslInsert(zskiplist *zsl, double score, robj *obj) {
    zskiplistNode *update[ZSKIPLIST_MAXLEVEL], *x;
    unsigned int rank[ZSKIPLIST_MAXLEVEL];
    int i, level;

    redisAssert(!isnan(score));

    x = zsl->header;            //获取跳跃表头结点地址，从头节点开始一层一层遍历
    for (i = zsl->level-1; i >= 0; i--) {       //遍历头节点的每个level，从下标最大层减1到0
        /* store rank that is crossed to reach the insert position */
        rank[i] = i == (zsl->level-1) ? 0 : rank[i+1];  //更新rank[i]为i+1所跨越的节点数，但是最外一层为0

        //这个while循环是查找的过程，沿着x指针遍历跳跃表，满足以下条件则要继续在当层往前走
        while (x->level[i].forward &&       //当前层的前进指针不为空且
            (x->level[i].forward->score < score ||  //当前的要插入的score大于当前层的score或
                (x->level[i].forward->score == score && //当前score等于要插入的score且
                compareStringObjects(x->level[i].forward->obj,obj) < 0))) {//当前层的对象与要插入的obj不等

            rank[i] += x->level[i].span;   //记录该层一共跨越了多少节点 加上 上一层遍历所跨越的节点数
            x = x->level[i].forward;       //指向下一个节点
        }
        //while循环跳出时，用update[i]记录第i层所遍历到的最后一个节点，遍历到i=0时，就要在该节点后要插入节点
        update[i] = x;
    }
    /* we assume the key is not already inside, since we allow duplicated
     * scores, and the re-insertion of score and redis object should never
     * happen since the caller of zslInsert() should test in the hash table
     * if the element is already inside or not.
     * zslInsert() 的调用者会确保同分值且同成员的元素不会出现，
     * 所以这里不需要进一步进行检查，可以直接创建新元素。
     */

    level = zslRandomLevel();       //获得一个随机的层数
    if (level > zsl->level) {       //如果大于当前所有节点最大的层数时
        for (i = zsl->level; i < level; i++) {
            rank[i] = 0;            //将大于等于原来zsl->level层以上的rank[]设置为0
            update[i] = zsl->header;    //将大于等于原来zsl->level层以上update[i]指向头结点
            update[i]->level[i].span = zsl->length; //update[i]已经指向头结点，将第i层的跨度设置为length
                                                    //length代表跳跃表的节点数量
        }
        zsl->level = level;     //更新表中的最大成数值
    }
    x = zslCreateNode(level,score,obj);     //创建一个节点
    for (i = 0; i < level; i++) {       //遍历每一层
        x->level[i].forward = update[i]->level[i].forward;  //设置新节点的前进指针为查找时（while循环）每一层最后一个节点的的前进指针
        update[i]->level[i].forward = x;//再把查找时每层的最后一个节点的前进指针设置为新创建的节点地址

        /* update span covered by update[i] as x is inserted here */
        x->level[i].span = update[i]->level[i].span - (rank[0] - rank[i]);  //更新插入节点的跨度值
        update[i]->level[i].span = (rank[0] - rank[i]) + 1;               //更新插入节点前一个节点的跨度值
    }

    /* increment span for untouched levels */
    for (i = level; i < zsl->level; i++) {  //如果插入节点的level小于原来的zsl->level才会执行
        update[i]->level[i].span++;             //因为高度没有达到这些层，所以只需将查找时每层最后一个节点的值的跨度加1
    }
    //设置插入节点的后退指针，就是查找时最下层的最后一个节点，该节点的地址记录在update[0]中
    //如果插入在第二个节点，也就是头结点后的位置就将后退指针设置为NULL
    x->backward = (update[0] == zsl->header) ? NULL : update[0];

    if (x->level[0].forward)    //如果x节点不是最尾部的节点
        x->level[0].forward->backward = x;  //就将x节点后面的节点的后退节点设置成为x地址
    else
        zsl->tail = x;  //否则更新表头的tail指针，指向最尾部的节点x
    zsl->length++;      //跳跃表节点计数器加1
    return x;           //返回x地址
}
```

### 4.3 删除节点
//被zslDelete, zslDeleteByScore and zslDeleteByRank使用的内部函数
```
void zslDeleteNode(zskiplist *zsl, zskiplistNode *x, zskiplistNode **update) {  //删除节点
    int i;

    //设置前进指针和跨度
    for (i = 0; i < zsl->level; i++) {              //遍历下标为0到跳跃表最大层数-1的层
        if (update[i]->level[i].forward == x) {     //如果找到该节点
            update[i]->level[i].span += x->level[i].span - 1;   //将前一个节点的跨度减1
            update[i]->level[i].forward = x->level[i].forward;
            //前一个节点的前进指针指向被删除的节点的后一个节点，跳过该节点
        } else {
            update[i]->level[i].span -= 1;  //在第i层没找到，只将该层的最后一个节点的跨度减1
        }
    }
    //设置后退指针
    if (x->level[0].forward) {      //如果被删除的前进节点不为空，后面还有节点
        x->level[0].forward->backward = x->backward;    //就将后面节点的后退指针指向被删除节点x的回退指针
    } else {
        zsl->tail = x->backward;       //否则直接将被删除的x节点的后退节点设置为表头的tail指针
    }

    //更新跳跃表最大层数
    while(zsl->level > 1 && zsl->header->level[zsl->level-1].forward == NULL)
        zsl->level--;
    zsl->length--;  //节点计数器减1
}
```

### 4.4 获取节点排名
```
unsigned long zslGetRank(zskiplist *zsl, double score, robj *o) {   //查找score和o对象在跳跃表中的排位
    zskiplistNode *x;
    unsigned long rank = 0;
    int i;

    x = zsl->header;        //遍历头结点的每一层
    for (i = zsl->level-1; i >= 0; i--) {
        while (x->level[i].forward &&
            (x->level[i].forward->score < score ||          //只要分值还小于给定的score或者
                (x->level[i].forward->score == score &&     //分值相等但是对象小于给定对象o
                compareStringObjects(x->level[i].forward->obj,o) <= 0))) {
            rank += x->level[i].span;   //更新排位值
            x = x->level[i].forward;    //指向下一个节点
        }

        /* x might be equal to zsl->header, so test if obj is non-NULL */
        //确保在第i层找到分值相同，且对象相同时才会返回排位值
        if (x->obj && equalStringObjects(x->obj,o)) {
            return rank;
        }
    }
    return 0;   //没找到
}
```

### 4.5 区间操作
```
zskiplistNode *zslFirstInRange(zskiplist *zsl, zrangespec *range) { //返回第一个分数在range范围内的节点
    zskiplistNode *x;
    int i;

    /* If everything is out of range, return early. */
    if (!zslIsInRange(zsl,range)) return NULL;  //如果不在范围内，则返回NULL，确保至少有一个节点符号range

    //判断下限
    x = zsl->header;//遍历跳跃表
    for (i = zsl->level-1; i >= 0; i--) {//遍历每一层
        /* Go forward while *OUT* of range. */
        while (x->level[i].forward &&           //如果该层有下一个节点且
            !zslValueGteMin(x->level[i].forward->score,range))//当前节点的score还小于(小于等于)range的min
                x = x->level[i].forward;        //继续指向下一个节点
    }

    /* This is an inner range, so the next node cannot be NULL. */
    x = x->level[0].forward;    //找到目标节点
    redisAssert(x != NULL);     //保证能找到

    /* Check if score <= max. */
    //判断上限
    if (!zslValueLteMax(x->score,range)) return NULL;   //该节点的分值如果比max还要大，就返回NULL
    return x;
}

zskiplistNode *zslLastInRange(zskiplist *zsl, zrangespec *range) {//返回最后一个分数在range范围内的节点
    zskiplistNode *x;
    int i;

    /* If everything is out of range, return early. */
    if (!zslIsInRange(zsl,range)) return NULL;  //如果不在范围内，则返回NULL，确保至少有一个节点符号range

    //判断上限
    x = zsl->header;//遍历跳跃表
    for (i = zsl->level-1; i >= 0; i--) {   //遍历每一层
        /* Go forward while *IN* range. */
        while (x->level[i].forward &&   //如果该层有下一个节点且
            zslValueLteMax(x->level[i].forward->score,range))//当前节点的score小于(小于等于)max
                x = x->level[i].forward;    //继续指向下一个节点
    }

    /* This is an inner range, so this node cannot be NULL. */
    redisAssert(x != NULL);//保证能找到

    /* Check if score >= min. */
    //判断下限
    if (!zslValueGteMin(x->score,range)) return NULL;   //如果找到的节点的分值比range的min还要小
    return x;
}
```