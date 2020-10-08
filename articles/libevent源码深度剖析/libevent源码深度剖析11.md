# libevent源码深度剖析11

**时间管理**

为了支持定时器，libevent必须和系统时间打交道，这一部分的内容也比较简单，主要涉及到时间的加减辅助函数、时间缓存、时间校正和定时器堆的时间值调整等。下面就结合源代码来分析一下。

### 1. 初始化检测

libevent在初始化时会检测系统时间的类型，通过调用函数**d****etect_monotonic()**完成，它通过调用**clock_gettime()**来检测系统是否支持monotonic时钟类型：

```
static void detect_monotonic(void){
#if defined(HAVE_CLOCK_GETTIME) && defined(CLOCK_MONOTONIC)
    struct timespec    ts;
    if (clock_gettime(CLOCK_MONOTONIC, &ts) == 0)
        use_monotonic = 1; // 系统支持monotonic时间
#endif
}
```

Monotonic时间指示的是系统从boot后到现在所经过的时间，如果系统支持Monotonic时间就将全局变量**use_monotonic**设置为1，设置use_monotonic到底有什么用，这个在后面说到时间校正时就能看出来了。

### 2. 时间缓存

结构体event_base中的**tv_cache**，用来记录时间缓存。这个还要从函数**gettime()**说起，先来看看该函数的代码：

```
static int gettime(struct event_base *base, struct timeval *tp){
    // 如果tv_cache时间缓存已设置，就直接使用
    if (base->tv_cache.tv_sec) {
        *tp = base->tv_cache;
        return (0);
    }
    // 如果支持monotonic，就用clock_gettime获取monotonic时间
#if defined(HAVE_CLOCK_GETTIME) && defined(CLOCK_MONOTONIC)
    if (use_monotonic) {
        struct timespec    ts;
        if (clock_gettime(CLOCK_MONOTONIC, &ts) == -1)
            return (-1);
        tp->tv_sec = ts.tv_sec;
        tp->tv_usec = ts.tv_nsec / 1000;
        return (0);
    }
#endif
    // 否则只能取得系统当前时间
    return (evutil_gettimeofday(tp, NULL));
}
```

如果tv_cache已经设置，那么就直接使用缓存的时间；否则需要再次执行系统调用获取系统时间。
函数**evutil_gettimeofday()**用来获取当前系统时间，在Linux下其实就是系统调用gettimeofday()；Windows没有提供函数gettimeofday，而是通过调用**_ftime()**来完成的。
在每次系统事件循环中，时间缓存tv_cache将会被相应的清空和设置，再次来看看下面event_base_loop的主要代码逻辑：

```
int event_base_loop(struct event_base *base, int flags){
    // 清空时间缓存
    base->tv_cache.tv_sec = 0;
    while(!done){
        timeout_correct(base, &tv); // 时间校正
        // 更新event_tv到tv_cache指示的时间或者当前时间（第一次）
         // event_tv <--- tv_cache
        gettime(base, &base->event_tv);
        // 清空时间缓存-- 时间点1
        base->tv_cache.tv_sec = 0;
        // 等待I/O事件就绪
        res = evsel->dispatch(base, evbase, tv_p);
        // 缓存tv_cache存储了当前时间的值-- 时间点2
         // tv_cache <--- now
        gettime(base, &base->tv_cache);
        // .. 处理就绪事件
    }
    // 退出时也要清空时间缓存
    base->tv_cache.tv_sec = 0;
    return (0);
}
```

时间event_tv指示了**dispatch()**上次返回，也就是I/O事件就绪时的时间，第一次进入循环时，由于tv_cache被清空，因此**gettime()**执行系统调用获取当前系统时间；而后将会更新为tv_cache指示的时间。
时间tv_cache在**dispatch()**返回后被设置为当前系统时间，因此它缓存了本次I/O事件就绪时的时间**（event_tv）**。
从代码逻辑里可以看出event_tv取得的是tv_cache上一次的值，因此event_tv应该小于tv_cache的值。
设置时间缓存的优点是不必每次获取时间都执行系统调用，这是个相对费时的操作；在上面标注的时间点2到时间点1的这段时间（处理就绪事件时），调用gettime()取得的都是tv_cache缓存的时间。

### 3. 时间校正

如果系统支持**monotonic**时间，该时间是系统从boot后到现在所经过的时间，因此不需要执行校正。
根据前面的代码逻辑，如果系统不支持monotonic时间，用户可能会手动的调整时间，如果时间被向前调整了（MS前面第7部分讲成了向后调整，要改正），比如从5点调整到了3点，那么在时间点2取得的值可能会小于上次的时间，这就需要调整了，下面来看看校正的具体代码，由函数**timeout_correct()**完成：

```
static void timeout_correct(struct event_base *base, struct timeval *tv){
    struct event **pev;
    unsigned int size;
    struct timeval off;
    if (use_monotonic) // monotonic时间就直接返回，无需调整
        return;
    gettime(base, tv); // tv <---tv_cache
    // 根据前面的分析可以知道event_tv应该小于tv_cache
    // 如果tv < event_tv表明用户向前调整时间了，需要校正时间
    if (evutil_timercmp(tv, &base->event_tv, >=)) {
        base->event_tv = *tv;
        return;
    }
    // 计算时间差值
    evutil_timersub(&base->event_tv, tv, &off);
    // 调整定时事件小根堆
    pev = base->timeheap.p;
    size = base->timeheap.n;
    for (; size-- > 0; ++pev) {
        struct timeval *ev_tv = &(**pev).ev_timeout;
        evutil_timersub(ev_tv, &off, ev_tv);
    }
    base->event_tv = *tv; // 更新event_tv为tv_cache
}
```

在调整小根堆时，因为所有定时事件的时间值都会被减去相同的值，因此虽然堆中元素的时间键值改变了，但是相对关系并没有改变，不会改变堆的整体结构。因此只需要遍历堆中的所有元素，将每个元素的时间键值减去相同的值即可完成调整，不需要重新调整堆的结构。
当然调整完后，要将event_tv值重新设置为tv_cache值了。

### 4. 小节

主要分析了一下libevent对系统时间的处理，时间缓存、时间校正和定时堆的时间值调整等，逻辑还是很简单的，时间的加减、设置等辅助函数则非常简单，主要在头文件evutil.h中，就不再多说了。
