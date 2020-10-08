# Memcached源码阅读十三 do_item_alloc操作

前面我们分析了`Memcached`的set操作，其set操作在经过所有的数据有效性检查之后，如果需要存储item，则会执行item的实际存储操作，我们下面分析下其过程。

```
//执行item的存储操作,该操作会将item挂载到LRU表和slabcalss中
item *do_item_alloc(char *key, const size_t nkey, const int flags,
                    const rel_time_t exptime, const int nbytes,
                    const uint32_t cur_hv) {
    uint8_t nsuffix;
    item *it = NULL;
    char suffix[40];
    //计算item的总大小(空间)
    size_t ntotal = item_make_header(nkey + 1, flags, nbytes, suffix, &nsuffix);
    //如果使用了cas
    if (settings.use_cas) {
        //增加cas的空间
        ntotal += sizeof(uint64_t);
    }

    unsigned int id = slabs_clsid(ntotal);
    //那大小选择合适的slab
    if (id == 0)
        return 0;

    //执行LRU锁
    mutex_lock(&cache_lock);
    //存储时，会尝试从LRU中选择合适的空间的空间
    int tries = 5;
    //如果LRU中尝试5次还没合适的空间，则执行申请空间的操作
    int tried_alloc = 0;
    item *search;
    void *hold_lock = NULL;
    //初始化时选择的过期时间
    rel_time_t oldest_live = settings.oldest_live;

    search = tails[id];//第id个LRU表的尾部

    for (; tries > 0 && search != NULL; tries--, search=search->prev) {
        uint32_t hv = hash(ITEM_key(search), search->nkey, 0);//获取分段锁

        //尝试执行锁操作，这里执行的乐观锁
        if (hv != cur_hv && (hold_lock = item_trylock(hv)) == NULL)
            continue;
        //判断item是否被锁住，item的引用次数其实充当的也是一种锁
        if (refcount_incr(&search->refcount) != 2) {
            refcount_decr(&search->refcount);//更新it的引用次数
            //如果it的添加时间比当前时间小于3*3600
            if (search->time + TAIL_REPAIR_TIME < current_time) {
                itemstats[id].tailrepairs++;//更新统计信息
                search->refcount = 1;
                do_item_unlink_nolock(search, hv);//执行分段解锁操作
            }

            if (hold_lock)
                item_trylock_unlock(hold_lock);//执行分段解锁操作
            continue;
        }

        //过期时间判断
        if ((search->exptime != 0 && search->exptime < current_time)
            || (search->time <= oldest_live && oldest_live <= current_time)) { 
            //过期时间的判断
            itemstats[id].reclaimed++;
            if ((search->it_flags & ITEM_FETCHED) == 0) {
                itemstats[id].expired_unfetched++;//更新统计信息
            }

            it = search;
            //slabclass申请合适的空间
            slabs_adjust_mem_requested(it->slabs_clsid, ITEM_ntotal(it), ntotal);
            //执行的Hash表的分段解锁操作
            do_item_unlink_nolock(it, hv);
            it->slabs_clsid = 0;
        } else if ((it = slabs_alloc(ntotal, id)) == NULL) {
            //申请失败一次
            tried_alloc = 1;
            //关闭了LRU的
            if (settings.evict_to_free == 0) {
                itemstats[id].outofmemory++;
                //统计信息更新
            } else {
                itemstats[id].evicted++;
                //更新统计信息
                itemstats[id].evicted_time = current_time - search->time;

                if (search->exptime != 0)
                    itemstats[id].evicted_nonzero++;
                if ((search->it_flags & ITEM_FETCHED) == 0) {
                    itemstats[id].evicted_unfetched++;
                }

                it = search;
                slabs_adjust_mem_requested(it->slabs_clsid, ITEM_ntotal(it), ntotal);//选择合适的slabclass空间
                do_item_unlink_nolock(it, hv);//执行it的分段解锁操作

                it->slabs_clsid = 0;

                if (settings.slab_automove == 2)//如果打开了slab调整
                    slabs_reassign(-1, id);//唤醒调整线程
            }
        }

         //更新引用次数
        refcount_decr(&search->refcount);
        if (hold_lock)
            item_trylock_unlock(hold_lock);//解分段锁
        break;
    }

    //5次循环查找，未找到合适的空间
    if (!tried_alloc && (tries == 0 || search == NULL))
        //则从内存池申请新的空间
        it = slabs_alloc(ntotal, id);

    //内存池申请失败
    if (it == NULL) {
        itemstats[id].outofmemory++;//更新统计信息
        mutex_unlock(&cache_lock);//释放LRU锁
        return NULL;
    }

    assert(it->slabs_clsid == 0);
    assert(it != heads[id]);

    it->refcount = 1;//更新it的引用次数
    mutex_unlock(&cache_lock);
    it->next = it->prev = it->h_next = 0;//执行初始化操作
    it->slabs_clsid = id;//it所属的slabclass为第id个

    DEBUG_REFCNT(it, '*');
    it->it_flags = settings.use_cas ? ITEM_CAS : 0;
    it->nkey = nkey;//it的key
    it->nbytes = nbytes;//it的缓冲区的数据
    memcpy(ITEM_key(it), key, nkey);//it的数据信息
    it->exptime = exptime;//it的过期时间
    memcpy(ITEM_suffix(it), suffix, (size_t)nsuffix);//it的前缀信息
    it->nsuffix = nsuffix;//it的一些前缀信息
    return it;
}

//计算item的大小
static size_t item_make_header(const uint8_t nkey, const int flags, const int nbytes,
                     char *suffix, uint8_t *nsuffix) {
    //suffix限定了40个字节
    *nsuffix = (uint8_t) snprintf(suffix, 40, " %d %d\r\n", flags, nbytes - 2);
    //返回item的长度
    return sizeof(item) + nkey + *nsuffix + nbytes;
}

//选择合适的slabclass
unsigned int slabs_clsid(const size_t size) {
    int res = POWER_SMALLEST;

    if (size == 0)
        return 0;

    //按slabclass的size的选择
    while (size > slabclass[res].size)
        //如果大于最大的slab的，则直接返回错误，按默认的，大于1M的申请空间失败   
        if (res++ == power_largest)
            return 0;
    return res;
}

//从内存池申请合适的空间
void slabs_adjust_mem_requested(unsigned int id, size_t old, size_t ntotal)
{
    //slabclass加锁，保持同步
    pthread_mutex_lock(&slabs_lock);
    slabclass_t *p;

    //判断数据合法性
    if (id < POWER_SMALLEST || id > power_largest) {
        fprintf(stderr, "Internal error! Invalid slab class\n");
        abort();
    }

    p = &slabclass[id];
    //调整request信息，request表示的是old所在的slab申请空间大小
    p->requested = p->requested - old + ntotal;
    pthread_mutex_unlock(&slabs_lock);
}
```
