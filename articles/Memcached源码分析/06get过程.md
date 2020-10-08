# Memcached源码阅读六 get过程

我们在前面分析过，`Memcached`从网络读取完数据，解析数据，如果是get操作，则执行get操作，下面我们分析下get操作的流程。

```
//根据key信息和key的长度信息读取数据
item *item_get(const char *key, const size_t nkey) {
    item *it;
    uint32_t hv;
    hv = hash(key, nkey, 0);//获得分段锁信息，如果未进行扩容，则item的hash表是多个hash桶共用同一个锁，即是分段的锁
    item_lock(hv);//执行分段加锁
    it = do_item_get(key, nkey, hv);//执行get操作
    item_unlock(hv);//释放锁
    return it;
}

//执行分段加锁
void item_lock(uint32_t hv) {
    uint8_t *lock_type = pthread_getspecific(item_lock_type_key);
    if (likely(*lock_type == ITEM_LOCK_GRANULAR)) {
        mutex_lock(&item_locks[(hv & hashmask(hashpower)) % item_lock_count]);//执行分段加锁
    } else {//如果在扩容过程中
        mutex_lock(&item_global_lock);
    }
}

//执行分段解锁
void item_unlock(uint32_t hv) {
    uint8_t *lock_type = pthread_getspecific(item_lock_type_key);
    if (likely(*lock_type == ITEM_LOCK_GRANULAR)) {
        mutex_unlock(&item_locks[(hv & hashmask(hashpower)) % item_lock_count]);//释放分段锁
    } else {//如果在扩容过程中
        mutex_unlock(&item_global_lock);
    }
}

//执行读取操作
item *do_item_get(const char *key, const size_t nkey, const uint32_t hv) {
    item *it = assoc_find(key, nkey, hv);//从Hash表中获取相应的结构
    if (it != NULL) {
        refcount_incr(&it->refcount);//item的引用次数+1
        if (slab_rebalance_signal && //如果正在进行slab调整，且该item是调整的对象
            ((void *)it >= slab_rebal.slab_start && (void *)it < slab_rebal.slab_end)) {
            do_item_unlink_nolock(it, hv);//将item从hashtable和LRU链中移除
            do_item_remove(it);//删除item
            it = NULL;//置为空
        }
    }

    int was_found = 0;
    //打印调试信息
    if (settings.verbose > 2) {
        if (it == NULL) {
            fprintf(stderr, "> NOT FOUND %s", key);
        } else {
            fprintf(stderr, "> FOUND KEY %s", ITEM_key(it));
            was_found++;
        }
    }

    if (it != NULL) {
        //判断Memcached初始化是否开启过期删除机制，如果开启，则执行删除相关操作
        if (settings.oldest_live != 0 && settings.oldest_live <= current_time &&
            it->time <= settings.oldest_live) {
            do_item_unlink(it, hv);//将item从hashtable和LRU链中移除           
            do_item_remove(it);//删除item
            it = NULL;
            if (was_found) {
                fprintf(stderr, " -nuked by flush");
            }
        //判断item是否过期
        } else if (it->exptime != 0 && it->exptime <= current_time) {
            do_item_unlink(it, hv);//将item从hashtable和LRU链中移除
            do_item_remove(it);//删除item
            it = NULL;
            if (was_found) {
                fprintf(stderr, " -nuked by expire");
            }
        } else {
            it->it_flags |= ITEM_FETCHED;//item的标识修改为已经读取
            DEBUG_REFCNT(it, '+');
        }
    }

    if (settings.verbose > 2)
        fprintf(stderr, "\n");

    return it;
}

//移除item
void do_item_remove(item *it) {
    MEMCACHED_ITEM_REMOVE(ITEM_key(it), it->nkey, it->nbytes);
    assert((it->it_flags & ITEM_SLABBED) == 0);//判断item的状态是否正确

    if (refcount_decr(&it->refcount) == 0) {//修改item的引用次数
        item_free(it);//释放item
    }
}

//释放item
void item_free(item *it) {
    size_t ntotal = ITEM_ntotal(it);//获得item的大小
    unsigned int clsid;
    assert((it->it_flags & ITEM_LINKED) == 0);//判断item的状态是否正确
    assert(it != heads[it->slabs_clsid]);//item不能为LRU的头指针
    assert(it != tails[it->slabs_clsid]);//item不能为LRU的尾指针
    assert(it->refcount == 0);//释放时，需保证引用次数为0

    /* so slab size changer can tell later if item is already free or not */
    clsid = it->slabs_clsid;
    it->slabs_clsid = 0;//断开slabclass的链接
    DEBUG_REFCNT(it, 'F');
    slabs_free(it, ntotal, clsid);//slabclass结构执行释放
}

//slabclass结构释放
void slabs_free(void *ptr, size_t size, unsigned int id) {
    pthread_mutex_lock(&slabs_lock);//保持同步
    do_slabs_free(ptr, size, id);//执行释放
    pthread_mutex_unlock(&slabs_lock);
}

//slabclass结构释放
static void do_slabs_free(void *ptr, const size_t size, unsigned int id) {
    slabclass_t *p;
    item *it;

    assert(((item *)ptr)->slabs_clsid == 0);//判断数据是否正确
    assert(id >= POWER_SMALLEST && id <= power_largest);//判断id合法性
    if (id < POWER_SMALLEST || id > power_largest)//判断id合法性
        return;

    MEMCACHED_SLABS_FREE(size, id, ptr);
    p = &slabclass[id];

    it = (item *)ptr;
    it->it_flags |= ITEM_SLABBED;//修改item的状态标识，修改为空闲
    it->prev = 0;//断开数据链表
    it->next = p->slots;
    if (it->next) it->next->prev = it;
    p->slots = it;

    p->sl_curr++;//空闲item个数+1
    p->requested -= size;//空间增加size
    return;
}

//将item从hashtable和LRU链中移除。是do_item_link的逆操作
void do_item_unlink(item *it, const uint32_t hv) {
    MEMCACHED_ITEM_UNLINK(ITEM_key(it), it->nkey, it->nbytes);
    mutex_lock(&cache_lock);//执行同步

    if ((it->it_flags & ITEM_LINKED) != 0) {//判断状态值，保证item还在LRU队列中
        it->it_flags &= ~ITEM_LINKED;//修改状态值
        STATS_LOCK();//更新统计信息
        stats.curr_bytes -= ITEM_ntotal(it);
        stats.curr_items -= 1;
        STATS_UNLOCK();
        assoc_delete(ITEM_key(it), it->nkey, hv);//从Hash表中删除
        item_unlink_q(it);//将item从slabclass对应的LRU队列摘除
        do_item_remove(it);//移除item
    }
    mutex_unlock(&cache_lock);
}
```

`Memcached`的get操作在读取数据时，会判断数据的有效性，使得不用额外去处理过期数据，get操作牵涉到**Slab结构，Hash表，LRU队列的更新**，我们后面专门分析这些的变更，这里暂不分析。
