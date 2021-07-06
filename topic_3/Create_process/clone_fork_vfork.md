# clone()、fork() 及 vfork() 系统调用

在 Linux 中，轻量级进程是由名为 `clone()` 的函数创建的，这个函数使用下列参数：

* *fn*  
指定一个由新进程执行的函数。当这个函数返回时，子进程终止。函数返回一个整数，表示子进程的退出代码。  
&emsp;

* *arg*  
指向传递给 `fn()` 函数的数据。  
&emsp;

* *flags*  
各种各样的信息。低字节指定子进程结束时发送到父进程的信号代码，通常选择 SIGCHLD 信号。剩余的 3 个字节给一 clone 标志组用于编码，如表 3-8 所示。  
&emsp;

* *child_stack*  
表示把用户态堆栈指针赋给子进程的 esp 寄存器。调用进程（指调用 `clone()` 的父进程）应该总是为子进程分配新的堆栈。  
&emsp;

* *tls*  
表示线程局部存储段（TLS）数据结构的地址，该结构是为新轻量级进程定义的（参见第二章 “Linux GDT” 一节）。只有在 CLONE_SETTLS 标志被设置时才有意义。  
&emsp;

* *ptid*  
表示父进程的用户态变量地址，该父进程具有与新轻量级进程箱同的 PID。只有在 CLONE_PARENT_SETTID 标志被设置时才有意义。  
&emsp;

* *ctid*  
表示新轻量级进程的用户态变量地址，该进程具有这一类进程的 PID。只有在 CLONE_CHILD_SETTID 标志被设置时才有意义。
&emsp;

**表 3-8：clone 标志**
标志名称 | 说明
--- | ---
CLONE_VM | 共享内存描述符和所有的页表（参见第九章）
CLONE_FS | 共享根目录和当前工作目录所在的表，以及用于屏蔽新文件初始许可权的位掩码值（所谓文件的 umask）
CLONE_FILES | 共享打开文件表（参见第十二章）
CLONE_SIGHAND | 共享信号处理程序的表、阻塞信号表和挂起信号表（参见第十一章）。如果这个标志为 true，就必须设置 CLONE_VM 标志
CLONE_PTRACE | 如果父进程被跟踪，那么，子进程也被跟踪。无其是 debugger 程序可能希望以自己作为父进程来跟踪子进程，在这种情况下，内核把该标志强置为 I
CLONE_VFORK | 在发出 `vfork()` 系统调用时设置（参见本节后面）
CLONE_PARENT | 设置子进程的父进程（进程描述符中的 parent 和 real_parent字段）为调用进程的父进程
CLONE_THREAD | 把子进程插入到父进程的同一线程组中，并迫使子进程共享交进程的信号描述符。因此也设置子进程的 tgid 字段和 group_leader 字段。如果这个标志位为 true，就必须设置 CLONE_SIGHAND 标志
CLONE_NEWNS | 当 clone 需要自己的命名空间时（即它自己的已挂载文件系统视图）设置这个标志（参见第十二章）。不能同时设置 CLONE_NEWNS 和 CLONE_FS
CLONE_SYSVSEM | 共享 System V IPC 取消信号量的操作（参见第十九章 “IPC 信号量” 一节）
CLONE_SETTLS | 为轻量级进程创建新的线程局部存储段（TLS），该段由参数 tls 所指向的结构进行描述
CLONE_PARENT_SETTID | 把子进程的 PID 写入由 ptid 参数所指向的父进程的用户态变量
CLONE_CHILD_CLEARTID | 如果该标志被设置，则内核建立一种触发机制，用在子进程要退出或要开始执行新程序时。在这些情况下，内核将清除由参数 ctid 所指向的用户态变量，并唤醒等待这个事件的任何进程
CLONE_DETACHED | 遗留标志，内核会忽略它
CLONE_UNTRACED | 内核设置这个标志以使 CLONE_PTRACE 标志失去作用（用来禁止内核线程跟踪进程，参见本章稍后的 “内核线程” 一节）
CLONE_CHILD_SETTID | 把子进程的 PID 写入由 ctid 参数所指向的子进程的用户态变量中
CLONE_STOPPED | 强迫子进程开始于 TASK_STOPPED 状态

实际上，`clone()` 是在 C 语言库中定义的一个封装（wrapper）函数（参见第十章 “POSIX API 和系统调用” 一节），它负责建立新轻量级进程的堆栈并且调用对编程者隐藏的 `clone()` 系统调用。实现 `clone()` 系统调用的 `sys_clone()` 服务例程没有 fn 和 arg 参数。实际上，封装函数把 fn 指针存放在子进程堆栈的某个位置处，该位置就是该封装函数本身返回地址存放的位置。arg 指针正好存放在子进程堆栈中 fn 的下面。当封装函数结束时，CPU 从堆栈中取出返回地址，然后执行 `fn(arg)` 函数。

传统的 `fork()` 系统调用在 Linux 中是用 `clone()` 实现的，其中 `clone()` 的 flags 参数指定为 SIGCHLD 信号及所有清 0 的 clone 标志，而它的 child_stack 参数是父进程当前的堆栈指针。因此，父进程和子进程暂时共享同一个用户态堆栈。但是，要感谢写时复制机制，通常只要交子进程中有一个试图去改变栈，则立即各自得到用户态堆栈的一份拷贝。

前一节描述的 `vfork()` 系统调用在 Linux 中也是用 `clone()` 实现的，其中 `clone()` 的参数 flags 指定为 SIGCHLD 信号和 CLONE_VM 及 CLONE_VFORK 标志，`clone()` 的
参数 child_stack 等于父进程当前的栈指针。

