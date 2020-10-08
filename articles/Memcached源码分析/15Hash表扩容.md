# Memcached阅读十五 Hash表扩容

`Hash表`是`Memcached`里面最重要的结构之一，其采用链接法来**处理Hash冲突**，当Hash表中的项太多时，也就是Hash冲突比较高的时候，Hash表的遍历就脱变成单链表，此时为了提供Hash的性能，Hash表需要扩容，Memcached的扩容条件是当表中元素个数超过Hash容量的1.5倍时就进行扩容，扩容过程由独立的线程来完成，扩容过程中会采用2个Hash表，将老表中的数据通过Hash算法映射到新表中，每次移动的桶的数目可以配置，默认是每次移动老表中的1个桶。

```
//hash表中增加元素
int assoc_insert(item *it, const uint32_t hv) {
    unsigned int oldbucket;

    //如果已经进行扩容且目前进行扩容还没到需要插入元素的桶，则将元素添加到旧桶中
    if (expanding &&(oldbucket = (hv & hashmask(hashpower - 1))) >= expand_bucket)
    {
        //添加元素
        it->h_next = old_hashtable[oldbucket];
        old_hashtable[oldbucket] = it;
    } else {
        //如果没扩容，或者扩容已经到了新的桶中，则添加元素到新表中
        it->h_next = primary_hashtable[hv & hashmask(hashpower)];//添加元素
        primary_hashtable[hv & hashmask(hashpower)] = it;
    }

    hash_items++;//元素数目+1

    //还没开始扩容，且表中元素个数已经超过Hash表容量的1.5倍
    if (! expanding && hash_items > (hashsize(hashpower) * 3) / 2) {
        //唤醒扩容线程
        assoc_start_expand();
    }

    MEMCACHED_ASSOC_INSERT(ITEM_key(it), it->nkey, hash_items);
    return 1;
}

//唤醒扩容线程
static void assoc_start_expand(void) {
    if (started_expanding)
        return;
    started_expanding = true;
    //唤醒信号量
    pthread_cond_signal(&maintenance_cond);
}

//启动扩容线程，扩容线程在main函数中会启动，启动运行一遍之后会阻塞在条件变量maintenance_cond上面，插入元素超过规定，唤醒条件变量
static void *assoc_maintenance_thread(void *arg) {
    //do_run_maintenance_thread的值为1，即该线程持续运行
    while (do_run_maintenance_thread) {
        int ii = 0;

        item_lock_global();//加Hash表的全局锁
        mutex_lock(&cache_lock);//加cache_lock锁

        //执行扩容时，每次按hash_bulk_move个桶来扩容
        for (ii = 0; ii < hash_bulk_move && expanding; ++ii) {
            item *it, *next;
            int bucket;

            //老表每次移动一个桶中的一个元素
            for (it = old_hashtable[expand_bucket]; NULL != it; it = next) {
                //要移动的下一个元素
                next = it->h_next;

                //按新的Hash规则进行定位
                bucket = hash(ITEM_key(it), it->nkey, 0) & hashmask(hashpower);
                it->h_next = primary_hashtable[bucket];//挂载到新的Hash表中
                primary_hashtable[bucket] = it;
            }

            //旧表中的这个Hash桶已经按新规则完成了扩容
            old_hashtable[expand_bucket] = NULL;
            //老表中的桶计数+1
            expand_bucket++;

            //hash表扩容结束,expand_bucket从0开始,一直递增
            if (expand_bucket == hashsize(hashpower - 1)) {
                //修改扩容标志
                expanding = false;
                //释放老的表结构
                free(old_hashtable);
                //更新一些统计信息
                STATS_LOCK();
                stats.hash_bytes -= hashsize(hashpower - 1) * sizeof(void *);
                stats.hash_is_expanding = 0;
                STATS_UNLOCK();
                if (settings.verbose > 1)
                    fprintf(stderr, "Hash table expansion done\n");
            }
        }

        mutex_unlock(&cache_lock);//释放cache_lock锁
        item_unlock_global();//释放Hash表的全局锁

        //完成扩容
        if (!expanding) {
            //修改Hash表的锁类型，此时锁类型更新为分段锁，默认是分段锁，在进行扩容时，改为全局锁
            switch_item_lock_type(ITEM_LOCK_GRANULAR);
            //释放用于扩容的锁
            slabs_rebalancer_resume();
            /* We are done expanding.. just wait for next invocation */
            mutex_lock(&cache_lock);
            //加cache_lock锁，保护条件变量
            started_expanding = false;
            //修改扩容标识
            pthread_cond_wait(&maintenance_cond, &cache_lock);
            //阻塞扩容线程
            mutex_unlock(&cache_lock);
            slabs_rebalancer_pause();
            //加用于扩容的锁
            switch_item_lock_type(ITEM_LOCK_GLOBAL);
            //修改锁类型为全局锁
            mutex_lock(&cache_lock);
            //临时用来实现临界区
            assoc_expand();//执行扩容
            mutex_unlock(&cache_lock);
        }
    }
    return NULL;
}

//按2倍容量扩容Hash表
static void assoc_expand(void) {
    //old_hashtable指向主Hash表
    old_hashtable = primary_hashtable;

    //申请新的空间
    primary_hashtable = calloc(hashsize(hashpower + 1), sizeof(void *));
    //空间申请成功
    if (primary_hashtable) {
        if (settings.verbose > 1)
            fprintf(stderr, "Hash table expansion starting\n");

        hashpower++;
        //hash等级+1
        expanding = true;
        //扩容标识打开
        expand_bucket = 0;
        STATS_LOCK();
        //更新全局统计信息
        stats.hash_power_level = hashpower;
        stats.hash_bytes += hashsize(hashpower) * sizeof(void *);
        stats.hash_is_expanding = 1;
        STATS_UNLOCK();
    } else {
        primary_hashtable = old_hashtable;
    }
}
```
