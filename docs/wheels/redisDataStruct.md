# 数据结构
> 以v0.091作为参考

## redis的基本数据结构
以redis.c-> *setGenericCommand*为例子作为参考
核心方法,执行dictAdd函数
```shell
int dictAdd(dict *ht, void *key, void *val)
{
    int index;
    dictEntry *entry;

    /* Get the index of the new element, or -1 if
     * the element already exists. */
    //如果key已经添加过,返回失败
    if ((index = _dictKeyIndex(ht, key)) == -1)
        return DICT_ERR;
    //创建entry,塞入table中

    /* Allocates the memory and stores key */
    entry = _dictAlloc(sizeof(*entry));
    entry->next = ht->table[index];
    ht->table[index] = entry;
    
    //set key/value 
    /* Set the hash entry fields. */
    dictSetHashKey(ht, entry, key);
    dictSetHashVal(ht, entry, val);

    ht->used++;
    
    return DICT_OK;
}
```
其中dict->table，table存储真正的数据格式
```shell
typedef struct dict {
    dictEntry **table;
    dictType *type;
    unsigned int size;
    unsigned int sizemask;
    unsigned int used;
    void *privdata;
} dict;
链表dictEntry
typedef struct dictEntry {
    void *key;
    void *val;
    struct dictEntry *next;
} dictEntry;
```
key,val是void类型,在C语言中无任何特指的类型(任何类型都可以)约等于Java中的Object







