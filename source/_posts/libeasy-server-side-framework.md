---
title: libeasy服务端框架实现原理
date: 2016-07-17 19:10:49
categories: 后台开发
tags:
  - 网络编程
  - libeasy
  - 高性能
---

# Introduction

本文分析libeasy服务端框架实现原理，主要分析libeasy是如何处理以下事件的

- libeasy服务端连接建立
- libeasy服务端连接关闭
- libeasy服务端可读事件
- libeasy服务端可写事件

# libeasy服务端框架实现原理

本节先讨论libeasy的网络和线程模型，接着描述libeasy是如何组织连接、消息、请求等之间的关系，即libeasy资源管理，最后分析其如何处理服务端连接建立、服务端连接关闭、服务端可读事件以及服务端可写事件的。

## libeasy网络和线程模型

libeasy网络模型基于libev的reactor模型，具体分析请参看文章([libev设计与实现](http://oserror.com/backend/libev-analysis/))，本节主要描述其线程模型。

libeasy支持两种常见的线程模型，一是IO线程和工作线程共用相同线程，二是IO线程和工作线程分开。

### I/O线程和工作线程共用
![libeasy thread model 1](http://o8m1nd933.bkt.clouddn.com/blog/libeasy/libeasy_thread_model_1.png)

如上图，I/O线程和工作线程共用的线程模型中，实际上是没有专门的工作线程的，I/O线程不仅需要负责处理I/O，还需要真正地处理请求，计算结果。一般典型的处理流程为

- Process Read I/O: 处理读I/O
- Process: 解析请求，计算结果
- Process Write I/O： 处理写I/O，把计算结果返回给客户端

这种线程模型的特点是

- 处理流程相对简单，解析好请求后就能直接在同一线程处理，省去了线程切换的开销，非常适合Process耗费时间较小的请求
- 由于Process过程需要耗费时间，对于大任务，可能时间较长，会影响其他请求的处理

### I/O线程和工作线程独立

![libeasy thread model 2](http://o8m1nd933.bkt.clouddn.com/blog/libeasy/libeasy_thread_model_2.png)

如上图，在I/O线程和工作线程独立的线程模型中，有专门的工作线程来处理请求，计算结果，I/O线程仅仅需要做读写数据相关的操作。在这种线程模型下，整个流程为

- Process Read I/O:处理读数据，然后解析请求，生成任务，推送到工作线程的队列中，然后以异步事件方式通知工作线程处理
- Process: 工作线程接收到异步事件后，从其工作队列中拿出任务，依次处理，处理完成后，生成结果，放到I/O线程的队列中，然后以异步事件方式通知I/O线程处理
- Process Write I/O：I/O线程收到通知后，依次处理写数据请求

这种线程模型的特点是

- I/O和计算分开处理，会引入线程切换开销，比较适合Process耗费时间长的任务请求
- 对于小任务请求不适合，大量时间耗费在线程切换开销

## libeasy资源管理

![ibeasy resource management](http://o8m1nd933.bkt.clouddn.com/blog/libeasy/libeasy_resource_management.png)

libeasy的资源管理方式如上图所示，主要包括

- easy_baseth_t: libeasy中入口结构体，主要包括libev的网络框架和easy\_io\_t。
- easy_io_t: libeasy io入口结构体，包含监听相关的结构体，io thread pool 以及 request thread pool
- easy_thread_pool_t： libeasy线程池结构体，通过zero byte数组来作为线程池数组
- easy_thread_t： libeasy中不存在这个结构体，是笔者添加的，libeasy中采用的是宏定义实现这个功能
- easy_io_thread_t: libeasy中负责io的线程结构体，其中包含了连接队列，请求队列等等
- easy_request_thread_t: libeasy负责任务处理的线程结构体，其中包含了任务队列等
- easy_connection_t: libeasy处理网络链接的结构体，其中包括文件描述符fd，连接地址，监听事件处理程序，以及连接对应的请求队列等等。
- easy_message_t: libeasy中消息管理结构体，其中一个消息可能包含几个请求，一个请求是按照用户自定义格式的完整的消息包。
- easy_request_t: libeasy中管理请求的结构体，按照用户自定义协议，一个完整的协议包看作一个请求。

在libeasy中，资源之间有比较多的联系，这也是libeasy组织资源之间关系的方式

- 整个easy\_io\_t中包含IO线程池和工作线程池
- 一个io线程中(`easy_io_thread_t`)会一般会处理多个连接(`easy_connection_t`)
- 一个连接中(`easy_connection_t`)一般会包括一个或多个message(`easy_message_t`)
- 一个message中(`easy_message_t`)一般会有一个或多个request(`easy_request_t`)
- 一个工作线程中(`easy_request_thread_t`)一般会处理一个或多个request(`easy_request_t`)，其中request也称作是任务

除了上述资源之外，libeasy还有个非常重要的结构体，如下

```c
struct easy_io_handler_pt {
    void               *(*decode)(easy_message_t *m);
    int                 (*encode)(easy_request_t *r, void *packet);
    easy_io_process_pt  *process;
    int                 (*batch_process)(easy_message_t *m);
    easy_io_cleanup_pt  *cleanup;
    uint64_t            (*get_packet_id)(easy_connection_t *c, void *packet);
    int                 (*on_connect) (easy_connection_t *c);
    int                 (*on_disconnect) (easy_connection_t *c);
    int                 (*new_packet) (easy_connection_t *c);
    int                 (*on_idle) (easy_connection_t *c);
    void                *user_data, *user_data2;
    int                 is_uthread;
};
```

这个结构体在`easy_listen_t`和`easy_connection_t`中有使用，在libeasy框架监控到网络事件发生后，其会调用`easy_io_handler_pt`中的用户自定义的方法来处理。通过用户自定义的处理函数，可以把libeasy扩展到各种各样的应用场景，例如，http服务端，mysql服务端等等。具体地，每个函数在何时被调用，会在下节详细地分析。

## libeasy事件处理

本节分析libeasy如何处理网络中的常见的四个事件

- libeasy服务端连接建立
- libeasy服务端连接关闭
- libeasy服务端可读事件
- libeasy服务端可写事件

### libeasy服务端连接建立

在说明libeasy如何处理服务端连接建立前，先说明libeasy建立listen端口的流程。

整个调用流程如下

```c
easy_connection_listen_addr
		|
easy_add_listen_addr
		|
ev_io_init(&l->read_watcher[i], easy_connection_on_accept, fd, EV_READ | EV_CLEANUP)
```
`easy_connection_on_accept`负责listen端口处理链接建立的函数，具体地逻辑如下

```c
static void easy_connection_on_accept(struct ev_loop *loop, ev_io *w, int revents)
{
    int                     fd;
    easy_listen_t           *listen;
    struct sockaddr_in      addr;
    socklen_t               addr_len;
    easy_connection_t       *c;
    easy_io_thread_t        *ioth;
    char                    buffer[32];

    listen = (easy_listen_t *) w->data;
    addr_len = sizeof(struct sockaddr_in);
    assert(w->fd == listen->fd);

    // accept
    if ((fd = accept(w->fd, (struct sockaddr *)&addr, &addr_len)) < 0) return;

    // 为新连接创建一个easy_connection_t对象
    if ((c = easy_connection_new()) == NULL) {
        easy_error_log("easy_connection_new\n");
        close(fd);
        return;
    }

    // 设为非阻塞
    easy_socket_non_blocking(fd);

    // 初始化
    c->fd = fd;
    c->type = EASY_TYPE_SERVER;
    c->handler = listen->handler;
    c->addr = *((easy_addr_t *)&addr);

    // 事件初始化
    ev_io_init(&c->read_watcher, easy_connection_on_readable, fd, EV_READ);
    ev_io_init(&c->write_watcher, easy_connection_on_writable, fd, EV_WRITE);
    ev_init(&c->timeout_watcher, easy_connection_on_timeout_conn);
    c->read_watcher.data = c;
    c->write_watcher.data = c;
    c->timeout_watcher.data = c;
    c->ioth = ioth = (easy_io_thread_t *)easy_baseth_self;
    c->loop = loop;
    c->start_time = ev_now(ioth->loop);

    easy_debug_log("accpet from '%s', connection=%p, fd=%d\n",
                   easy_inet_addr_to_str(&c->addr, buffer, 32), c, c->fd);

    // on connect
    if (c->handler->on_connect)
        (c->handler->on_connect)(c);

    // start idle
    if (c->handler->on_idle) {
        double t = easy_max(1.0, c->idle_time / 2000.0);
        ev_timer_set(&c->timeout_watcher, 0.0, t);
        ev_timer_again(c->loop, &c->timeout_watcher);
    }


    easy_atomic_add(&ioth->eio->connect_num, 1);
    int th_idx = (ioth->eio->connect_num) % (ioth->eio->io_thread_count);
    th_idx = th_idx<0 ? (0-th_idx) : th_idx;
    // 让出来给其他的线程
    if (ioth->eio->listen_all == 0 && listen->old_ioth == NULL
            && listen->curr_ioth == ioth) {
        easy_io_thread_t *nextth;
        listen->old = listen->cur;
        listen->curr_ioth = NULL;
        //listen->old_ioth = ioth;
        //ioth->listen_watcher.repeat = 0.5;
        ev_io_stop(ioth->loop, &listen->read_watcher[listen->old]);
        listen->old_ioth = NULL;
        nextth = easy_thread_pool_index(ioth->eio->io_thread_pool,th_idx);
        easy_unlock(&listen->listen_lock);

        ev_async_send(nextth->loop, &nextth->listen_watcher);

        //ev_timer_again (ioth->loop, &ioth->listen_watcher);

    }

    // start read
    easy_list_add_tail(&c->conn_list_node, &c->ioth->connected_list);
    c->event_status = EASY_EVENT_READ;
    easy_connection_on_readable(loop, &c->read_watcher, 0);

    return;
}
```

其处理流程如下

- 调用accept，建立新的网络连接，然后创建`easy_connection_t`结构体来处理此连接
- 设置此fd是非阻塞的，这个是多路复用的标配了
- 初始化此connection的读事件、写事件和tiemout事件的处理程序
- 调用用户自定义的`on_connect`函数
- 把listen的监听任务让出来给其他的线程
- 把connection加入到io thread的connected_list中
- 开始尝试处理一次READ事件，这个具体地到处理可读事件逻辑

### libeasy服务端连接关闭

连接关闭的处理逻辑在`easy_connection_destroy`中，如下

```c
void easy_connection_destroy(easy_connection_t *c)
{
    easy_message_t  *m, *m2;
    easy_io_t       *eio;

    // release session
    easy_connection_wakeup_session(c);

    // disconnect
    eio = c->ioth->eio;

    if (c->status != EASY_CONN_CLOSE && c->handler->on_disconnect) {
        (c->handler->on_disconnect)(c);
    }

    // refcount
    if (c->status != EASY_CONN_CLOSE && c->pool->ref > 0 && eio->stoped == 0) {
        ev_io_stop(c->loop, &c->read_watcher);
        ev_io_stop(c->loop, &c->write_watcher);
        c->status = EASY_CONN_CLOSE;

        // wakeup other thread
        if (c->type == EASY_TYPE_SERVER) {
            easy_thread_pool_t *tp = eio->thread_pool;

            while (tp) {
                easy_baseth_pool_on_wakeup(tp);
                tp = tp->next;
            }
        }

        if (c->pool->ref > 0) {
            ev_timer_set(&c->timeout_watcher, 0.0, 0.5);
            ev_timer_again(c->loop, &c->timeout_watcher);
        }
    }

    if (c->type == EASY_TYPE_SERVER) {
           easy_request_t *r, *n;
           easy_list_for_each_entry_safe(r, n, &c->session_list, request_list_node) {
               easy_request_server_done(r);
           }
       }


    if (c->pool->ref > 0 && eio->stoped == 0) return;

    // release request
    if (c->type == EASY_TYPE_SERVER) {
        easy_request_t *r, *n;
        easy_list_for_each_entry_safe(r, n, &c->session_list, request_list_node) {
            easy_list_del(&r->request_list_node);
        }
    }

    // release message
    easy_list_for_each_entry_safe(m, m2, &c->message_list, message_list_node) {
        easy_message_destroy(m, 1);
    }
    easy_buf_chain_clear(&c->output);
    ev_io_stop(c->loop, &c->read_watcher);
    ev_io_stop(c->loop, &c->write_watcher);
    ev_timer_stop(c->loop, &c->timeout_watcher);

    // close
    if (c->fd >= 0) {
        easy_debug_log("[%p] connection close: %d\n", c, c->fd);

        if (!c->read_eof) {
            char buf[EASY_POOL_PAGE_SIZE];

            while (read(c->fd, buf, EASY_POOL_PAGE_SIZE) > 0);
        }

        close(c->fd);
        c->fd = -1;
    }

    // autoreconn
    if (c->auto_reconn && eio->stoped == 0) {
        c->status = EASY_CONN_AUTO_CONN;
        double t = c->reconn_time / 1000.0 * (1 << c->reconn_fail);

        if (t > 15) t = 15;

        if (c->reconn_fail < 16) c->reconn_fail ++;

        easy_warn_log("[%p] connection reconn_time: %f, reconn_fail: %d\n", c, t, c->reconn_fail);
        ev_timer_set(&c->timeout_watcher, 0.0, t);
        ev_timer_again(c->loop, &c->timeout_watcher);
        return;
    }

    easy_list_del(&c->conn_list_node);

    if (c->client) c->client->c = NULL;

    if (eio->stoped) c->pool->ref = 0;

    easy_pool_destroy(c->pool);
}
```

- 调用用户自定义的on\_disconnect函数
- 取消注册读写等事件，清理所有的request，清理所有的message
- 读完所有的剩余数据，关闭fd

### libeasy服务端可读事件

其中，read事件的注册是在`easy_connection_on_accept`中完成的，在libeasy服务端连接建立中已经说明过，这里再描述以下它的调用流程。

```c
    ev_io_init(&c->read_watcher, easy_connection_on_readable, fd, EV_READ);
    ev_io_init(&c->write_watcher, easy_connection_on_writable, fd, EV_WRITE);
    ev_init(&c->timeout_watcher, easy_connection_on_timeout_conn);
    easy_list_add_tail(&c->conn_list_node, &c->ioth->connected_list);
    c->event_status = EASY_EVENT_READ;
```
其中，初始化了读、写和超时事件的处理函数，但只注册了读事件。

读事件的具体逻辑是在`easy_connection_on_readable`中处理的，其代码如下

```c
static void easy_connection_on_readable(struct ev_loop *loop, ev_io *w, int revents)
{
    easy_connection_t   *c;
    easy_message_t      *m;
    int                 n;

    c = (easy_connection_t *)w->data;
    assert(c->fd == w->fd);

    // 防止请求过多
    if (c->type == EASY_TYPE_SERVER && (c->doing_request_count > EASY_CONN_DOING_REQ_CNT ||
                                        c->ioth->doing_request_count > EASY_IOTH_DOING_REQ_CNT)) {
        easy_warn_log("c->doing_request_count: %d, c->ioth->doing_request_count: %d\n",
                      c->doing_request_count, c->ioth->doing_request_count);
        goto error_exit;
    }

    // 最后的请求, 如果数据没完, 需要继续读
    m = easy_list_get_last(&c->message_list, easy_message_t, message_list_node);

    // 第一次读或者上次读完整了, 重新建一个easy_message_t
    if (m == NULL || m->status != EASY_MESG_READ_AGAIN) {
        if ((m = easy_message_create(c)) == NULL) {
            easy_error_log("easy_message_create failure, c=%p\n", c);
            goto error_exit;
        }
    }

    // 检查buffer大小
    if (easy_buf_check_read_space(m->pool, m->input, m->next_read_len) != EASY_OK) {
        easy_error_log("easy_buf_check_read_space failure, m=%p, len=%d\n", m, m->next_read_len);
        goto error_exit;
    }

    // 从conn里读入数据
    if ((n = read(c->fd, m->input->last, m->next_read_len)) <= 0) {
        easy_debug_log("n: %d, errno: %s(%d)\n", n, strerror(errno), errno);
        goto error_exit;
    }

    c->read_eof = (n < m->next_read_len);
    c->last_time = ev_now(loop);
    c->reconn_fail = 0;

    easy_debug_log("read: %d, fd: %d\n", n, c->fd);

    if (easy_log_level >= EASY_LOG_TRACE) {
        char btmp[128];
        easy_trace_log("read: %d => %s", n, easy_string_tohex(m->input->last, n, btmp, 128));
    }

    easy_atomic_add(&EASY_IOTH_SELF->eio->recv_byte, n);
    m->input->last += n;

    if (c->default_message_len < EASY_IO_BUFFER_SIZE && m->next_read_len == n)
        c->default_message_len = EASY_IO_BUFFER_SIZE;

    // client
    if (EASY_ERROR == ((c->type == EASY_TYPE_CLIENT) ?
                       easy_connection_do_response(m) : easy_connection_do_request(m))) {
        easy_debug_log("c->type=%d, fd=%d\n", c->type, c->fd);
        goto error_exit;
    }

    return;
error_exit:
    easy_connection_destroy(c);
}
```
处理逻辑主要包括两块

- 如果第一次读数据，需要创建`easy_message_t`，作为载体
- 读数据
- 调用`easy_connection_do_request`处理读到的数据

`easy_connection_do_request`的处理逻辑如下

```c
static int easy_connection_do_request(easy_message_t *m)
{
    easy_connection_t       *c;
    void                    *packet;
    easy_request_t          *r, *rn;
    int                     cnt, ret;

    cnt = 0;
    c = m->c;

    // 处理buf, decode
    while (m->input->pos < m->input->last) {
        if ((packet = (c->handler->decode)(m)) == NULL) {
            if (m->status != EASY_ERROR)
                break;

            easy_warn_log("decode error, m=%p, fd=%d\n", m, c->fd);
            return EASY_ERROR;
        }

        // new request
        r = (easy_request_t *)easy_pool_calloc(m->pool, sizeof(easy_request_t));
        r->ms = (easy_message_session_t *)m;
        r->ipacket = packet;    //进来的数据包

        // add m->request_list
        easy_list_add_tail(&r->request_list_node, &m->request_list);
        cnt ++;
    }

    // cnt
    if (cnt) {
        m->request_list_count += cnt;
        c->doing_request_count += cnt;
        easy_atomic32_add(&c->ioth->doing_request_count, cnt);
        m->recycle_cnt ++;
    }

    if ((m = easy_connection_recycle_message(m)) == NULL)
        return EASY_ERROR;

    m->status = ((m->input->pos < m->input->last) ? EASY_MESG_READ_AGAIN : 0);


    // batch process
    if (c->handler->batch_process)
        (c->handler->batch_process)(m);

    // process
    cnt = 0;
    easy_list_for_each_entry_safe(r, rn, &m->request_list, request_list_node) {
        easy_list_del(&r->request_list_node);
        EASY_IOTH_SELF->done_request_count ++;

        // process
        if ((ret = (c->handler->process)(r)) == EASY_ERROR)
            return EASY_ERROR;

        if (ret == EASY_OK && easy_connection_request_done(r) == EASY_OK) {
            cnt ++;
        }

        // write to socket
        if (cnt >= 128) {
            cnt = 0;

            if (easy_connection_write_socket(c) == EASY_ABORT)
                return EASY_ERROR;
        }
    }

    // 所有的request都有reply了,一起才响应
    if (easy_connection_write_socket(c) == EASY_ABORT) {
        return EASY_ERROR;
    }

    if (m->request_list_count == 0 && m->status != EASY_MESG_READ_AGAIN) {
        easy_message_destroy(m, 1);
    }

    // 加入监听
    if (c->event_status == EASY_EVENT_READ && !c->wait_close)
        easy_connection_evio_start(c);

    easy_connection_redispatch_thread(c);

    return EASY_OK;
}
```
主要的逻辑如下

- 解析读入数据，decode成一个个的请求
- 调用用户自定义地batch\_process函数
- 遍历每个请求，调用用户自定义的process函数
- 当处理了一批请求后，调用`easy_connection_write_socket`写结果
- 继续加入读监控事件

decode函数的一个简单的例子如下

```
static inline void *easy_simple_decode(easy_message_t *m)
{
    easy_simple_packet_t    *packet;
    uint32_t                len, datalen;

    // length
    if ((len = m->input->last - m->input->pos) < EASY_SIMPLE_PACKET_HEADER_SIZE)
        return NULL;

    // data len
    datalen = *((uint32_t *)m->input->pos);

    if (datalen > 0x4000000) { // 64M
        easy_error_log("data_len is invalid: %d\n", datalen);
        m->status = EASY_ERROR;
        return NULL;
    }

    // 长度不够
    len -= EASY_SIMPLE_PACKET_HEADER_SIZE;

    if (len < datalen) {
        m->next_read_len = datalen - len;
        return NULL;
    }

    // alloc packet
    if ((packet = (easy_simple_packet_t *)easy_pool_calloc(m->pool,
                  sizeof(easy_simple_packet_t))) == NULL) {
        m->status = EASY_ERROR;
        return NULL;
    }

    packet->chid = *((uint32_t *)(m->input->pos + sizeof(int)));
    m->input->pos += EASY_SIMPLE_PACKET_HEADER_SIZE;
    packet->len = datalen;
    packet->data = (char *)m->input->pos;
    m->input->pos += datalen;

    return packet;
}
```

主要的逻辑就是按照既定地协议来解析数据。

batch\_process和process一般是作为两种不同的方式，用户一般只使用其中的一种。不管采用那种处理方式，都分为以下两种处理方式

1. I/O线程和工作线程独立的方式
2. I/O线程和工作线程共用的方式

#### I/O线程和工作线程独立的方式

这种处理方式，process函数一般会调用`easy_thread_pool_push_message`，如下

```c
int easy_thread_pool_push_message(easy_thread_pool_t *tp, easy_message_t *m, uint64_t hv)
{
    easy_request_thread_t   *rth;

    // dispatch
    if (hv == 0) hv = easy_hash_key((long)m->c);

    rth = (easy_request_thread_t *)easy_thread_pool_hash(tp, hv);

    // 引用次数
    m->c->pool->ref += m->request_list_count;
    easy_atomic_add(&m->pool->ref, m->request_list_count);
    easy_pool_set_lock(m->pool);

    easy_spin_lock(&rth->thread_lock);
    easy_list_join(&m->request_list, &rth->task_list);
    rth->task_list_count += m->request_list_count;
    ev_async_send(rth->loop, &rth->thread_watcher);
    easy_spin_unlock(&rth->thread_lock);

    easy_list_init(&m->request_list);

    return EASY_OK;
}
```
可以看出，该函数的最主要的功能是把message推送到request thread，由它继续处理，接着看其处理的流程，先分析`easy_async_send(rth->loop, &rth->thread_watcher)`是如何唤醒request thread的，以及唤醒的是request thread的哪个处理函数。

```c
easy_thread_pool_create(cnt, callback, NULL)
		|
easy_thread_pool_create_ex
        |
easy_baseth_init(rth, tp, start, easy_request_on_wakeup)
        |
ev_async_init(&th->thread_watcher, wakeup)
```
可以看出，最终唤醒的是request thread的`easy_request_on_wakeup`来处理。

```c
static void easy_request_on_wakeup(struct ev_loop *loop, ev_async *w, int revents)
{
    easy_request_thread_t       *th;
    easy_list_t                 request_list;
    easy_list_t                 session_list;

    th = (easy_request_thread_t *) w->data;

    // 取回list
    easy_spin_lock(&th->thread_lock);
    th->task_list_count = 0;
    easy_list_movelist(&th->task_list, &request_list);
    easy_list_movelist(&th->session_list, &session_list);
    easy_spin_unlock(&th->thread_lock);

    easy_request_doreq(th, &request_list);
    easy_request_dosess(th, &session_list);
}

static void easy_request_doreq(easy_request_thread_t *th, easy_list_t *request_list)
{
    easy_request_t              *r, *r2;
    easy_connection_t           *c;
    int                         cnt;

    cnt = 0;
    char ioth_flag[th->eio->io_thread_count];
    easy_list_t ioth_list[th->eio->io_thread_count];
    memset(ioth_flag, 0, sizeof(ioth_flag));

    // process
    easy_list_for_each_entry_safe(r, r2, request_list, request_list_node) {
        c = r->ms->c;
        easy_list_del(&r->request_list_node);

        // 处理
        if (c->status != EASY_CONN_CLOSE) {
            r->retcode = (th->process)(r, th->args);
        } else {
            r->retcode = EASY_ERROR;
        }

        if (!ioth_flag[c->ioth->idx])
            easy_list_init(&ioth_list[c->ioth->idx]);

        easy_list_add_tail(&r->request_list_node, &ioth_list[c->ioth->idx]);
        ioth_flag[c->ioth->idx] = 1;

        if (++ cnt >= 32) {
            easy_request_wakeup_ioth(th->eio, ioth_flag, ioth_list);
            cnt = 0;
        }
    }

    if (cnt > 0) {
        easy_request_wakeup_ioth(th->eio, ioth_flag, ioth_list);
    }
}
```

`easy_request_wakeup`会调用`easy_request_doreq`来最终处理请求，它的工作逻辑是

- 遍历所有请求，调用创建线程池时，用户提供的自定义的函数
- 处理完成后，调用`easy_request_wakeup_ioth`唤醒I/O线程来处理结果数据的发送

其中，用户自定义的函数一般也是处理输入数据，计算结果，然后产生输出的结果数据，但这个和`easy_io_handler_pt`中的process不是一个函数，但处理流程是类似的，主要是对请求的分析和计算过程。

`easy_request_wakeup_ioth`的处理逻辑为

```c
static void easy_request_wakeup_ioth(easy_io_t *eio, char *ioth_flag, easy_list_t *ioth_list)
{
    int                 i;
    easy_io_thread_t    *ioth;

    for(i = 0; i < eio->io_thread_count; i++) {
        if (!ioth_flag[i])
            continue;

        ioth_flag[i] = 0;
        ioth = (easy_io_thread_t *)easy_thread_pool_index(eio->io_thread_pool, i);

        // dispatch to ioth
        easy_spin_lock(&ioth->thread_lock);
        easy_list_join(&ioth_list[ioth->idx], &ioth->request_list);
        ev_async_send(ioth->loop, &ioth->thread_watcher);
        easy_spin_unlock(&ioth->thread_lock);
    }
}
```
接着看I/O thread的thread\_watcher唤醒的是I/O线程的哪个处理函数。

```c
在easy_eio_create函数中
easy_baseth_init(ioth, tp, easy_io_on_thread_start, easy_connection_on_wakeup);
```
I/O thread的唤醒是由`easy_connection_on_wakeup`来处理，如下

```c
easy_connection_send_response(&request_list);
```

而`easy_connection_send_response`的处理逻辑为

```c
    // foreach write socket
    if (wlist.tail) {
        wlist.tail->next = NULL;
        nc = wlist.head;

        while((c = nc)) {
            nc = c->next;
            c->next = NULL;

            if (easy_connection_write_socket(c) == EASY_ABORT) {
                easy_connection_destroy(c);
            } else if (c->type == EASY_TYPE_SERVER) {
                easy_connection_redispatch_thread(c);
            }
        }
    }
```
调用`easy_connection_write_socket`来处理一个个的`easy_connection_t`的输出数据，具体的逻辑为

```c
int easy_connection_write_socket(easy_connection_t *c)
{
    int ret;

    // 空的直接返回
    if (easy_list_empty(&c->output))
        return EASY_OK;

    // 加塞
    if (EASY_IOTH_SELF->eio->tcp_cork && c->tcp_cork_flag == 0) {
        easy_socket_set_tcpopt(c->fd, TCP_CORK, 1);
        c->tcp_cork_flag = 1;
    }

    ret = easy_socket_write(c->fd, &c->output);

    if (ret == EASY_ERROR) {
        char buffer[32];
        easy_warn_log("ret=%d, addr: %s, error: %s (%d)\n", ret,
                      easy_inet_addr_to_str(&c->addr, buffer, 32), strerror(errno), errno);
        c->conn_has_error = 1;
        return EASY_ABORT;
    }

    c->last_time = ev_now(c->loop);

    return easy_connection_write_again(c);
}
```
调用`easy_socket_write`来写数据，如果一次性没写完，则会在`easy_connection_write_again`中继续处理，如下

```c
    if (easy_list_empty(&c->output) == 0) {
        ev_io_start(c->loop, &c->write_watcher);
        return EASY_AGAIN;
    }
```
主要的逻辑就是继续监听写事件，等待下次唤醒。

#### I/O线程和工作线程共用

调用用户自定义的process处理(process，一般包含对请求的计算，生成输出的数据。)，然后调用`easy_connection_write_socket`写数据。注意，这种处理方式是I/O线程和工作线程共用的方式，process是在I/O线程中处理的，这种线程模型逻辑简单，比较容易理解，编码也相对容易很多。

### libeasy服务端可写事件

当fd可写时，libeasy会调用`easy_connection_on_writable`来处理，而它又会调用`easy_connection_write_socket`来处理。在上面一节已经分析过其处理逻辑了，这里不再讨论了。

PS:
本博客更新会在第一时间推送到微信公众号，欢迎大家关注。

![qocde_wechat](http://o8m1nd933.bkt.clouddn.com/blog/qcode_wechat.jpg)

# 参考文献

- [OceanBase使用libeasy原理源码分析：服务器端](http://www.cnblogs.com/foxmailed/archive/2013/02/17/2908180.html)




