# Memcached源码阅读十二 set操作

之前分析了`Memcached`的get操作，下面分析**set操作**的流程。

```
//存储item
enum store_item_type store_item(item *item, int comm, conn* c) {
    enum store_item_type ret;
    uint32_t hv;

    hv = hash(ITEM_key(item), item->nkey, 0);//获取Hash表的分段锁
    item_lock(hv);//执行数据同步
    ret = do_store_item(item, comm, c, hv);//存储item
    item_unlock(hv);
    return ret;
}

//存储item
enum store_item_type do_store_item(item *it, int comm, conn *c,const uint32_t hv)
{
    char *key = ITEM_key(it);//读取item对应的key
    item *old_it = do_item_get(key, it->nkey, hv);
    //读取相应的item,如果没有相关的数据,old_it为NULL
    enum store_item_type stored = NOT_STORED;//item状态标记

    item *new_it = NULL;
    int flags;

    //如果old_it不为NULL,且操作为add操作
    if (old_it != NULL && comm == NREAD_ADD)
    {
        do_item_update(old_it);//更新数据
    }
    //old_it为空，且操作为REPLACE，则什么都不做
    else if (!old_it
            && (comm == NREAD_REPLACE || comm == NREAD_APPEND
                    || comm == NREAD_PREPEND)) 
    {
          //memcached的Replace操作是替换已有的数据，如果没有相关数据，则不做任何操作
    }
    //以cas方式读取
    else if (comm == NREAD_CAS)
    {
        if (old_it == NULL) //为空
        {
            // LRU expired
            stored = NOT_FOUND;//修改状态
            pthread_mutex_lock(&c->thread->stats.mutex);//更新Worker线程统计数据
            c->thread->stats.cas_misses++;
            pthread_mutex_unlock(&c->thread->stats.mutex);
        }
        //old_it不为NULL，且cas属性一致
        else if (ITEM_get_cas(it) == ITEM_get_cas(old_it))
        {
            pthread_mutex_lock(&c->thread->stats.mutex);
            c->thread->stats.slab_stats[old_it->slabs_clsid].cas_hits++;
            //更新Worker线程统计信息
            pthread_mutex_unlock(&c->thread->stats.mutex);

            item_replace(old_it, it, hv);
            //执行item的替换操作，用新的item替换老的item
            stored = STORED;//修改状态值
        }
        else
        //old_it不为NULL,且cas属性不一致
        {
            pthread_mutex_lock(&c->thread->stats.mutex);
            c->thread->stats.slab_stats[old_it->slabs_clsid].cas_badval++;
            //更新Worker线程统计信息
            pthread_mutex_unlock(&c->thread->stats.mutex);

            if (settings.verbose > 1)
            {
                fprintf(stderr, "CAS:  failure: expected %llu, got %llu\n",
                        (unsigned long long) ITEM_get_cas(old_it),
                        (unsigned long long) ITEM_get_cas(it));
            }
            stored = EXISTS;
            //修改状态值，修改状态值为已经存在，且不存储最新的数据
        }
    }
    else //执行其他操作的写
    {
        //以追加的方式执行写
        if (comm == NREAD_APPEND || comm == NREAD_PREPEND)
        {
            //验证cas有效性
            if (ITEM_get_cas(it) != 0)
            {
                //cas验证不通过
                if (ITEM_get_cas(it) != ITEM_get_cas(old_it))
                {
                    stored = EXISTS;//修改状态值为已存在
                }
            }

             //状态值为没有存储,也就是cas验证通过，则执行写操作
            if (stored == NOT_STORED)
            {
                flags = (int) strtol(ITEM_suffix(old_it), (char **) NULL, 10);
                //申请新的空间
                new_it = do_item_alloc(key, it->nkey, flags, old_it->exptime,it->nbytes + old_it->nbytes - 2 , hv);

                if (new_it == NULL)
                {
                    //空间不足
                    if (old_it != NULL)
                        do_item_remove(old_it);//删除老的item

                    return NOT_STORED;
                }

                if (comm == NREAD_APPEND)//追加方式
                {   
                    memcpy(ITEM_data(new_it), ITEM_data(old_it),old_it->nbytes);//老数据拷贝到新数据中
                    memcpy(ITEM_data(new_it) + old_it->nbytes - 2,ITEM_data(it), it->nbytes);//同时拷贝最近缓冲区已有的数据
                }
                else
                {
                    //这里和具体协议相关
                    memcpy(ITEM_data(new_it), ITEM_data(it), it->nbytes);//拷贝it的数据到new_it中
                    memcpy(ITEM_data(new_it) + it->nbytes - 2 ,ITEM_data(old_it), old_it->nbytes);//同时拷贝最近缓冲区已有的数据
                }

                it = new_it;
            }
        }

        if (stored == NOT_STORED)
        {
            if (old_it != NULL)//如果old_it不为空
                item_replace(old_it, it, hv);//替换老的值
            else
                do_item_link(it, hv);//重新存储数据

            c->cas = ITEM_get_cas(it);//获取cas值

            stored = STORED;
        }
    }

    if (old_it != NULL)
        do_item_remove(old_it);//释放空间
    if (new_it != NULL)
        do_item_remove(new_it);//释放空间

    if (stored == STORED)//如果已经存储了
    {
        c->cas = ITEM_get_cas(it);//获取cas属性
    }

    return stored;
}

//更新item，这个只更新时间
void do_item_update(item *it) {
    MEMCACHED_ITEM_UPDATE(ITEM_key(it), it->nkey, it->nbytes);
    if (it->time < current_time - ITEM_UPDATE_INTERVAL) {
        //更新有时间限制
        assert((it->it_flags & ITEM_SLABBED) == 0);

        mutex_lock(&cache_lock);//保持同步
        //更新LRU队列的Item
        if ((it->it_flags & ITEM_LINKED) != 0) {
            item_unlink_q(it);//断开连接
            it->time = current_time;//更新item的时间
            item_link_q(it);//重新添加
        }
        mutex_unlock(&cache_lock);
    }
} 

//用新的item替换老的item
int do_item_replace(item *it, item *new_it, const uint32_t hv) {
    MEMCACHED_ITEM_REPLACE(ITEM_key(it), it->nkey, it->nbytes,
                           ITEM_key(new_it), new_it->nkey, new_it->nbytes);
    //判断it是已经分配过的，如果未分配，则断言失败
    assert((it->it_flags & ITEM_SLABBED) == 0);

    do_item_unlink(it, hv);//断开连接
    return do_item_link(new_it, hv);//重新添加
}
```

有些item的操作已经在get操作中有分析，我们此处不做分析，我们下一篇分析下`Memcached`内部如何选择合适的空间来存放item。
