# 进程链表

我们首先介绍双向链表的第一个例子 --- 进程链表，进程链表把所有进程的描述符链表起来。每个 task_struct 结构都包含一个 list_head 类型的 tasks 字段，这个类型的 prev 和 next 字段分别指向前面和后面的 task_struct 元素。

进程链表的头是 init_task 描述符，它是所谓的 0 进程（*process 0*）或 *swapper* 进程的进程描述符（参见本章 “内核线程” 一节）。init_task 的 tasks.prev 字段指向链表中最后插入的进程描述符的 tasks 字段。

`SET_LINKS` 和 `REMOVE_LINKS` 宏分别用于从进程链表中插入和删除一个进程描述符。这些宏考虑了进程间的父子关系（见本章后面 “如何组织进程” 一节）。

还有一个很有用的宏就是 `for_each_process`，它的功能是扫描整个进程链表，其定义如下：
```
# define for_each_process(p) \
    for (p=&init_task; (p=list_entry((p)->tasks.next, \
                                     struct task_struct, tasks) \
                                    ) != &init_task; )
```

这个宏是循环控制语句，内核开发者利用它提供循环。注意 init_task 进程描述符是如何起到链表头作用的。这个宏从指向 init_task 的指针开始，把指针移到下一个任务，然后继续，直到又到 init_task 为止（感谢链表的循环性）。在每一次循环时，传递给这个宏的参变量中存放的是当前被扫描进程描述符的地址，这与 `list_entry` 宏的返回值一样。
