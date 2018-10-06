---
title: libev设计与实现
date: 2016-06-18 10:25:38
categories: 后台开发
tags:
  - libeasy
  - libev
  - 网络编程
  - 设计模式
---

# Introduction

libev是一个高性能的事件处理框架，基于reactor模式实现。本文先讨论reactor模式，然后具体地分析libev的设计和实现。

本文组织结构如下：

- reactor模式
- libev设计和实现
- 参考文献

# Reactor

由于libev是基于reactor模式实现的事件处理框架，所以，先简单介绍下reactor模型的整体思路，方便讨论libev设计和实现时对照分析。

reactor模式底层使用了I/O多路复用模型，是单个线程中管理多个I/O流的技术。I/O多路复用模型是unix环境中的一种I/O模型，除了它之外，还有阻塞I/O，非阻塞I/O，I/O多路复用，信号驱动I/O和异步I/O。

## Unix 5种I/O模型

**阻塞I/O**

对于阻塞I/O，应用调用I/O操作后，会一直等待阻塞，等待数据准备好。此时进程无法做其他的任何事情，只能傻傻等着recvfrom返回。

![blocking io](http://oserror.com/images/io_model_blocking.png)

**非阻塞I/O**

非阻塞I/O与阻塞I/O的区别是，非阻塞I/O在等待数据的时候，进程不是阻塞的，它可以先做其他的事情，然后定期的来询问数据是否准备好。

![non-blocking io](http://oserror.com/images/io_model_nonblocking.png)

**I/O多路复用**

I/O多路复用的流程是先使用select/epoll之类的接口等待I/O事件就绪，待select/epoll返回后，数据已经准备好了，这时候，一般再调用recvfrom来读取数据。相对于阻塞式I/O，这里涉及到了select/epoll以及recvfrom两次系统调用，看起来效率应该比阻塞式I/O还要低。对于只有单个fd活跃的进程来讲确实是效率会更低，但是select/epoll可以同时等待多个fd，对于活跃fd比较多的进程，使用I/O多路复用是较好的选择。

![io multiplexing](http://oserror.com/images/io_model_multiplexing.png)

**信号驱动I/O**

信号驱动I/O的流程是建立一个SIGIO信号处理程序，当数据准备好后，再调用recvfrom函数来处理，期间，进程可以完全来处理自己的事情，不必像阻塞式I/O那样阻塞，也不必像非阻塞I/O那样轮询。

![signal io](http://oserror.com/images/io_model_signal.png)

**异步I/O**

异步I/O的流程是在调用aio_read之后直到数据拷贝完成都不需要进程等待。对于前面四种I/O模型，拷贝数据的过程进程是必须要自己处理的，而对于异步I/O来讲，这个步骤也不需要自己处理。

![async io](http://oserror.com/images/io_model_async.png)

对于目前主流的reactor网络模型中，采用的是I/O多路复用模型，其优点是能同时处理等待多个fd的数据准备过程，非常适合互联网领域服务端需要处理大规模网络链接的情况。

## Reactor模式

一个reactor模式如下图：

![reactor](http://oserror.com/images/reactor_model.png)

reactor中组件包括reactor，EventHandler，I/O multiplexing和Timer

- EventHandler是事件的接口，一般分为I/O事件、定时器事件等等
- I/O multiplexing即I/O多路复用，linux中一般采用epoll接口
- Timer是管理定时器的类，主要负责注册事件、获取超时事件列表等等，一般由网络框架开发者实现
- reactor中使用了I/O multiplexing和Timer，有EventHandler注册时，会调用相应的接口。reactor的HandleEvents中需要先调用I/O multiplexing和Timer的接口，获取已就绪好的事件，最终调用每个EventHandler的HandleEvent接口

# libev设计与实现

根据reactor模型的组件构成，本节从libev的EventHandler，I/O multiplexing，Timer以及reactor四个方面来分析其设计和实现。

## EventHandler

libev支持多种事件的处理，包括io，timer，signal等等。其全部支持的事件如下

```c++
ev_io
ev_timer
ev_periodic
ev_signal
ev_child
ev_stat
ev_idle
ev_prepare and ev_check
ev_embed
ev_fork
ev_cleanup
ev_async
```
libev使用宏定义实现了类似C++继承的组织结构，具体如下图：

![libev event handle](http://oserror.com/images/libev_event_handle.png)

如上图，只列举了libev比较常见的ev\_io,ev\_timer和ev\_signal。EV_WATCHER相当于Reactor模式中的EventHandler基类，而ev\_io，ev\_timer和ev\_signal属于EventHandler的子类，实现各自的事件。

- 所有事件中都会有一个cb函数指针，相当于EventHandler中的HandleEvent函数，用于事件发生后调用
- ev\_io和ev\_signal都有EV_WAITCHER的链表，其上可以注册一个或者多个Handle函数。
- ev\_timer拥有repeat字段，可以用来设置重复调用的定时任务。

这些事件还提供了一些操作函数，以ev\_io为例，如下

```c++
#define ev_io_set(ev,fd_,events_)            \
do { (ev)->fd = (fd_); (ev)->events = (events_) | EV__IOFDSET; } while (0)

void ev_io_start       (struct ev_loop *loop,  ev_io *w);

void ev_io_stop        (struct ev_loop *loop,  ev_io *w);
```

ev\_io还提供了初始化等功能，例如，`ev_io_init`函数初始化fd，cb等，其他的像`ev_io_start`和`ev_io_stop`函数是由libev的reactor组件提供的接口，将在reactor章节分析。

## I/O multiplexing

libev支持多种I/O multiplexing的接口，例如select，epoll，kequeue。为了方便统一管理，libev提供一套标准接口，对应的I/O multiplexing需要实现这套标准接口，以供上层调用，虽然libev不是以OO方式实现的，这里还是以UML图的方式描述，方便讨论。

![libev IO multiplexing](http://oserror.com/images/libev_io_multiplexing.png)

libev本身是通过函数指针方式实现的，即在reactor模块，使用了backend_modify和backend_poll来个函数指针，具体指向的I/O multiplexing接口跟实现有关，在linux下可能是select实现的接口，也可能是epoll实现的接口。

在OO设计中，理念是libev的reactor中，使用了I/O multiplexing的基类的引用或指针，I/O multiplexing Select和I/O multiplexing Epoll分别利用select、epoll实现基类的两个接口。最终使用时，根据操作系统提供接口的情况，把两个子类的其中一个赋值给reactor中的基类的引用或指针。

具体地，以linux场景下使用最多的epoll为例，来看看libev是如何实现backend_modify和backend_poll的。

**backend_modify**

backend_modify中，最重要的一行代码为

```c++
    if (expect_true (!epoll_ctl (backend_fd, oev && oldmask != nev ? EPOLL_CTL_MOD : EPOLL_CTL_ADD, fd, &ev)))
        return;
```

即如果要监听的事件发生了改变，则用`EPOLL_CTL_MOD`来修改，否则调用`EPOLL_CTL_ADD`来添加到epoll监听列表中。

**backend_poll**

backend_poll的代码如下

```c++
static void
epoll_poll (EV_P_ ev_tstamp timeout)
{
    int i;
    int eventcnt;

    /* epoll wait times cannot be larger than (LONG_MAX - 999UL) / HZ msecs, which is below */
    /* the default libev max wait time, however. */
    EV_RELEASE_CB;
    eventcnt = epoll_wait (backend_fd, epoll_events, epoll_eventmax, (int)ceil (timeout * 1000.));
    EV_ACQUIRE_CB;

    if (expect_false (eventcnt < 0)) {
        if (errno != EINTR)
            ev_syserr ("(libev) epoll_wait");

        return;
    }

    for (i = 0; i < eventcnt; ++i) {
        struct epoll_event *ev = epoll_events + i;

        int fd = (uint32_t)ev->data.u64; /* mask out the lower 32 bits */
        int want = anfds [fd].events;
        int got  = (ev->events & (EPOLLOUT | EPOLLERR | EPOLLHUP) ? EV_WRITE : 0)
                   | (ev->events & (EPOLLIN  | EPOLLERR | EPOLLHUP) ? EV_READ  : 0);

        /* check for spurious notification */
        /* we assume that fd is always in range, as we never shrink the anfds array */
        if (expect_false ((uint32_t)anfds [fd].egen != (uint32_t)(ev->data.u64 >> 32))) {
            /* recreate kernel state */
            postfork = 1;
            continue;
        }

        if (expect_false (got & ~want)) {
            anfds [fd].emask = want;

            /* we received an event but are not interested in it, try mod or del */
            /* I don't think we ever need MOD, but let's handle it anyways */
            ev->events = (want & EV_READ  ? EPOLLIN  : 0)
                         | (want & EV_WRITE ? EPOLLOUT : 0);

            /* pre-2.6.9 kernels require a non-null pointer with EPOLL_CTL_DEL, */
            /* which is fortunately easy to do for us. */
            if (epoll_ctl (backend_fd, want ? EPOLL_CTL_MOD : EPOLL_CTL_DEL, fd, ev)) {
                postfork = 1; /* an error occurred, recreate kernel state */
                continue;
            }
        }

        fd_event (EV_A_ fd, got);
    }

    /* if the receive array was full, increase its size */
    if (expect_false (eventcnt == epoll_eventmax)) {
        ev_free (epoll_events);
        epoll_eventmax = array_nextsize (sizeof (struct epoll_event), epoll_eventmax, epoll_eventmax + 1);
        epoll_events = (struct epoll_event *)ev_malloc (sizeof (struct epoll_event) * epoll_eventmax);
    }
}
```

1. 调用epoll\_wait，获取就绪事件列表
2. 对于就绪事件，放入到reactor的pending数组中存放，以方便reactor后续处理，具体地，在reactor中分析
3. epoll_wait调用的时候，需要指定数组存放就绪的文件描述符，如果某次就绪事件的个数等于数组元素的个数，说明可能是由于数组长度不够，导致所有的就绪事件并没有一次性的返回，影响程序性能。libev采用的方法是把对应的epoll_events数组扩容到原来的两倍，以应对网络链接数增大的场景。

## Timer

libev在timer管理采用了堆结构，采用了二叉堆和四叉堆。二叉堆对缓存不够友好，经常操作的元素是N和N/2，中间间隔元素太多，局部性不好，四叉堆在一定程度上会缓解这个问题。本节将会以四叉堆来分析libev的timer实现原理。

对于四叉堆，下标为k的元素，其儿子节点的下标范围为[4*k+1, 4*k+4]，反过来，对于下标为x的儿子节点，其父节点的下标为(x - 1) / 4。

分析堆实现之前，先来看看libev关于堆相关的宏定义

```c++
#define DHEAP 4
#define HEAP0 (DHEAP - 1) /* index of first element in heap */
#define HPARENT(k) ((((k) - HEAP0 - 1) / DHEAP) + HEAP0)
#define UPHEAP_DONE(p,k) ((p) == (k))
```

可以看出libev中，四叉堆第一个元素的下表是3，所以，由儿子节点计算父节点的公式变成了`(x - 3 - 1) / 4 + 3`。

### 堆元素的定义

```c++
    typedef struct {
        ev_tstamp at;
        int active; /* private */         \
    	int pending; /* private */            \
    	int priority; /* private */        \
    	void *data; /* rw */                \
    	void (*cb)(struct ev_loop *loop, struct ev_watcher *w, int revents);
    } ANHE;
```

其中at为定时器到期事件，堆排序则按照at的值由小到大排序。

### 堆操作

分为downheap、upheap、adjustheap和reheap。

**downheap**

```c++
   /* away from the root */
    inline_speed void
    downheap (ANHE *heap, int N, int k)
    {
        ANHE he = heap [k];
        ANHE *E = heap + N + HEAP0;

        for (;;) {
            ev_tstamp minat;
            ANHE *minpos;
            ANHE *pos = heap + DHEAP * (k - HEAP0) + HEAP0 + 1;

            /* find minimum child */
            if (expect_true (pos + DHEAP - 1 < E)) {
                /* fast path */                               (minpos = pos + 0), (minat = ANHE_at (*minpos));

                if (               ANHE_at (pos [1]) < minat) (minpos = pos + 1), (minat = ANHE_at (*minpos));

                if (               ANHE_at (pos [2]) < minat) (minpos = pos + 2), (minat = ANHE_at (*minpos));

                if (               ANHE_at (pos [3]) < minat) (minpos = pos + 3), (minat = ANHE_at (*minpos));
            } else if (pos < E) {
                /* slow path */                               (minpos = pos + 0), (minat = ANHE_at (*minpos));

                if (pos + 1 < E && ANHE_at (pos [1]) < minat) (minpos = pos + 1), (minat = ANHE_at (*minpos));

                if (pos + 2 < E && ANHE_at (pos [2]) < minat) (minpos = pos + 2), (minat = ANHE_at (*minpos));

                if (pos + 3 < E && ANHE_at (pos [3]) < minat) (minpos = pos + 3), (minat = ANHE_at (*minpos));
            } else
                break;

            if (ANHE_at (he) <= minat)
                break;

            heap [k] = *minpos;
            ev_active (ANHE_w (*minpos)) = k;

            k = minpos - heap;
        }

        heap [k] = he;
        ev_active (ANHE_w (he)) = k;
    }
```

在上面函数中，N为堆中元素个数，k为要调整的下标。

调整的思路为获取四个子节点的元素，取其中的最小值，然后和父节点比较，如果父节点小，说明已经是调整好了；如果父节点大，则把最小子节点赋值给父节点，以此为循环。最后，退出循环的时候，把开始调整的元素的值赋值给对应的元素。

在堆排序中，downheap用于的场景是，堆顶元素被弹出，然后，把最后一个堆元素赋值给第一个元素，然后重新调整。

**upheap**

```c++
    /* towards the root */
    inline_speed void
    upheap (ANHE *heap, int k)
    {
        ANHE he = heap [k];

        for (;;) {
            int p = HPARENT (k);

            if (UPHEAP_DONE (p, k) || ANHE_at (heap [p]) <= ANHE_at (he))
                break;

            heap [k] = heap [p];
            ev_active (ANHE_w (heap [k])) = k;
            k = p;
        }

        heap [k] = he;
        ev_active (ANHE_w (he)) = k;
    }
```
k为要调整的元素，跟其父节点比较，如果比父节点小，则往上赋值，否则退出循环。

**adjustheap**

```c++
    /* move an element suitably so it is in a correct place */
    inline_size void
    adjustheap (ANHE *heap, int N, int k)
    {
        if (k > HEAP0 && ANHE_at (heap [k]) <= ANHE_at (heap [HPARENT (k)]))
            upheap (heap, k);
        else
            downheap (heap, N, k);
    }
```

对于k == HEAP0的场景，都是用downheap的，因为不可能再往上调整了。

**reheap**

```c++
    /* rebuild the heap: this function is used only once and executed rarely */
    inline_size void
    reheap (ANHE *heap, int N)
    {
        int i;

        /* we don't use floyds algorithm, upheap is simpler and is more cache-efficient */
        /* also, this is easy to implement and correct for both 2-heaps and 4-heaps */
        for (i = 0; i < N; ++i)
            upheap (heap, i + HEAP0);
    }
```

初始化建堆的函数，只会在开始的时候调用一次。

libev的reactor依赖于timer的heap管理提供的接口来完成定时任务的管理。

## reactor

本节将分别以ev\_io和ev\_timer两种事件来分析libev的reactor模式的实现，先看其公共的数据结构的定义

```c++

    typedef struct {
        WL head;  //events watcher list
        unsigned char events; /* the events watched for */
        unsigned char reify;  /* flag set when this ANFD needs reification (EV_ANFD_REIFY, EV__IOFDSET) */
        unsigned char emask;  /* the epoll backend stores the actual kernel mask in here */
        unsigned char cdel;
    } ANFD;
    
    typedef struct {
        struct event_watcher *w;
        int events; /* the pending event set for the given watcher */
    } ANPENDING;
    
    struct ev_loop {
  		double ev_rt_now;
		int backend; /*EPOLL SEECT KQUEUE等*/
  		int activecnt; /* total number of active events ("refcount") */
  		int loop_done; /* signal by ev_break */

  		int backend_fd; /* e.g. epoll fd, created by epoll_create*/
  		void (*backend_modify)(EV_P_ int fd, int oev, int nev)); /* epoll_ctl */
  		void (*backend_poll)(EV_P_ ev_tstamp timeout)); /*  epoll_wait */

  		void (*invoke_cb)(struct ev_loop *loop);

  		ANFD *anfds; /* 把初始化后的 ev_io 结构体绑定在 anfds[fd].head 事件链表上，方便根据 fd 直接查找。*/

  		int *fdchanges; /* 监听的事件fd列表 */
  		ANPENDING *pendings [NUMPRI]; /* 存放等待被调用 callback 的 watcher */
        ANHE *  timers; /*管理timer任务的堆结构*/
}
```

其中ev_loop是libev中的reactor的结构体，最重要的一些定义都列举在上面的代码中，具体含义在后面注释中备注。下面分别讨论libev为ev\_io事件和ev\_timer事件提供的一些接口，分析这些接口与ev\_loop内部的数据的交互流程。

### ev\_io

libev给ev\_io事件提供的接口包括

- `ev_io_start`
- `ev_io_stop`

**ev\_io\_start处理流程**

![libev ev_io_start](http://oserror.com/images/libev_ev_io_start.png)

1. 对于已经是active的watcher则直接返回，表明已经添加过
2. 如果是非active的，则首先置成active
3. 然后把watcher加入到对应fd所在的anfds数组的watcher链表中
4. 把fd加入到fdchanges数组的末尾

备注：这里修改了anfs和fdchanges两个数组

**ev\_io\_stop处理流程**

ev\_io\_stop的处理流程相当于把ev\_io\_start所做的事情都撤销，不再赘述。

### ev\_timer

libev给ev\_timer提供的接口包括

- `ev_timer_start`
- `ev_timer_stop`

**ev\_timer\_start处理流程**

1. 把active置成数组的下标
2. 把新加入的ev\_timer\_watcher放到timers数组最末尾
3. 调用upheap调整堆

**ev\_timer\_stop处理流程**

1. 把watcher对应下标的事件用timers数组最后一个元素覆盖
2. 从watcher调用adjustheap调整堆

到目前为止，讨论了libev的reactor中ev\_io的start和stop，ev\_timer的start和stop操作，相当于reactor模式中的Register和Unregister接口，接下来讨论最后的HandleEvents接口。

## HandleEvents

libev的HandleEvents接口为ev\_run函数，其流程为

![libev ev_run](http://oserror.com/images/libev_ev_event_loop.png)

fd\_reify的操作为遍历fdchanges数组，然后根据fd从anfds中拿具体数据，最后调用epoll\_modify函数注册事件。

timers\_reify的操作为对于设置了repeat的timer，更新过期事件，然后调用调整堆操作；对于未设置repeat的过期事件，则直接清除，调用的是ev\_timer\_stop操作

backend\_poll和具体的实现有关，以epoll为例，之前讨论过对于就绪事件，会放到reactor的pending数组中，具体的为调用fd_event接口，最终调用ev\_feed\_event来更新pending数组，如下

```c++
    void noinline
    ev_feed_event (EV_P_ void *w, int revents)
    {
        W w_ = (W)w;
        int pri = ABSPRI (w_);

        if (expect_false (w_->pending))
            pendings [pri][w_->pending - 1].events |= revents;
        else {
            w_->pending = ++pendingcnt [pri];
            array_needsize (ANPENDING, pendings [pri], pendingmax [pri], w_->pending, EMPTY2);
            pendings [pri][w_->pending - 1].w      = w_;
            pendings [pri][w_->pending - 1].events = revents;
        }
    }
```

即把就绪事件，放到对应的优先级队列的最后。

最后，调用ev\_invoke\_pending来处理就绪事件，如下

```c++
    void noinline
    ev_invoke_pending (EV_P)
    {
        int pri;

        for (pri = NUMPRI; pri--; )
            while (pendingcnt [pri]) {
                ANPENDING *p = pendings [pri] + --pendingcnt [pri];

                /*assert (("libev: non-pending watcher on pending list", p->w->pending));*/
                /* ^ this is no longer true, as pending_w could be here */

                p->w->pending = 0;
                EV_CB_INVOKE (p->w, p->events);
                EV_FREQUENT_CHECK;
            }
    }
```

具体地，即遍历整个pending数组，来调用事件上的处理函数即可。

PS:
本博客更新会在第一时间推送到微信公众号，欢迎大家关注。

![qocde_wechat](http://oserror.com/images/qcode_wechat.jpg)

# 参考文献

- https://github.com/enki/libev
- http://www.jianshu.com/p/3299e19d9bf4
- http://m.blog.csdn.net/article/details?id=49203223
- Unix环境高级编程

