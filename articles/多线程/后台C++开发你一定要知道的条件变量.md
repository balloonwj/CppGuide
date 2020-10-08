## 后台C++开发你一定要知道的条件变量

今天因为工作需要，需要帮同事用C语言（不是C++）写一个生产者消费者的任务队列工具库，考虑到不能使用任何第三库和C++的任何特性，所以我将任务队列做成一个链表，生产者在队列尾部加入任务，消费者在队列头部取出任务。很快就写好了，代码如下：

```
/**
 *  线程池工具, ctrip_thread_pool.h
 *  zhangyl 2018.03.23
 */

#ifndef __CTRIP_THREAD_POOL_H__
#define __CTRIP_THREAD_POOL_H__

#include <pthread.h>

#ifndef NULL
#define NULL 0
#endif

#define PUBLIC 

PUBLIC struct ctrip_task
{
    struct ctrip_task*   pNext;
    int                  value;
};

struct ctrip_thread_info
{
    //线程退出标志
    int                    thread_running;
    int                    thread_num;
    int                    tasknum;
    struct ctrip_task*     tasks;
    pthread_t*             threadid;
    pthread_mutex_t        mutex;
    pthread_cond_t         cond;
};

/* 初始化线程池线程数目
 * @param thread_num 线程数目, 默认为8个
 */
PUBLIC void  ctrip_init_thread_pool(int thread_num);

/* 销毁线程池
 */
PUBLIC void  ctrip_destroy_thread_pool();

/**向任务池中增加一个任务
 * @param t 需要增加的任务
 */
PUBLIC void ctrip_thread_pool_add_task(struct ctrip_task* t);

/**从任务池中取出一个任务
 * @return 返回得到的任务
 */
struct ctrip_task* ctrip_thread_pool_retrieve_task();

/**执行任务池中的任务
 * @param t 需要执行的任务
 */
PUBLIC void ctrip_thread_pool_do_task(struct ctrip_task* t);

/**线程函数
 * @param thread_param 线程参数
 */
void* ctrip_thread_routine(void* thread_param);

#endif //!__CTRIP_THREAD_POOL_H__
/**
 *  线程池工具, ctrip_thread_pool.c
 *  zhangyl 2018.03.23
 */

#include "ctrip_thread_pool.h"
#include <stdio.h>
#include <stdlib.h>

struct ctrip_thread_info g_threadinfo;
int thread_running = 0;

void ctrip_init_thread_pool(int thread_num)
{
    if (thread_num <= 0)
        thread_num = 5;

    pthread_mutex_init(&g_threadinfo.mutex, NULL);
    pthread_cond_init(&g_threadinfo.cond, NULL);

    g_threadinfo.thread_num = thread_num;
    g_threadinfo.thread_running = 1;
    g_threadinfo.tasknum = 0;
    g_threadinfo.tasks = NULL;
    thread_running = 1;

    g_threadinfo.threadid = (pthread_t*)malloc(sizeof(pthread_t) * thread_num);

    int i;
    for (i = 0; i < thread_num; ++i)
    {
        pthread_create(&g_threadinfo.threadid[i], NULL, ctrip_thread_routine, NULL);
    }
}

void ctrip_destroy_thread_pool()
{
    g_threadinfo.thread_running = 0;
    thread_running = 0;
    pthread_cond_broadcast(&g_threadinfo.cond);

    int i;
    for (i = 0; i < g_threadinfo.thread_num; ++i)
    {
        pthread_join(g_threadinfo.threadid[i], NULL);
    }

    free(g_threadinfo.threadid);

    pthread_mutex_destroy(&g_threadinfo.mutex);
    pthread_cond_destroy(&g_threadinfo.cond);
}

void ctrip_thread_pool_add_task(struct ctrip_task* t)
{
    if (t == NULL)
        return;

    pthread_mutex_lock(&g_threadinfo.mutex);
    
    struct ctrip_task* head = g_threadinfo.tasks;
    if (head == NULL)
        g_threadinfo.tasks = t;
    else
    {
        while (head->pNext != NULL)
        {
            head = head->pNext;
        }

        head->pNext = t;
    }
    
    ++g_threadinfo.tasknum;
    //当有变化后，使用signal通知wait函数
    pthread_cond_signal(&g_threadinfo.cond);
    pthread_mutex_unlock(&g_threadinfo.mutex);
}


struct ctrip_task* ctrip_thread_pool_retrieve_task()
{
    struct ctrip_task* head = g_threadinfo.tasks;
    if (head != NULL)
    {
        g_threadinfo.tasks = head->pNext;
        --g_threadinfo.tasknum;
        printf("retrieve a task, task value is [%d]\n", head->value);
        return head;
    }

    printf("no task\n");

    return NULL;
}

void* ctrip_thread_routine(void* thread_param)
{
    printf("thread NO.%d start.\n", (int)pthread_self());

    while (thread_running/*g_threadinfo.thread_running*/)
    {
        struct ctrip_task* current = NULL;
        
        pthread_mutex_lock(&g_threadinfo.mutex);

        while (g_threadinfo.tasknum <= 0)
        {
            //如果获得了互斥锁，但是条件不合适的话，wait会释放锁，不往下执行。
            //当变化后，条件合适，将直接获得锁。
            pthread_cond_wait(&g_threadinfo.cond, &g_threadinfo.mutex);

            if (!g_threadinfo.thread_running)
                break;

            current = ctrip_thread_pool_retrieve_task();

            if (current != NULL)
                break;
        }// end inner-while-loop

        pthread_mutex_unlock(&g_threadinfo.mutex);

        ctrip_thread_pool_do_task(current);
    }// end outer-while-loop

    printf("thread NO.%d exit.\n", (int)pthread_self());
}


void ctrip_thread_pool_do_task(struct ctrip_task* t)
{
    if (t == NULL)
        return;


    //TODO: do your work here
    printf("task value is [%d]\n", t->value);
    //TODO：如果t需要释放，记得在这里释放
}
```

测试代码如下：

```
// ctrip_thread_pool.cpp : Defines the entry point for the console application.
//

//#include "stdafx.h"
#include "ctrip_thread_pool.h"
#include <stdlib.h>
#include <unistd.h>

int main(int argc, char* argv[])
{
    ctrip_init_thread_pool(5);
    struct ctrip_task* task = NULL;
    int i;
    for (i = 0; i < 100; ++i)
    {
        task = (struct ctrip_task*)malloc(sizeof(struct ctrip_task));
        task->value = i + 1;
        task->pNext = NULL;
        printf("add task, task value [%d]\n", task->value);
        ctrip_thread_pool_add_task(task);
    }

    sleep(10);

    ctrip_destroy_thread_pool();
    
    return 0;
}
```



代码很快就写好了，但是每次程序只能执行前几个加到任务池子里面的任务，导致池子有不少任务积累在池子里面。甚是奇怪，我也看了半天才看出结果。聪明的你，能看出上述代码为啥只能执行加到池子里面的前几个任务？先不要看答案，自己想一会儿。



linux条件变量是做后台开发必须熟练掌握的基础知识，而条件变量使用存在以下几个非常让人迷惑的地方，讲解如下

第一、必须要结合一个互斥体一起使用。使用流程如下：

```
pthread_mutex_lock(&g_threadinfo.mutex)           
pthread_cond_wait(&g_threadinfo.cond, &g_threadinfo.mutex);
pthread_mutex_unlock(&g_threadinfo.mutex);
```

上面的代码，我们分为一二三步，当条件不满足是pthread_cond_wait会挂起线程，但是不知道你有没有注意到，如果在第二步挂起线程的话，第一步的mutex已经被上锁，谁来解锁？mutex的使用原则是谁上锁谁解锁，所以不可能在其他线程来给这个mutex解锁，但是这个线程已经挂起了，这就死锁了。所以pthread_cond_wait在挂起之前，额外做的一个事情就是给绑定的mutex解锁。反过来，如果条件满足，pthread_cond_wait不挂起线程，pthread_cond_wait将什么也不做，这样就接着走pthread_mutex_unlock解锁的流程。而在这个加锁和解锁之间的代码就是我们操作受保护资源的地方。

第二，不知道你有没有注意到pthread_cond_wait是放在一个while循环里面的：

```
        pthread_mutex_lock(&g_threadinfo.mutex);

        while (g_threadinfo.tasknum <= 0)
        {
            //如果获得了互斥锁，但是条件不合适的话，wait会释放锁，不往下执行。
            //当变化后，条件合适，将直接获得锁。
            pthread_cond_wait(&g_threadinfo.cond, &g_threadinfo.mutex);

            if (!g_threadinfo.thread_running)
                break;

            current = ctrip_thread_pool_retrieve_task();

            if (current != NULL)
                break;
        }// end inner-while-loop

        pthread_mutex_unlock(&g_threadinfo.mutex);
```

注意，我说的是内层的while循环，不是外层的。pthread_cond_wait一定要放在一个while循环里面吗？一定要的。这里有一个非常重要的关于条件变量的基础知识，叫条件变量的虚假唤醒（spurious wakeup），那啥叫条件变量的虚假唤醒呢？假设pthread_cond_wait不放在这个while循环里面，正常情况下，pthread_cond_wait因为条件不满足，挂起线程。然后，外部条件满足以后，调用pthread_cond_signal或pthread_cond_broadcast来唤醒挂起的线程。这没啥问题。但是条件变量可能在某些情况下也被唤醒，这个时候pthread_cond_wait处继续往下执行，但是这个时候，条件并不满足（比如任务队列中仍然为空）。这种唤醒我们叫“虚假唤醒”。为了避免虚假唤醒时，做无意义的动作，我们将pthread_cond_wait放到while循环条件中，这样即使被虚假唤醒了，由于while条件（比如任务队列是否为空，资源数量是否大于0）仍然为true，导致线程进行继续挂起。有人说条件变量是最不可能用错的线程之间同步技术，我却觉得这是最容易使用错误的线程之间同步技术。



上述代码存在的问题是，只考虑了任务队列开始为空，生产者后来添加了任务，条件变量被唤醒，然后消费者取任务执行的逻辑。假如一开始池中就有任务呢？这个原因导致，只有开始的几个添加到任务队列中任务被执行。因为一旦任务队列不为空。内层while循环条件将不再满足，导致消费者线程不再从任务队列中取任务消费。正确的代码如下：

```
/**
 *  线程池工具, ctrip_thread_pool.c（修正后的代码）
 *  zhangyl 2018.03.23
 */

#include "ctrip_thread_pool.h"
#include <stdio.h>
#include <stdlib.h>

struct ctrip_thread_info g_threadinfo;

void ctrip_init_thread_pool(int thread_num)
{
    if (thread_num <= 0)
        thread_num = 5;

    pthread_mutex_init(&g_threadinfo.mutex, NULL);
    pthread_cond_init(&g_threadinfo.cond, NULL);

    g_threadinfo.thread_num = thread_num;
    g_threadinfo.thread_running = 1;
    g_threadinfo.tasknum = 0;
    g_threadinfo.tasks = NULL;

    g_threadinfo.threadid = (pthread_t*)malloc(sizeof(pthread_t) * thread_num);

    int i;
    for (i = 0; i < thread_num; ++i)
    {
        pthread_create(&g_threadinfo.threadid[i], NULL, ctrip_thread_routine, NULL);
    }
}

void ctrip_destroy_thread_pool()
{
    g_threadinfo.thread_running = 0;
    pthread_cond_broadcast(&g_threadinfo.cond);

    int i;
    for (i = 0; i < g_threadinfo.thread_num; ++i)
    {
        pthread_join(g_threadinfo.threadid[i], NULL);
    }

    free(g_threadinfo.threadid);

    pthread_mutex_destroy(&g_threadinfo.mutex);
    pthread_cond_destroy(&g_threadinfo.cond);
}

void ctrip_thread_pool_add_task(struct ctrip_task* t)
{
    if (t == NULL)
        return;

    pthread_mutex_lock(&g_threadinfo.mutex);
    
    struct ctrip_task* head = g_threadinfo.tasks;
    if (head == NULL)
        g_threadinfo.tasks = t;
    else
    {
        while (head->pNext != NULL)
        {
            head = head->pNext;
        }

        head->pNext = t;
    }
    
    ++g_threadinfo.tasknum;
    //当有变化后，使用signal通知wait函数
    pthread_cond_signal(&g_threadinfo.cond);
    pthread_mutex_unlock(&g_threadinfo.mutex);
}


struct ctrip_task* ctrip_thread_pool_retrieve_task()
{
    struct ctrip_task* head = g_threadinfo.tasks;
    if (head != NULL)
    {
        g_threadinfo.tasks = head->pNext;
        --g_threadinfo.tasknum;
        printf("retrieve a task, task value is [%d]\n", head->value);
        return head;
    }

    printf("no task\n");

    return NULL;
}

void* ctrip_thread_routine(void* thread_param)
{
    printf("thread NO.%d start.\n", (int)pthread_self());

    while (g_threadinfo.thread_running)
    {
        struct ctrip_task* current = NULL;
        
        pthread_mutex_lock(&g_threadinfo.mutex);

        while (g_threadinfo.tasknum <= 0)
        {
            //如果获得了互斥锁，但是条件不合适的话，wait会释放锁，不往下执行。
            //当变化后，条件合适，将直接获得锁。
            pthread_cond_wait(&g_threadinfo.cond, &g_threadinfo.mutex);

            if (!g_threadinfo.thread_running)
                break;

        }// end inner-while-loop

        current = ctrip_thread_pool_retrieve_task();
        pthread_mutex_unlock(&g_threadinfo.mutex);

        ctrip_thread_pool_do_task(current);
    }// end outer-while-loop

    printf("thread NO.%d exit.\n", (int)pthread_self());
}


void ctrip_thread_pool_do_task(struct ctrip_task* t)
{
    if (t == NULL)
        return;


    //TODO: do your work here
    printf("task value is [%d]\n", t->value);
    //TODO：如果t需要释放，记得在这里释放
}
```

ok，不知道你有没有看明白呀？