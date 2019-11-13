# 笔记-Redis 字典结构

#### 这篇笔记主要记录Redis字典结构的主要实现，hash过程,解决hash冲突的方式以及rehash的过程。

## Dict 主要结构

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

## hash过程

Redis计算哈希值和索引值的方法:
```
# 使用字典设置的哈希函数，计算键 key 的哈希值
hash = dict->type->hashFunction(key);
# 使用哈希表的 sizemask 属性和哈希值，计算出索引值
# 根据情况不同， ht[x] 可以是 ht[0] 或者 ht[1]
index = hash & dict->ht[x].sizemask;
```
##### 关于hash算法——Redis4.0.0版本开始算法已经从MurmurHash2更改为SipHash。（关于hash算法相关的内容会在后面的笔记中讲到。）

当有新的键值对需要添加时，