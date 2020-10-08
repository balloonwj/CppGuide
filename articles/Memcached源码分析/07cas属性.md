# Memcached源码阅读七 cas属性

cas即`compare and set`或者`compare and swap`,是实现**乐观锁**的一种技术，乐观锁是相对悲观锁来说的，所谓**悲观锁**是在数据处理过程中，完全锁定，这种能完全保证数据的一致性，但在多线程情况下，**并发性能差**，通常是使用各种锁技术实现；而乐观锁是通过版本号机制来实现数据一致性，过程中会使用CPU提供的原子操作指令，乐观锁能提高系统的并发性能，Memcached使用cas是保证数据的一致性，不是严格为了实现锁。

`Memcached`是多客户端应用，在多个客户端修改同一个数据时，会出现相互覆盖的情况，在这种情况下，使用cas版本号验证，可以有效的保证数据的一致性，Memcached默认是打开cas属性的，每次存储数据时，都会生成其cas值并和item一起存储，后续的get操作会返回系统生成的cas值，在执行set等操作时，需要将cas值传入，下面我们看看Memcached内部是如何实现cas的，关于如何使用Mecached的CAS协议，请参考文章：Memcached的CAS协议（链接：http://langyu.iteye.com/blog/680052）。

```
//为新的item生成cas值
uint64_t get_cas_id(void)
{
    static uint64_t cas_id = 0;
    return ++cas_id;
}

//这段代码是store_item的代码片段，这里是执行cas存储时执行的判断逻辑，
else if (ITEM_get_cas(it) == ITEM_get_cas(old_it))//cas值一致
{
    pthread_mutex_lock(&c->thread->stats.mutex);
    c->thread->stats.slab_stats[old_it->slabs_clsid].cas_hits++;
    pthread_mutex_unlock(&c->thread->stats.mutex);

    item_replace(old_it, it, hv);//执行存储逻辑
    stored = STORED;
}
//cas值不一致，不进行实际的存储
else 
{
    pthread_mutex_lock(&c->thread->stats.mutex);
    c->thread->stats.slab_stats[old_it->slabs_clsid].cas_badval++; 
    //更新统计信息
    pthread_mutex_unlock(&c->thread->stats.mutex);

    if (settings.verbose > 1)
    {
        //打印错误日志
        fprintf(stderr, "CAS:  failure: expected %llu, got %llu\n",
                (unsigned long long) ITEM_get_cas(old_it),
                (unsigned long long) ITEM_get_cas(it));
    }
    stored = EXISTS;
}
```
