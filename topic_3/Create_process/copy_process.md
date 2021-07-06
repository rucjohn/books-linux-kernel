# copy_process() 函数

`copy_process()` 创建进程描述符以及子进程执行所需要的所有其他数据结构。它的参数与 `do_fork()` 的参数相同，外加子进程的 PID。下面描述 `copy_process()` 的最重要的步骤：

1. 检查参数 clone_flags 所传递标志的一致性。无其是，在下列情况下，它返回错误代号：  

    a. CLONE_NEWNS 和 CLONE_FS 标志都被设置。  
    &emsp;
    b. CLONE_THREAD 标志被设置，但 CLONE_SIGHAND 标志被清 0（同一线程组中的轻量级进程必须共享信号）。  
    &emsp;
    c. CLONE_SIGHAND 标志被设置，但 CLONE_VM 被清 0（共享信号处理程序的轻量级进程也必须共享内存描述符）。  
&emsp;

2. 通过调用 `security_task_create()` 以及稍后调用的 `security_task_alloc()` 执行所有附加的安全检查。Linux 2.6 提供扩展安全性的钩子函数，与传统 Unix 相比它具有更加强壮的安全模型。详情参见第二干章。  
&emsp;

3. 调用 `dup_task_struct()` 为子进程获取进程描述符。该函数执行如下操作：  

    a. 如果需要，则在当前进程中调用 `__unlazy_fpu()`，把 FPU、MMX 和 SSE/SSE2 寄存器的内容保存到父进程的 thread_info 结构中。稍后，`dup_task_struct()` 将把这些值复制到子进程的 thread_info 结构中。  
    &emsp;
    b. 执行 `alloc_task_struct()` 宏，为新进程获取进程描述符（task_struct 结构），并将描述符地址保存在 tsk 局部变量中。  
    &emsp;
    c. 执行 `alloc_thread_info` 宏以获取一块空闲内存区，用来存放新进程的 thread_info 结构和内核栈，并将这块内存区字段的地址存在局部变量 ti 中。正如在本章前面 “标识一个进程” 一节中所述：这块内存区字段的大小是 8KB 或 4KB 。  
    &emsp;
    d. 将 current 进程描述符的内容复制到 tsk 所指向的 task_struct 结构中，然后把 `tsk->thread_info` 为 ti。  
    &emsp;
    e. 把 current 进程的 thread_info 描述符的内容复制到 ti 所指向的结构中，然后把 `ti->task` 置为 tsk。  
    &emsp;
    f. 把新进程描述符的使用计数器（`tsk->usage`）置为 2，用来表示进程描述符正在被使用而且其相应的进程处于活动状态（进程状态即不是 EXIT_ZOMBIE，也不是 EXITDEAD）。  
    &emsp;
    g. 返回新进程的进程描述符指针（tsk）。  
&emsp;

4. 检查存放在 `current->signal->rlim[RLIMITNPROC].rlim_cur` 变量中的值是否小于或等于用户所拥有的进程数。如果是，则返回错误码，除非进程没有 root 权限。该函数从每用户数据结构 user_struct 中获取用户所拥有的进程数。通过进程描述符 user 字段的指针可以找到这个数据结构。  
&emsp;

5. 递增 user_struct 结构的使用计数器（`tsk->user->__count` 字段）和用户所拥有的进程的计数器（`tsk->user->processes`）。  
&emsp;

6. 检查系统中的进程数量（存放在 nr_threads 变量中）是否超过 max_threads 变量的值。这个变量的缺省值取决于系统内存容量的大小。总的原则是：所有 thread_info 描述符和内核栈所占用的空间不能超过物理内存大小的 1/8。不过，系统管理员可以通过写 `/proc/sys/kernel/threads-max` 文件来改变这个值。  
&emsp;

7. 如果实现新进程的执行域和可执行格式的内核函数（参见第二十章）都包含在内核模块中，如递增它们的使用计数器（参见附录二）。  
&emsp;

8. 设置与进程状态相关的几个关键字段：  

    a. 把大内核锁计数器 `tsk->lock_depth` 始化为 -1（参见第五章 “大内核锁” 一节）。  
    &emsp;
    b. 把 `tsk->did_exec` 字段初始化为 0：它记录了进程发出的 `execve()` 系统调用的次数。  
    &emsp;
    c. 更新从父进程复制到 `tsk->flags` 字段中的一些标志：首先清除 PF_SUPERPRIV 标志，该标志表示进程是否使用了某种超级用户权限。然后设置 PF_FORKNOEXEC 标志，它表示子进程还没有发出 `execve()` 系统调用。  
&emsp;

9. 把新进程的 PID 存入 `tsk->pid` 字段。  
&emsp;

10. 如果 clone_flags 参数中的 CLONE_PARENT_SETTID 标志被设置，就把子进程的 PID 复制到参数 parent_tidptr 指向的用户态变量中。  
&emsp;

11. 初始化子进程描述符中的 list_head 数据结构和自旋锁，并为与挂起信号、定时器及时间统计表相关的几个字段赋初值。  
&emsp;

12. 调用 `copy_semundo()`，`copy_files()`，`copy_fs()`，`copy_sighana()`，`copy_signal()`，`copy_mm()` 和 `copy_namespace()` 来创建新的数据结构，并把父进程相应数据结构的值复制到新数据结构中，除非 clone_flags 参数指出它们有不同的值。  
&emsp;

13. 调用 `copy_thread()`，用发出 `clone()` 系统调用时 CPU 寄存器的值（正如第十章所述，这些值已经被保存在父进程的内核栈中）来初始化子进程的内核栈。不过 `copy_thread()` 把 eax 寄存器对应字段的值（这是 `fork()` 和 `clone()` 系统调用在子进程中的返回值）字段强行置为 0。子进程描述符的 thread.esp 字段初始化为子进程内核栈的基地址，汇编语言函数（`ret_from_fork()`）的地址存放在 thread.eip 字段中。如果父进程使用 I/O 权限位图，则子进程获取该位图的一个拷贝。最后，如果 CLONE_SETTLS 标志被设置，则子进程获取由 `clone()` 系统调用的参数 tls 指向的用户态数据结构所表示的 TLS 段。  
&emsp;

> 细心的读者可能想知道 `copy_thread()` 怎样获得 `clone()` 的 tls 参数的值，因为 tls 并 不被传递给 `do_fork()` 和嵌套函数。我们将在第十章看到，通常通过拷贝系统调用的参数的值到某个 CPU 寄存器来把它们传递给内核；图此，这些值与其他专存器一起械保存在内
核态维栈中。`copy_thread()` 函数只查看 esi 的值在内核堆栈中对应的位置保存的地址。  

&emsp;

14. 如果 clone_flags 参数的值被置为 CLONE_CHILD_SETTID 或 CLONE_CHILD_CLEARTID，就把 child_tidptr 参数的值分别复制到 `tsk->set_chid_tid` 或 `tsk->clear_child_tid` 字段。这些标志说明：必须改变子进程用户态地址空面的 chila_tidptr 所指向的变量的值，不过实际的写操作要稍后再执行。  
&emsp;

15. 清除子进程 thread_info 结构的 TIF_SYSCALL_TRACE 标志，以便 `ret_from_fork()` 函数不会把系统调用结束的消息通知给调试进程（参见第十章 “进入和退出系统调用” 一节）。（因为对子进程的跟踪是由 `tsk->ptrace` 中的 PTRACE_SYSCALL 标志来控制的，所以子进程的系统调用跟踪不会被禁用。）  
&emsp;

16. 用 clone_flags 参数低位的信号数字编码初始化 `tsk->exit_signal` 字段，如果 CLONE_THREAD 标志被置位，就把 `tsk->exit_signal` 字段初始化为 -1。正如我们将在本章稍后 “进程终止” 一节所看见的，只有当线程组的最后一个成员（通常是线程组的领头）“死亡”，才会产生一个信号，以通知线程组的领头进程的父进程。  
&emsp;

17. 调用 `schea_fork()` 完成对新进程调度程序数据结构的初始化。该函数把新进程的状态设置为 TASK_RUNNING，并把 thread_info 结构的 preempt_count 字段设置为 1，从而禁止内核抢占（参见第五章 “内核抢占” 一节）。此外，为了保证公平的进程调度，该函数在父子进程之间共享父进程的时间片（参见第七章 “scheduler_tick() 函数” 一节）。  
&emsp;

18. 把新进程的 thread_info 结构的 cpu 字段设置为由 `smp_processor_id()` 所返回的本地 CPU 号。  
&emsp;

19. 初始化表示亲子关系的字段。尤其是，如果 CLONE_PARENT 或 CLONE_THREAD 被设置，就用 `current->real_parert` 的值初始化 `tsk->real_parent` 和 `tsk->parent`，因此，子进程的父进程似乎是当前进程的父进程。否则，把 `tsk->real_parent` 和 `tsk-parent` 置为当前进程。  
&emsp;

20. 如果不需要跟踪子进程（没有设置 CLONE_PTRAC 标志），就把 `tsk->ptrace` 字段设置为 0。`tsk->ptrace` 字段会存放一些标志，而这些标志是在一个进程被另外一个进程跟踪时才会用到的。采用这种方式，即使当前进程被跟踪，子进程也不会被跟踪。  
&emsp;

21. 执行 `SET_LINKS` 宏，把新进程描述符插人进程链表。  
&emsp;

22. 如果子进程必须被跟踪（`tsk->ptrace` 字段的 PT_PTRACED 标志被设置），就把 `current->parent` 赋给 `tsk->parent`，并将子进程插入调试程序的跟踪链表中。  
&emsp;

23. 调用 `attach_pid()` 把新进程描述符的 PID 插入 pidhash[PIDTYPE_PID] 散列表。  
&emsp;

24. 如果子进程是线程组的领头进程（CLONE_THREAD 标志被清 0）：  

    a. 把 `tsk->tgid` 的初值置为 `tsk->pid`。  
    &emsp;
    b. 把 `tsk->group_leader` 的初值置为 tsk 。  
    &emsp;
    c. 调用三次 `attach_pid()`，把子进程分别插入 PIDTYPE_TGID、PIDTYPE_PGID 和 PIDTYPE_SID 类型的 PID 散列表。  
&emsp;

25. 否则，如果子进程属于它的父进程的线程组（CLONE_THREAD 标志被设置）：  

    a. 把 `tsk->tgid` 的初值置为 `tsk->current->tgid`。  
    &emsp;
    b. 把 `tsk->group_leader` 的初值置为 `current->group_leader` 的值。  
    &emsp;
    c. 调用 `attach_pid()`，把子进程插入 PIDTYPE_TGID 类型的散列表中（更具体地说，插入 `current->group_leader` 进程的每个 PID 链表）。  
&emsp;

26. 现在，新进程已经被加人进程集合：递增 nr_threads 变量的值。  
&emsp;

27. 递增 total_forks 变量以记录被创建的进程的数量。  
&emsp;

28. 终止并返回子进程描述符指针（tsk）。
&emsp;

让我们回头看看在 `do_fork()` 结束之后都发生了什么。现在，我们有了处于可运行状态的完整的子进程。但是，它还没有实际运行，调度程序要决定何时把 CPU 交给这个子进程。在以后的进程切换中，调度程序继续完善子进程：把子进程描述符 thread 字段的值装入几个 CPU 寄存器。特别是把 thread.esp（即把子进程内核态堆栈的地址）装入 esp 寄存器，把函数 `ret_from_fork()` 的地址装入 eip 寄存器。这个汇编语言函数调用 `schedule_tail()` 函数（它依次调用 `finish_task_switch()` 来完成进程切换，参见第七章 “schedule() 函数” 一节），用存放在栈中的值再装载所有的寄存器，并强迫 CPU 返回到用户态。然后，在`fork()`、`vfork()` 或 `clone()` 系统调用结束时，新进程将开始执行。系统调用的返回值放在 eax 寄存器中：返回给子进程的值是 0，返回给父进程的值是子进
程的 PID。回顾 `copy_thread()` 对子进程的 eax 寄存器所执行的操作（`copy_process()` 的第13步），就能理解这是如何实现的。

除非 `fork()` 系统调用返回 0，否则，子进程将与父进程执行相同的代码（参见 `copy_process()` 的第13步）。应用程序的开发者可以按照 Unix 编程者熟悉的方式利用这一事实，在基于 PID 值的程序中插入一个条件语句使子进程与父进程有不同的行为。

