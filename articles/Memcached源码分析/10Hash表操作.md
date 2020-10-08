# Memcached源码阅读十 Hash表操作

`Memcached`的**Hash表**用来提高**数据访问性能**，通过链接法来解决Hash冲突，当Hash表中数据多余Hash表容量的1.5倍时，Hash表就会扩容，Memcached的Hash表操作没什么特别的，我们这里简单介绍下Memcached里面的Hash表操作。

```
//hash表插入元素
int assoc_insert(item *it, const uint32_t hv) {
    unsigned int oldbucket;

    //如果已经开始扩容，且扩容的桶编号大于目前的item所在桶的编号
    if (expanding &&
        (oldbucket = (hv & hashmask(hashpower - 1))) >= expand_bucket)
    {
        //这里是类似单链表的，按单链表的操作进行插入
        it->h_next = old_hashtable[oldbucket];
        old_hashtable[oldbucket] = it;
    } else {
        //已经扩容，则按新的Hash规则进行路由
        it->h_next = primary_hashtable[hv & hashmask(hashpower)];
        //这里在新的Hash表中执行单链表插入
        primary_hashtable[hv & hashmask(hashpower)] = it;
    }

    hash_items++;//元素个数+1
    if (! expanding && hash_items > (hashsize(hashpower) * 3) / 2) {
         //开始扩容
        assoc_start_expand();//唤醒扩容条件变量
    }

    MEMCACHED_ASSOC_INSERT(ITEM_key(it), it->nkey, hash_items);
    return 1;
}

//hash表删除元素
void assoc_delete(const char *key, const size_t nkey, const uint32_t hv) {
    item **before = _hashitem_before(key, nkey, hv);
    //获得item对应的桶的前一个元素
    if (*before) {
        item *nxt;
        hash_items--;//元素个数-1
        MEMCACHED_ASSOC_DELETE(key, nkey, hash_items);
        nxt = (*before)->h_next;//执行单链表的删除操作
        (*before)->h_next = 0;
        *before = nxt;
        return;
    }
    assert(*before != 0);
}
```

像Hash表的扩容，初始化等已经在其他博客中介绍过了，这里就不在阐述。
