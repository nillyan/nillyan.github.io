# 笔记-Redis 字典结构

#### 这篇笔记主要记录Redis字典结构的主要实现，hash过程,解决hash冲突的方式以及rehash的过程。

### Dict 主要结构

Redis源码中对于hash结构的定义位于dict.h中，其中dict结构定义如下：
```
typedef struct dict {
    dictType *type;
    void *privdata;
    dictht ht[2];
    long rehashidx; /* rehashing not in progress if rehashidx == -1 */
    unsigned long iterators; /* number of iterators currently running */
} dict;
```
该结构中有两个关键属性ht[2] 和rehashidx。

* ht属性是一个包含两个元素的数组，每个元素都是一个dictht哈希表。在非rehash阶段，redis的字典只会操作ht[0],当需要进行rehash操作时，才会同时使用到ht[1]。
* rehashidx属性用来记录rehash当前的进度，在非rehash阶段，该值=-1。

其中dictht 结构定义如下：
```
typedef struct dictht {
    dictEntry **table;
    unsigned long size;
    unsigned long sizemask;
    unsigned long used;
} dictht;
```

正常状态下（非rehash阶段）字典的key-value都存储在ht[0]中。

### Hash过程
当有新的key-value需要存储时Redis计算哈希值和索引值的方法如下:
```
# 使用字典设置的哈希函数，计算键 key 的哈希值
hash = dict->type->hashFunction(key);
# 使用哈希表的 sizemask 属性和哈希值，计算出索引值
# 根据情况不同， ht[x] 可以是 ht[0] 或者 ht[1]
index = hash & dict->ht[x].sizemask;
```
##### 关于hash算法——Redis4.0.0版本开始算法已经从MurmurHash2更改为SipHash。（关于hash算法相关的内容会在后面的笔记中讲到。）

### Hash冲突
当两个或者以上的key通过hash分配到同一个index时，会出现hash冲突。Redis中解决hash冲突的方式为链地址法：每个hash表节点都有一个next指针，出现冲突的其他节点就可以利用该指针存储在单链表中。

### Rehash过程
由于redis字典保存的键值对数量会动态变化，为了让hash表的负载因子维持一个合理的范围，当保存的键值对过大或者过小时，需要对哈希表的大小进行动态的扩展或收缩。该过程通过执行rehash的操作来完成。在Redis中这个rehash的动作并不是一次性操作整个ht[0]的节点，而是分批次的，渐进式的完成。
以扩展操作为例，Redis中通过dictExpand()实现，代码如下：
```
    /* 根据相关触发条件扩展字典 */
static int _dictExpandIfNeeded(dict *d) 
{ 
    if (dictIsRehashing(d)) return DICT_OK;  // 如果正在进行Rehash，则直接返回
    if (d->ht[0].size == 0) return dictExpand(d, DICT_HT_INITIAL_SIZE);  // 如果ht[0]字典为空，则创建并初始化ht[0]  
    /* (ht[0].used/ht[0].size)>=1前提下，
       当满足dict_can_resize=1或ht[0].used/t[0].size>5时，便对字典进行扩展 */
    if (d->ht[0].used >= d->ht[0].size && 
        (dict_can_resize || 
         d->ht[0].used/d->ht[0].size > dict_force_resize_ratio)) 
    { 
        return dictExpand(d, d->ht[0].used*2);   // 扩展字典为原来的2倍
    } 
    return DICT_OK; 
}


...

/* 计算存储Key的bucket的位置 */
static int _dictKeyIndex(dict *d, const void *key) 
{ 
    unsigned int h, idx, table; 
    dictEntry *he; 
 
    /* 检查是否需要扩展哈希表，不足则扩展 */ 
    if (_dictExpandIfNeeded(d) == DICT_ERR)  
        return -1; 
    /* 计算Key的哈希值 */ 
    h = dictHashKey(d, key); 
    for (table = 0; table <= 1; table++) { 
        idx = h & d->ht[table].sizemask;  //计算Key的bucket位置
        /* 检查节点上是否存在新增的Key */ 
        he = d->ht[table].table[idx]; 
        /* 在节点链表检查 */ 
        while(he) { 
            if (key==he->key || dictCompareKeys(d, key, he->key)) 
                return -1; 
            he = he->next;
        } 
        if (!dictIsRehashing(d)) break;  // 扫完ht[0]后，如果哈希表不在rehashing，则无需再扫ht[1]
    } 
    return idx; 
} 

...

/* 将Key插入哈希表 */
dictEntry *dictAddRaw(dict *d, void *key) 
{ 
    int index; 
    dictEntry *entry; 
    dictht *ht; 
 
    if (dictIsRehashing(d)) _dictRehashStep(d);  // 如果哈希表在rehashing，则执行单步rehash
 
    /* 调用_dictKeyIndex() 检查键是否存在，如果存在则返回NULL */ 
    if ((index = _dictKeyIndex(d, key)) == -1) 
        return NULL; 
 

    ht = dictIsRehashing(d) ? &d->ht[1] : &d->ht[0]; 
    entry = zmalloc(sizeof(*entry));   // 为新增的节点分配内存
    entry->next = ht->table[index];  //  将节点插入链表表头
    ht->table[index] = entry;   // 更新节点和桶信息
    ht->used++;    //  更新ht
 
    /* 设置新节点的键 */ 
    dictSetKey(d, entry, key); 
    return entry; 
}

...
/* 添加新键值对 */
int dictAdd(dict *d, void *key, void *val) 
{ 
    dictEntry *entry = dictAddRaw(d,key);  // 添加新键
 
    if (!entry) return DICT_ERR;  // 如果键存在，则返回失败
    dictSetVal(d, entry, val);   // 键不存在，则设置节点值
    return DICT_OK; 
}
继续dictExpand的源码实现：

int dictExpand(dict *d, unsigned long size) 
{ 
    dictht n; // 新哈希表
    unsigned long realsize = _dictNextPower(size);  // 计算扩展或缩放新哈希表的大小(调用下面函数_dictNextPower())
 
    /* 如果正在rehash或者新哈希表的大小小于现已使用，则返回error */ 
    if (dictIsRehashing(d) || d->ht[0].used > size) 
        return DICT_ERR; 
 
    /* 如果计算出哈希表size与现哈希表大小一样，也返回error */ 
    if (realsize == d->ht[0].size) return DICT_ERR; 
 
    /* 初始化新哈希表 */ 
    n.size = realsize; 
    n.sizemask = realsize-1; 
    n.table = zcalloc(realsize*sizeof(dictEntry*));  // 为table指向dictEntry 分配内存
    n.used = 0; 
 
    /* 如果ht[0] 为空，则初始化ht[0]为当前键值对的哈希表 */ 
    if (d->ht[0].table == NULL) { 
        d->ht[0] = n; 
        return DICT_OK; 
    } 
 
    /* 如果ht[0]不为空，则初始化ht[1]为当前键值对的哈希表，并开启渐进式rehash模式 */ 
    d->ht[1] = n; 
    d->rehashidx = 0; 
    return DICT_OK; 
}
...
static unsigned long _dictNextPower(unsigned long size) { 
    unsigned long i = DICT_HT_INITIAL_SIZE;  // 哈希表的初始值：4
 

    if (size >= LONG_MAX) return LONG_MAX; 
    /* 计算新哈希表的大小：第一个大于等于size的2的N 次方的数值 */
    while(1) { 
        if (i >= size) 
            return i; 
        i *= 2; 
    } 
}

```

总结一下流程大致如下：

<div align=center><img src="https://github.com/nillyan/nillyan.github.io/blob/master/blog/4e1551b0.png?raw=true" width="50%" height="50%">

1. 为ht[1]哈希表分配空间，空间大小与ht[0]当前的键值对数量有关（ht[0].used属性）：

* 如果是扩展，那么ht[1].size为第一个大于等于ht[0].used*2的N 次方的数值；
* 如果是收缩，那么ht[1].size为第一个大于等于ht[0].used的N 次方的数值。
2. 此时字典同时拥有ht[0],ht[1]两个hash表。此时将rehashidx的值设置为0，表示rehash正式开始。
3. 将ht[0]在rehashidx索引上的所有节点rehash到ht[1],当此次操作结束时，更新rehashidx的值+1。
4. 整个rehash期间，对字典的delete,find,update操作会在两个哈希表上进行（先在ht[0]上查找，当找不到时再去ht[1]上查找）。而对字典的insert操作将统一保存到ht[1]里面。
5. 当ht[0]的全部节点都rehash到ht[1]时，交换ht[0]和ht[1]的指针，并将rehashidx设为-1。

