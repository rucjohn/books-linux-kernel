# 如何组织进程

运行队列链表把处于 TASK_RUNNING 状态的所有进程组织在一起。当要把其他状态的进程分组时，不同的状态要求不同的处理，Linux 选择了下列方式之一：

- 没有为处于 TASK_STOPPED、EXIT_ZOMBIE 或 EXIT_DEAD 状态的进程建立专门的链表。由于对处于暂停、僵死、死亡状态进程的访问比较简单，或者通过 PID， 或者通过特定父进程的子进程链表，所以不必对这三种状态进程分组。
- 没有为处于状态的


## 等待队列

等待队列在内核中有很多用途，尤其用在中断处理、进程同步及定时。因为这些主题将在以后的章节中讨论，所以我们只在这里说明，进程必须经常等待某些事件的发生，例如，等待一个磁盘操作的终止，等待释放系统资源，或等待时间经过固定的间隔。

等待队列实现了在事件上的条件等待：希望等待特定事件的进程把自己放进合适的等待队列，并放弃控制权。因此，**等待队列表示一组睡眠的进程，当某一条件变为真时，由内核唤醒它们**。

等待队列由双向链表实现，其元素包括指向进程描述符的指针。每个等待队列都有一个等待队列头（*wait queue head*），等待队列头是一个类型为 wait_queue_head_t 的数据结构：
```
struct _ _wait_queue_head {
    spinlock_t lock;
    struct list_head task_list;
};
typedef struct _ _wait_queue_head wait_queue_head_t;
```

因为等待队列是由中断处理程序和主要内核函数修改的，因此必须对其双向链表进行保护以免对其进行同时访问，因为同时访问会导致不可预测的后果（参见第五章）。同步是通过等待队列头中的 lock 自旋锁达到的。task_list 字段是等待进程链表的头。

等待队列链表中的元素类型为 wait_queue_t：
```
struct _ _wait_queue {
    unsigned int flags;
    struct task_struct * task;
    wait_queue_func_t func;
    struct list_head task_list;
};
typedef struct _ _wait_queue wait_queue_t;
```

等待队列链表中的每个元素代表一个睡眠进程，该进程等待某一事件的发生；它的描述符地址存放在 task 字段中。task_list 字段中包含的是指针，由这个指针把一个元素链接到等待相同事件的进程链表中。

然而，要唤醒等待队列中所有睡眠的进程有时并不方便。例如，如果两个或多个进程正在等待互斥访问某一要释放的资源，仅唤醒等待队列中的一个进程才有意义。这个进程占有资源，而其他进程继续睡眠。（这就避免了所谓 “雷鸣般兽群” 问题，即唤醒多个进程只为了竞争一个资源，而这个资源只能有一个进程访问，结果是其他进程必须再次回去睡眠。）

因此，有两种睡眠进程：互斥进程（等待队列元素的 flags 字段为 1）由内核有选择地唤醒，而非互斥进程（flags 为 0）总是由内核在事件发生时唤醒。等待访问临界资源的进程就是互斥进程的典型例子。等待相关事件的进程是非互斥的。例如，我们考虑等待磁盘传输结束的一组进程：一但磁盘传输完成，所有等待的进程都会被唤醒。正如我们将在下面所看到的那样，等待队列元素的 func 字段用来表示等待队列中睡眠进程应该用什么方式唤醒。

## 等待队列的操作

可以用 `DECLARE_WAIT_QUEUE_HEAD(name)` 宏定义一个新等待队列的头，它静态地声明一个叫 name 的等待队列的头变量并对该变量的 lock 和 task_list 字段进行初始化。函数 `init_waitqueue_head()` 可以用来初始化动态分配的等待队列的头变量。

函数 `init_waitqueue_entry(q,p)` 如下所示，初始化 wait_queue_t 结构的变量 q：
```
q>flags = 0;
q>task = p;
q>func = default_wake_function;
```

非互斥进程 p 将由 `default_wake_function()` 唤醒，`default_wake_function()` 是在第七章中要讨论的 `try_to_wake_up()` 函数的一个简单的封装。

也可以选择 `DEFAULT_WAIT` 宏声明一个 wait_queue_t 类型的新变量，并且 CPU 上运行的当前进程的描述符和唤醒函数 `autoremove_wake_function()` 的地址初始化这个新变量。这个函数调用 `default_wake_function()` 来唤醒睡眠进程，然后从等待队列的链表中删除对应的元素（每个等待队列链表中的一个元素其实就是指向睡眠进程描述符的指针）。最后，内核开发者可以通过 `init_waitqueue_func_entry()` 函数来自定义唤醒函数，该函数负责初始化等待队列的元素。

一旦定义了一个元素，必须把它插入等待队列。
- `add_wait_queue()` 函数把一个非互斥进程插入等待队列链表的第一个位置。
- `add_wait_queue_exclusive()` 函数把一个互斥进程插入等待队列链表的最后一个位置。
- `remove_wait_queue()` 函数从等待队列链表中删除一个进程。
- `waitqueue_active()` 函数检查 一个给定的等待队列是否为空。

要等待特定条件的进程可以调用如下列表中的任何一个函数：

* `sleep_on()` 对当前进程进行操作：  
```
void sleep_on(wait_queue_head_t *wq)
{
    wait_queue_t wait;
    init_waitqueue_entry(&wait, current);
    current->state = TASK_UNINTERRUPTIBLE;
    add_wait_queue(wq,&wait);
    schedule();
    remove_wait_queue(wq, &wait);
}
```
该函数把当前进程的状态设置为 TASK_UNINTERRUPTIBLE，并把它插入到特定的等待队列。然后，它调用调度程序，而调度程序重新开始另一个程序的执行。当睡眠进程被唤醒时，调度程序重新开始执行 `sleep_on()` 函数，把该进程从等待队列中删除。  
&emsp;

* `interruptible_sleep_on()` 与 `sleep_on()` 函数是一样的，但稍有不同，前者把当前进程的状态设置为 TASK_INTERRUPTIBLE 而不是 TASK_UNINTERRUPTIBLE，因此，接受一个信号就可以唤醒当前进程。  
&emsp;

* `sleep_on_timeout()` 和 `interruptible_sleep_on_timeout()` 与前面函数类似，但它们允许调用者定义一个时间间隔，过了这个间隔以后，进程将由内核唤醒。为了做到这点，它们调用 `schedule_timeout()` 函数而不是 `schedule()` 函数（参见第六章中 “动态定时器的应用” 一节）。  
&emsp;

* 在 Linux 2.6 中引入的 `prepare_to_wait()`、`prepare_to_wait_exclusive()` 和 `finish_wait()` 函数提供了另外一种途径来使当前进程在一个等待队列中睡眠。它们的典型应用如下：  
```
DEFINE_WAIT(wait);
prepare_to_wait_exclusive(&wq, &wait, TASK_INTERRUPTIBLE); /* wq 是等待队列的头 */
...
if (!codition)
    schedule();
finish_wait(&wq, &wait);

```
函数 `prepare_to_wait()` 和 `prepare_to_wait_exclusive()` 用传递的第三个参数设置进程的状态，然后把等待队列元素的互斥标志 flag 分别设置为 0（非互斥）或 1（互斥），最后，把等待元素 wait 插入到以 wq 为头的等待队列的链表中。  

进程一旦被唤醒就执行 `finish_wait()` 函数，它把进程的状态再次设置为 TASK_RUNNING（仅发生在调用 `schedule()` 之前，唤醒条件变为真的情况下），并从等待队列中删除等待元素（除非这个工作已经由唤醒函数完成）。  
&emsp;

* `wait_event` 和 `wait_event_interruptible` 宏使它们的调用进程在等待队列上睡眠，一直到修改了给定条件为止。例如，宏 `wait_event(wq,condition)` 本质上实现下面的功能：  
```
DEFINE_WAIT(_ _wait);
for (;;) {
    prepare_to_wait(&wq, &_ _wait, TASK_UNINTERRUPTIBLE);
    if (condition)
        break;
    schedule();
}
finish_wait(&wq, &_ _wait);
```
&emsp;

对上面列出的函数做一些说明：`sleep_on()` 类函数在以下条件下不能使用，那就是必须测试条件并且当条件还没有得到验证时又紧接着让进程去睡眠；由于那些条件是众所周知的竞争条件产生的根源，所以不鼓励这样使用。此外，为了把一个互斥进程插入到等待队列，内核必须使用 `prepare_to_wait_exclusive()` 函数（或者只是直接调用 `add_wait_queue_exclusive()`）。所有其他的相关函数把进程当作非互斥进程来插入。最后，除非使用 `DEFINE_WAIT` 或 `finish_wait()`，否则内核必须在唤醒等待进程后从等待队列中删除对应的等待队列元素。

内核通过下面的任何一个宏唤醒等待队列中的进程并把它们的状态置为 TASK_RUNNING：
- `wake_up`
- `wake_up_nr`
- `wake_up_all`
- `wake_up_interruptible`
- `wake_up_interruptible_nr`
- `wake_up_interruptible_all`
- `wake_up_interruptible_sync`
- `wake_up_locked`

从每个宏的名字我们可以明白功能：
- 所有宏都考虑到处于 TASK_INTERRUPTIBLE 状态的睡眠进程；如果宏的名字中不含有字符串 “interruptible”，那么处于 TASK_UNINTERRUPTIBLE 状态的睡眠进程也被考虑到。
- 所有宏都唤醒具有请求状态的所有非互斥进程（参见上一项）。
- 名字中含有 “nr” 字符串的宏唤醒给定数的具有有请求状态的互斥进程；这个数字是宏的一个参数。名字中含有 “all” 字符串的宏唤醒具有请求状态的所有互斥进程。最后，名字中不含有 “nr” 或 “all” 字符串的宏只唤醒具有请求状态的一个互斥进程。
- 名字中不含有 “sync” 字符串的宏检查被唤醒进程的优先级是否高于系统中正在运行进程的优先级，并在必要时调用 `schedule()`。这些检查并不是由名字含有 “sync” 字符串的宏进行的，造成的结果是高优先级进程的执行稍有延迟。
- `wake_up_locked` 和 `wake_up` 宏相类似，仅有的不同是当 `wait_queue_head_t` 中的自旋锁已经被持有时要调用 `wake_up_locked`。

例如，`wake_up` 宏等价于下列代码片段：
```
void wake_up(wait_queue_head_t *q)
{
    struct list_head *tmp;
    wait_queue_t *curr;

    list_for_each(tmp, &q->task_list) {
        curr = list_entry(tmp, wait_queue_t, task_list);
        if (curr->func(curr, TASK_INTERRUPTIBLE|TASK_UNINTERRUPTIBLE, 0, NULL) && curr->flags)
            break;
    }
}
```

`list_for_each` 宏扫描双向链表 `q->task_list` 中的所有项，即等待队列中的所有进程。对第一项，`list_entry` 宏都计算 wait_queue_t 变量对应的地址。这个变量的 func 字段存放唤醒函数的地址，它试图唤醒由等待队列元素的 task 字段标识的进程。如果一个进程已经被有效地唤醒（函数返回 1）并且进程是互斥的（`curr->flags == 1`），循环结束。因为所有的非互斥进程总是在双向链表的开始位置，而所有的互斥进程在双向链表的尾部，所以函数总是先唤醒非互斥进程然后再唤醒互斥进程，如果有进程存在的话。

> 顺便提一下，一个等待队列中同时包含互斥进程和非互斥进程的情况是非常罕见的。



