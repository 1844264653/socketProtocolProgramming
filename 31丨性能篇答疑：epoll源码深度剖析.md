# 31丨性能篇答疑：epoll源码深度剖析

**在性能篇中，我主要围绕 C10K 问题进行了深入剖析，最后引出了事件分发机制和多线程。可以说，基于 epoll 的事件分发能力，是 Linux 下高性能网络编程的不二之选。如果你觉得还不过瘾，期望有更深刻的认识和理解，那么在性能篇的答疑中，我就带你一起梳理一下 epoll 的源代码，从中我们一定可以有更多的发现和领悟。**



### 基本数据结构

在开始研究源代码之前，我们先看一下 epoll 中使用的数据结构，分别是 eventpoll、epitem 和 eppoll_entry。

我们先看一下 **eventpoll 这个数据结构，这个数据结构是我们在调用 epoll_create 之后内核侧创建的一个句柄，表示了一个 epoll 实例。后续如果我们再调用 epoll_ctl 和 epoll_wait 等，都是对这个 eventpoll 数据进行操作，这部分数据会被保存在 epoll_create 创建的匿名文件 file 的 private_data 字段中**。

~~~LINUX

/*
 * This structure is stored inside the "private_data" member of the file
 * structure and represents the main data structure for the eventpoll
 * interface.
 */
struct eventpoll {
    /* Protect the access to this structure */
    spinlock_t lock;

    /*
     * This mutex is used to ensure that files are not removed
     * while epoll is using them. This is held during the event
     * collection loop, the file cleanup path, the epoll file exit
     * code and the ctl operations.
     */
    struct mutex mtx;

    /* Wait queue used by sys_epoll_wait() */
    //这个队列里存放的是执行epoll_wait从而等待的进程队列
    wait_queue_head_t wq;

    /* Wait queue used by file->poll() */
    //这个队列里存放的是该eventloop作为poll对象的一个实例，加入到等待的队列
    //这是因为eventpoll本身也是一个file, 所以也会有poll操作
    wait_queue_head_t poll_wait;

    /* List of ready file descriptors */
    //这里存放的是事件就绪的fd列表，链表的每个元素是下面的epitem
    struct list_head rdllist;

    /* RB tree root used to store monitored fd structs */
    //这是用来快速查找fd的红黑树
    struct rb_root_cached rbr;

    /*
     * This is a single linked list that chains all the "struct epitem" that
     * happened while transferring ready events to userspace w/out
     * holding ->lock.
     */
    struct epitem *ovflist;

    /* wakeup_source used when ep_scan_ready_list is running */
    struct wakeup_source *ws;

    /* The user that created the eventpoll descriptor */
    struct user_struct *user;

    //这是eventloop对应的匿名文件，充分体现了Linux下一切皆文件的思想
    struct file *file;

    /* used to optimize loop detection check */
    int visited;
    struct list_head visited_list_link;

#ifdef CONFIG_NET_RX_BUSY_POLL
    /* used to track busy poll napi_id */
    unsigned int napi_id;
#endif
};
~~~

你能看到在代码中我提到了 epitem，这个 epitem 结构是干什么用的呢？

每当我们调用 epoll_ctl 增加一个 fd 时，内核就会为我们创建出一个 epitem 实例，并且把这个实例作为红黑树的一个子节点，增加到 eventpoll 结构体中的红黑树中，对应的字段是 rbr。这之后，查找每一个 fd 上是否有事件发生都是通过红黑树上的 epitem 来操作。

~~~LINUX

/*
 * Each file descriptor added to the eventpoll interface will
 * have an entry of this type linked to the "rbr" RB tree.
 * Avoid increasing the size of this struct, there can be many thousands
 * of these on a server and we do not want this to take another cache line.
 */
struct epitem {
    union {
        /* RB tree node links this structure to the eventpoll RB tree */
        struct rb_node rbn;
        /* Used to free the struct epitem */
        struct rcu_head rcu;
    };

    /* List header used to link this structure to the eventpoll ready list */
    //将这个epitem连接到eventpoll 里面的rdllist的list指针
    struct list_head rdllink;

    /*
     * Works together "struct eventpoll"->ovflist in keeping the
     * single linked chain of items.
     */
    struct epitem *next;

    /* The file descriptor information this item refers to */
    //epoll监听的fd
    struct epoll_filefd ffd;

    /* Number of active wait queue attached to poll operations */
    //一个文件可以被多个epoll实例所监听，这里就记录了当前文件被监听的次数
    int nwait;

    /* List containing poll wait queues */
    struct list_head pwqlist;

    /* The "container" of this item */
    //当前epollitem所属的eventpoll
    struct eventpoll *ep;

    /* List header used to link this item to the "struct file" items list */
    struct list_head fllink;

    /* wakeup_source used when EPOLLWAKEUP is set */
    struct wakeup_source __rcu *ws;

    /* The structure that describe the interested events and the source fd */
    struct epoll_event event;
};
~~~

每次当一个 fd 关联到一个 epoll 实例，就会有一个 eppoll_entry 产生。eppoll_entry 的结构如下：

~~~LINUX

/* Wait structure used by the poll hooks */
struct eppoll_entry {
    /* List header used to link this structure to the "struct epitem" */
    struct list_head llink;

    /* The "base" pointer is set to the container "struct epitem" */
    struct epitem *base;

    /*
     * Wait queue item that will be linked to the target file wait
     * queue head.
     */
    wait_queue_entry_t wait;

    /* The wait queue head that linked the "wait" wait queue item */
    wait_queue_head_t *whead;
};
~~~

### epoll_create

我们在使用 epoll 的时候，首先会调用 epoll_create 来创建一个 epoll 实例。这个函数是如何工作的呢?

首先，epoll_create 会对传入的 flags 参数做简单的验证。

~~~LINUX

/* Check the EPOLL_* constant for consistency.  */
BUILD_BUG_ON(EPOLL_CLOEXEC != O_CLOEXEC);

if (flags & ~EPOLL_CLOEXEC)
    return -EINVAL;
/*
~~~

接下来，内核申请分配 eventpoll 需要的内存空间。

~~~LINUX

/* Create the internal data structure ("struct eventpoll").
*/
error = ep_alloc(&ep);
if (error < 0)
  return error;
~~~

在接下来，epoll_create 为 epoll 实例分配了匿名文件和文件描述字，其中 fd 是文件描述字，file 是一个匿名文件。这里充分体现了 UNIX 下一切都是文件的思想。注意，eventpoll 的实例会保存一份匿名文件的引用，通过调用 fd_install 函数将匿名文件和文件描述字完成了绑定。

这里还有一个特别需要注意的地方，在调用 anon_inode_get_file 的时候，epoll_create 将 eventpoll 作为匿名文件 file 的 private_data 保存了起来，这样，在之后通过 epoll 实例的文件描述字来查找时，就可以快速地定位到 eventpoll 对象了。

最后，这个文件描述字作为 epoll 的文件句柄，被返回给 epoll_create 的调用者。

~~~LINUX

/*
 * Creates all the items needed to setup an eventpoll file. That is,
 * a file structure and a free file descriptor.
 */
fd = get_unused_fd_flags(O_RDWR | (flags & O_CLOEXEC));
if (fd < 0) {
    error = fd;
    goto out_free_ep;
}
file = anon_inode_getfile("[eventpoll]", &eventpoll_fops, ep,
             O_RDWR | (flags & O_CLOEXEC));
if (IS_ERR(file)) {
    error = PTR_ERR(file);
    goto out_free_fd;
}
ep->file = file;
fd_install(fd, file);
return fd;
~~~

### epoll_ctl

接下来，我们看一下一个套接字是如何被添加到 epoll 实例中的。这就要解析一下 epoll_ctl 函数实现了。

### 查找 epoll 实例

首先，epoll_ctl 函数通过 epoll 实例句柄来获得对应的匿名文件，这一点很好理解，UNIX 下一切都是文件，epoll 的实例也是一个匿名文件。

~~~lOINUX

//获得epoll实例对应的匿名文件
f = fdget(epfd);
if (!f.file)
    goto error_return;
~~~

接下来，获得添加的套接字对应的文件，这里 tf 表示的是 target file，即待处理的目标文件。

~~~LINUX

/* Get the "struct file *" for the target file */
//获得真正的文件，如监听套接字、读写套接字
tf = fdget(fd);
if (!tf.file)
    goto error_fput;
~~~

再接下来，进行了一系列的数据验证，以保证用户传入的参数是合法的，比如 epfd 真的是一个 epoll 实例句柄，而不是一个普通文件描述符。

~~~LINUX

/* The target file descriptor must support poll */
//如果不支持poll，那么该文件描述字是无效的
error = -EPERM;
if (!tf.file->f_op->poll)
    goto error_tgt_fput;
...
~~~

如果获得了一个真正的 epoll 实例句柄，就可以通过 private_data 获取之前创建的 eventpoll 实例了。

~~~LINUX

/*
 * At this point it is safe to assume that the "private_data" contains
 * our own data structure.
 */
ep = f.file->private_data;
~~~

### 红黑树查找

接下来 epoll_ctl 通过目标文件和对应描述字，在红黑树中查找是否存在该套接字，这也是 epoll 为什么高效的地方。红黑树（RB-tree）是一种常见的数据结构，这里 eventpoll 通过红黑树跟踪了当前监听的所有文件描述字，而这棵树的根就保存在 eventpoll 数据结构中。

~~~LINUX

/* RB tree root used to store monitored fd structs */
struct rb_root_cached rbr;
~~~

对于每个被监听的文件描述字，都有一个对应的 epitem 与之对应，epitem 作为红黑树中的节点就保存在红黑树中。

~~~LINUX

/*
 * Try to lookup the file inside our RB tree, Since we grabbed "mtx"
 * above, we can be sure to be able to use the item looked up by
 * ep_find() till we release the mutex.
 */
epi = ep_find(ep, tf.file, fd);
~~~

红黑树是一棵二叉树，作为二叉树上的节点，epitem 必须提供比较能力，以便可以按大小顺序构建出一棵有序的二叉树。其排序能力是依靠 epoll_filefd 结构体来完成的，epoll_filefd 可以简单理解为需要监听的文件描述字，它对应到二叉树上的节点。

可以看到这个还是比较好理解的，按照文件的地址大小排序。如果两个相同，就按照文件文件描述字来排序。

~~~LINUX

struct epoll_filefd {
  struct file *file; // pointer to the target file struct corresponding to the fd
  int fd; // target file descriptor number
} __packed;

/* Compare RB tree keys */
static inline int ep_cmp_ffd(struct epoll_filefd *p1,
                            struct epoll_filefd *p2)
{
  return (p1->file > p2->file ? +1:
       (p1->file < p2->file ? -1 : p1->fd - p2->fd));
}
~~~

在进行完红黑树查找之后，如果发现是一个 ADD 操作，并且在树中没有找到对应的二叉树节点，就会调用 ep_insert 进行二叉树节点的增加。

~~~LINUX

case EPOLL_CTL_ADD:
    if (!epi) {
        epds.events |= POLLERR | POLLHUP;
        error = ep_insert(ep, &epds, tf.file, fd, full_check);
    } else
        error = -EEXIST;
    if (full_check)
        clear_tfile_check_list();
    break;
~~~

### ep_insert

ep_insert 首先判断当前监控的文件值是否超过了 /proc/sys/fs/epoll/max_user_watches 的预设最大值，如果超过了则直接返回错误。

~~~LINUX

user_watches = atomic_long_read(&ep->user->epoll_watches);
if (unlikely(user_watches >= max_user_watches))
    return -ENOSPC;
~~~

接下来是分配资源和初始化动作。

~~~LINUX

if (!(epi = kmem_cache_alloc(epi_cache, GFP_KERNEL)))
        return -ENOMEM;

    /* Item initialization follow here ... */
    INIT_LIST_HEAD(&epi->rdllink);
    INIT_LIST_HEAD(&epi->fllink);
    INIT_LIST_HEAD(&epi->pwqlist);
    epi->ep = ep;
    ep_set_ffd(&epi->ffd, tfile, fd);
    epi->event = *event;
    epi->nwait = 0;
    epi->next = EP_UNACTIVE_PTR;
~~~

**再接下来的事情非常重要，ep_insert 会为加入的每个文件描述字设置回调函数。这个回调函数是通过函数 ep_ptable_queue_proc 来进行设置的。这个回调函数是干什么的呢？其实，对应的文件描述字上如果有事件发生，就会调用这个函数，比如套接字缓冲区有数据了，就会回调这个函数。这个函数就是 ep_poll_callback。这里你会发现，原来内核设计也是充满了事件回调的原理。**

~~~LINUX

/*
 * This is the callback that is used to add our wait queue to the
 * target file wakeup lists.
 */
static void ep_ptable_queue_proc(struct file *file, wait_queue_head_t *whead,poll_table *pt)
{
    struct epitem *epi = ep_item_from_epqueue(pt);
    struct eppoll_entry *pwq;

    if (epi>nwait >= 0 && (pwq = kmem_cache_alloc(pwq_cache, GFP_KERNEL))) {
        init_waitqueue_func_entry(&pwq->wait, ep_poll_callback);
        pwq->whead = whead;
        pwq->base = epi;
        if (epi->event.events & EPOLLEXCLUSIVE)
            add_wait_queue_exclusive(whead, &pwq->wait);
        else
            add_wait_queue(whead, &pwq->wait);
        list_add_tail(&pwq->llink, &epi->pwqlist);
        epi->nwait++;
    } else {
        /* We have to signal that an error occurred */
        epi->nwait = -1;
    }
}
~~~

### ep_poll_callback

**ep_poll_callback 函数的作用非常重要，它将内核事件真正地和 epoll 对象联系了起来**。它又是怎么实现的呢

首先，通过这个文件的 wait_queue_entry_t 对象找到对应的 epitem 对象，因为 eppoll_entry 对象里保存了 wait_quue_entry_t，根据 wait_quue_entry_t 这个对象的地址就可以简单计算出 eppoll_entry 对象的地址，从而可以获得 epitem 对象的地址。这部分工作在 ep_item_from_wait 函数中完成。一旦获得 epitem 对象，就可以寻迹找到 eventpoll 实例。

~~~LINUX

/*
 * This is the callback that is passed to the wait queue wakeup
 * mechanism. It is called by the stored file descriptors when they
 * have events to report.
 */
static int ep_poll_callback(wait_queue_entry_t *wait, unsigned mode, int sync, void *key)
{
    int pwake = 0;
    unsigned long flags;
    struct epitem *epi = ep_item_from_wait(wait);
    struct eventpoll *ep = epi->ep;
~~~

接下来，进行一个加锁操作。

~~~LINUX

spin_lock_irqsave(&ep->lock, flags);
~~~

**下面对发生的事件进行过滤，为什么需要过滤呢？为了性能考虑，ep_insert 向对应监控文件注册的是所有的事件，而实际用户侧订阅的事件未必和内核事件对应。比如，用户向内核订阅了一个套接字的可读事件，在某个时刻套接字的可写事件发生时，并不需要向用户空间传递这个事件。**

~~~LINUX

/*
 * Check the events coming with the callback. At this stage, not
 * every device reports the events in the "key" parameter of the
 * callback. We need to be able to handle both cases here, hence the
 * test for "key" != NULL before the event match test.
 */
if (key && !((unsigned long) key & epi->event.events))
    goto out_unlock;
~~~

接下来，判断是否需要把该事件传递给用户空间。

~~~LINUX

if (unlikely(ep->ovflist != EP_UNACTIVE_PTR)) {
  if (epi->next == EP_UNACTIVE_PTR) {
      epi->next = ep->ovflist;
      ep->ovflist = epi;
      if (epi->ws) {
          /*
           * Activate ep->ws since epi->ws may get
           * deactivated at any time.
           */
          __pm_stay_awake(ep->ws);
      }
  }
  goto out_unlock;
}
~~~

如果需要，而且该事件对应的 event_item 不在 eventpoll 对应的已完成队列中，就把它放入该队列，以便将该事件传递给用户空间。

~~~LINUX

/* If this file is already in the ready list we exit soon */
if (!ep_is_linked(&epi->rdllink)) {
    list_add_tail(&epi->rdllink, &ep->rdllist);
    ep_pm_stay_awake_rcu(epi);
}
~~~

我们知道，当我们调用 epoll_wait 的时候，调用进程被挂起，在内核看来调用进程陷入休眠。如果该 epoll 实例上对应描述字有事件发生，这个休眠进程应该被唤醒，以便及时处理事件。下面的代码就是起这个作用的，wake_up_locked 函数唤醒当前 eventpoll 上的等待进程。

~~~LINUX 

/*
 * Wake up ( if active ) both the eventpoll wait list and the ->poll()
 * wait list.
 */
if (waitqueue_active(&ep->wq)) {
    if ((epi->event.events & EPOLLEXCLUSIVE) &&
                !((unsigned long)key & POLLFREE)) {
        switch ((unsigned long)key & EPOLLINOUT_BITS) {
        case POLLIN:
            if (epi->event.events & POLLIN)
                ewake = 1;
            break;
        case POLLOUT:
            if (epi->event.events & POLLOUT)
                ewake = 1;
            break;
        case 0:
            ewake = 1;
            break;
        }
    }
    wake_up_locked(&ep->wq);
}
~~~



### 查找 epoll 实例

epoll_wait 函数首先进行一系列的检查，例如传入的 maxevents 应该大于 0。

~~~LINUX

/* The maximum number of event must be greater than zero */
if (maxevents <= 0 || maxevents > EP_MAX_EVENTS)
    return -EINVAL;

/* Verify that the area passed by the user is writeable */
if (!access_ok(VERIFY_WRITE, events, maxevents * sizeof(struct epoll_event)))
    return -EFAULT;
~~~

和前面介绍的 epoll_ctl 一样，通过 epoll 实例找到对应的匿名文件和描述字，并且进行检查和验证。

~~~LINUX

/* Get the "struct file *" for the eventpoll file */
f = fdget(epfd);
if (!f.file)
    return -EBADF;

/*
 * We have to check that the file structure underneath the fd
 * the user passed to us _is_ an eventpoll file.
 */
error = -EINVAL;
if (!is_file_epoll(f.file))
    goto error_fput;
~~~

还是通过读取 epoll 实例对应匿名文件的 private_data 得到 eventpoll 实例。

~~~LINUX 

/*
 * At this point it is safe to assume that the "private_data" contains
 * our own data structure.
 */
ep = f.file->private_data;
~~~

接下来调用 ep_poll 来完成对应的事件收集并传递到用户空间。

~~~lINUX

/* Time to fish for events ... */
error = ep_poll(ep, events, maxevents, timeout);
~~~



### ep_poll

还记得第 23 讲里介绍 epoll 函数的时候，对应的 timeout 值可以是大于 0，等于 0 和小于 0 么？这里 ep_poll 就分别对 timeout 不同值的场景进行了处理。如果大于 0 则产生了一个超时时间，如果等于 0 则立即检查是否有事件发生。

~~~LINUX

*/
static int ep_poll(struct eventpoll *ep, struct epoll_event __user *events,int maxevents, long timeout)
{
int res = 0, eavail, timed_out = 0;
unsigned long flags;
u64 slack = 0;
wait_queue_entry_t wait;
ktime_t expires, *to = NULL;

if (timeout > 0) {
    struct timespec64 end_time = ep_set_mstimeout(timeout);
    slack = select_estimate_accuracy(&end_time);
    to = &expires;
    *to = timespec64_to_ktime(end_time);
} else if (timeout == 0) {
    /*
     * Avoid the unnecessary trip to the wait queue loop, if the
     * caller specified a non blocking operation.
     */
    timed_out = 1;
    spin_lock_irqsave(&ep->lock, flags);
    goto check_events;
}
~~~

接下来尝试获得 eventpoll 上的锁：

~~~LINUX

spin_lock_irqsave(&ep->lock, flags);
~~~

获得这把锁之后，检查当前是否有事件发生，如果没有，就把当前进程加入到 eventpoll 的等待队列 wq 中，这样做的目的是当事件发生时，ep_poll_callback 函数可以把该等待进程唤醒。

~~~LINUX

if (!ep_events_available(ep)) {
    /*
     * Busy poll timed out.  Drop NAPI ID for now, we can add
     * it back in when we have moved a socket with a valid NAPI
     * ID onto the ready list.
     */
    ep_reset_busy_poll_napi_id(ep);

    /*
     * We don't have any available event to return to the caller.
     * We need to sleep here, and we will be wake up by
     * ep_poll_callback() when events will become available.
     */
    init_waitqueue_entry(&wait, current);
    __add_wait_queue_exclusive(&ep->wq, &wait);
~~~

**紧接着是一个无限循环, 这个循环中通过调用 schedule_hrtimeout_range，将当前进程陷入休眠，CPU 时间被调度器调度给其他进程使用，当然，当前进程可能会被唤醒，唤醒的条件包括有以下四种：**

1. 当前进程超时；
2. 当前进程收到一个 signal 信号；
3. 某个描述字上有事件发生；
4. 当前进程被 CPU 重新调度，进入 for 循环重新判断，如果没有满足前三个条件，就又重新进入休眠。

对应的 1、2、3 都会通过 break 跳出循环，直接返回。

~~~LINUX

//这个循环里，当前进程可能会被唤醒，唤醒的途径包括
//1.当前进程超时
//2.当前进行收到一个signal信号
//3.某个描述字上有事件发生
//对应的1.2.3都会通过break跳出循环
//第4个可能是当前进程被CPU重新调度，进入for循环的判断，如果没有满足1.2.3的条件，就又重新进入休眠
for (;;) {
    /*
     * We don't want to sleep if the ep_poll_callback() sends us
     * a wakeup in between. That's why we set the task state
     * to TASK_INTERRUPTIBLE before doing the checks.
     */
    set_current_state(TASK_INTERRUPTIBLE);
    /*
     * Always short-circuit for fatal signals to allow
     * threads to make a timely exit without the chance of
     * finding more events available and fetching
     * repeatedly.
     */
    if (fatal_signal_pending(current)) {
        res = -EINTR;
        break;
    }
    if (ep_events_available(ep) || timed_out)
        break;
    if (signal_pending(current)) {
        res = -EINTR;
        break;
    }

    spin_unlock_irqrestore(&ep->lock, flags);

    //通过调用schedule_hrtimeout_range，当前进程进入休眠，CPU时间被调度器调度给其他进程使用
    if (!schedule_hrtimeout_range(to, slack, HRTIMER_MODE_ABS))
        timed_out = 1;

    spin_lock_irqsave(&ep->lock, flags);
}
~~~

如果进程从休眠中返回，则将当前进程从 eventpoll 的等待队列中删除，并且设置当前进程为 TASK_RUNNING 状态。

~~~LINUX

//从休眠中结束，将当前进程从wait队列中删除，设置状态为TASK_RUNNING，接下来进入check_events，来判断是否是有事件发生
    __remove_wait_queue(&ep->wq, &wait);
    __set_current_state(TASK_RUNNING);
~~~

最后，调用 ep_send_events 将事件拷贝到用户空间。

~~~LINUX

//ep_send_events将事件拷贝到用户空间
/*
 * Try to transfer events to user space. In case we get 0 events and
 * there's still timeout left over, we go trying again in search of
 * more luck.
 */
if (!res && eavail &&
    !(res = ep_send_events(ep, events, maxevents)) && !timed_out)
    goto fetch_events;


return res;
~~~

### ep_send_events

ep_send_events 这个函数会将 ep_send_events_proc 作为回调函数并调用 ep_scan_ready_list 函数，ep_scan_ep_send_events_proc 循环处理就绪事件时，会再次调用每个文件描述符的 poll 方法，以便确定确实有事件发生。为什么这样做呢？这是为了确定注册的事件在这个时刻还是有效的。

ep_send_events_proc 循环处理就绪事件时，会再次调用每个文件描述符的 poll 方法，以便确定确实有事件发生。为什么这样做呢？这是为了确定注册的事件在这个时刻还是有效的。

可以看到，尽管 ep_send_events_proc 已经尽可能的考虑周全，使得用户空间获得的事件通知都是真实有效的，但还是有一定的概率，当 ep_send_events_proc 再次调用文件上的 poll 函数之后，用户空间获得的事件通知已经不再有效，这可能是用户空间已经处理掉了，或者其他什么情形。还记得第 22 讲吗，在这种情况下，**如果套接字不是非阻塞的，整个进程将会被阻塞，这也是为什么将非阻塞套接字配合 epoll 使用作为最佳实践的原因**。

在进行简单的事件掩码校验之后，ep_send_events_proc 将事件结构体拷贝到用户空间需要的数据结构中。这是通过 __put_user 方法完成的。

~~~LINUX

//这里对一个fd再次进行poll操作，以确认事件
revents = ep_item_poll(epi, &pt);

/*
 * If the event mask intersect the caller-requested one,
 * deliver the event to userspace. Again, ep_scan_ready_list()
 * is holding "mtx", so no operations coming from userspace
 * can change the item.
 */
if (revents) {
    if (__put_user(revents, &uevent->events) ||
        __put_user(epi->event.data, &uevent->data)) {
        list_add(&epi->rdllink, head);
        ep_pm_stay_awake(epi);
        return eventcnt ? eventcnt : -EFAULT;
    }
    eventcnt++;
    uevent++;
~~~



### Level-triggered VS Edge-triggered

在前面的文章里，我们一直都在强调 level-triggered 和 edge-triggered 之间的区别。

从实现角度来看其实非常简单，在 ep_send_events_proc 函数的最后，针对 level-triggered 情况，当前的 epoll_item 对象被重新加到 eventpoll 的就绪列表中，这样在下一次 epoll_wait 调用时，这些 epoll_item 对象就会被重新处理。

在前面我们提到，**在最终拷贝到用户空间有效事件列表中之前，会调用对应文件的 poll 方法，以确定这个事件是不是依然有效。所以，如果用户空间程序已经处理掉该事件，就不会被再次通知；如果没有处理，意味着该事件依然有效，就会被再次通知。**

~~~LINUX

//这里是Level-triggered的处理，可以看到，在Level-triggered的情况下，这个事件被重新加回到ready list里面
//这样，下一轮epoll_wait的时候，这个事件会被重新check
else if (!(epi->event.events & EPOLLET)) {
    /*
     * If this file has been added with Level
     * Trigger mode, we need to insert back inside
     * the ready list, so that the next call to
     * epoll_wait() will check again the events
     * availability. At this point, no one can insert
     * into ep->rdllist besides us. The epoll_ctl()
     * callers are locked out by
     * ep_scan_ready_list() holding "mtx" and the
     * poll callback will queue them in ep->ovflist.
     */
    list_add_tail(&epi->rdllink, &ep->rdllist);
    ep_pm_stay_awake(epi);
}
~~~



### epoll VS poll/select

最后，我们从实现角度来说明一下为什么 epoll 的效率要远远高于 poll/select。

首先，**poll/select 先将要监听的 fd 从用户空间拷贝到内核空间, 然后在内核空间里面进行处理之后，再拷贝给用户空间。这里就涉及到内核空间申请内存，释放内存等等过程，这在大量 fd 情况下，是非常耗时的。而 epoll 维护了一个红黑树，通过对这棵黑红树进行操作，可以避免大量的内存申请和释放的操作，而且查找速度非常快**。

下面的代码就是 poll/select 在内核空间申请内存的展示。可以看到 select 是先尝试申请栈上资源, 如果需要监听的 fd 比较多, 就会去申请堆空间的资源。

~~~LINUX

int core_sys_select(int n, fd_set __user *inp, fd_set __user *outp,
               fd_set __user *exp, struct timespec64 *end_time)
{
    fd_set_bits fds;
    void *bits;
    int ret, max_fds;
    size_t size, alloc_size;
    struct fdtable *fdt;
    /* Allocate small arguments on the stack to save memory and be faster */
    long stack_fds[SELECT_STACK_ALLOC/sizeof(long)];

    ret = -EINVAL;
    if (n < 0)
        goto out_nofds;

    /* max_fds can increase, so grab it once to avoid race */
    rcu_read_lock();
    fdt = files_fdtable(current->files);
    max_fds = fdt->max_fds;
    rcu_read_unlock();
    if (n > max_fds)
        n = max_fds;

    /*
     * We need 6 bitmaps (in/out/ex for both incoming and outgoing),
     * since we used fdset we need to allocate memory in units of
     * long-words. 
     */
    size = FDS_BYTES(n);
    bits = stack_fds;
    if (size > sizeof(stack_fds) / 6) {
        /* Not enough space in on-stack array; must use kmalloc */
        ret = -ENOMEM;
        if (size > (SIZE_MAX / 6))
            goto out_nofds;


        alloc_size = 6 * size;
        bits = kvmalloc(alloc_size, GFP_KERNEL);
        if (!bits)
            goto out_nofds;
    }
    fds.in      = bits;
    fds.out     = bits +   size;
    fds.ex      = bits + 2*size;
    fds.res_in  = bits + 3*size;
    fds.res_out = bits + 4*size;
    fds.res_ex  = bits + 5*size;
    ...
~~~

**第二，select/poll 从休眠中被唤醒时，如果监听多个 fd，只要其中有一个 fd 有事件发生，内核就会遍历内部的 list 去检查到底是哪一个事件到达，并没有像 epoll 一样, 通过 fd 直接关联 eventpoll 对象，快速地把 fd 直接加入到 eventpoll 的就绪列表中。**

~~~LINUX

static int do_select(int n, fd_set_bits *fds, struct timespec64 *end_time)
{
    ...
    retval = 0;
    for (;;) {
        unsigned long *rinp, *routp, *rexp, *inp, *outp, *exp;
        bool can_busy_loop = false;

        inp = fds->in; outp = fds->out; exp = fds->ex;
        rinp = fds->res_in; routp = fds->res_out; rexp = fds->res_ex;

        for (i = 0; i < n; ++rinp, ++routp, ++rexp) {
            unsigned long in, out, ex, all_bits, bit = 1, mask, j;
            unsigned long res_in = 0, res_out = 0, res_ex = 0;

            in = *inp++; out = *outp++; ex = *exp++;
            all_bits = in | out | ex;
            if (all_bits == 0) {
                i += BITS_PER_LONG;
                continue;
            }
        
        if (!poll_schedule_timeout(&table, TASK_INTERRUPTIBLE,
                   to, slack))
        timed_out = 1;
...
~~~

### 总结

在这次答疑中，我希望通过深度分析 epoll 的源码实现，帮你理解 epoll 的实现原理。

**epoll 维护了一棵红黑树来跟踪所有待检测的文件描述字，黑红树的使用减少了内核和用户空间大量的数据拷贝和内存分配，大大提高了性能。**

**同时，epoll 维护了一个链表来记录就绪事件，内核在每个文件有事件发生时将自己登记到这个就绪事件列表中，通过内核自身的文件 file-eventpoll 之间的回调和唤醒机制，减少了对内核描述字的遍历，大大加速了事件通知和检测的效率，这也为 level-triggered 和 edge-triggered 的实现带来了便利。**

**通过对比 poll/select 的实现，我们发现 epoll 确实克服了 poll/select 的种种弊端，不愧是 Linux 下高性能网络编程的皇冠。我们应该感谢 Linux 社区的大神们设计了这么强大的事件分发机制，让我们在 Linux 下可以享受高性能网络服务器带来的种种技术红利。**