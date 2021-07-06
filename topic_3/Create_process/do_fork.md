# do_fork() 函数

`do_fork()` 函数负责处理 `clone()`、`fork()` 和 `vfork()` 系统调用，执行时使用下列参数：  

* *clone_flags*  
与 `clone()` 的参数 flags 相同。  
&emsp;

* *stack_start*  
与 `clone()` 的参数 child_stack 相同。  
&emsp;

* *regs*  
指向通用寄存器值的指针，通用寄用器的值是在从用户态切换到内核态时被保存到内核态堆栈中的（参见第四章 “do_IRQ() 函数” 一节）。  
&emsp;

* *stack_size*  
未使用（总是被设置为0）。  
&emsp;

* *parent_tidptr,child_tidptr*  
与 `clone()` 中的对应参数 ptid 和 ctid 相同。  
&emsp;

`do_fork()` 利用辅助函数 `copy_process()` 来创建进程描述符以及子进程执行所需要的所有其他内核数据结构。下面是 `do_fork()` 执行的主要步骤：  
1. 通过查找 pidmap_array 位图，为子进程分配新的PID（参见本章前面 “标识一个进程”一节）。  
&emsp;

2. 检查父进程的 ptrace 字段（current->ptrace）：如果它的值不等于 0，说明有另外一个进程正在跟踪父进程，因而，`do_fork()` 检查 debugger 程序是否自己想跟踪子进程（独立于由父进程指定的 CLONE_PTRACE 标志的值）。在这种情况下，如果子进程不是内核线程（CLONE_UNTRACED 标志被清 0），那么 `do_fork()` 函数设置 CLONE_PTRACE 标志。  
&emsp;

3. 调用 `copy_process()` 复制进程描述符。如果所有必须的资源都是可用的，该函数返回刚创建的 task_struct 描述符的地址。这是创建过程的关键步骤，我们将在 `do_fork()` 之后描述它。  
&emsp;

4. 如果设置了 CLONE_STOPPED 标志，或者必须跟踪子进程，即在 `p->ptrace` 中设置了 PT_PTRACED 标志，那么子进程的状态被设置成 TASK_STOPPED，并为子进程增加挂起的 SIGSTOP 信号（参见第十一章 “信号的作用” 一节）。在另外一个进程（不妨假设是跟踪进程或是父进程）把子进程的状态恢复为 TASK_RUNNING 之前（通常是通过发送 SIGCONT 信号），子进程将一直保持 TASK_STOPPED 状态。  
&emsp;

5. 如果没有设置 CLONE_STOPPED 标志，则调用 `wake_up_new_task()` 函数以执行下述操作：
    a. 调整父进程和子进程的调度参数（参见第七章 “调度算法” 一节）

    b. 如果子进程将和父进程运行在同一个 CPU 上，而且父进程和子进程不能共享同一组页表（ CLONE_VM 标志被清 0），那么，就把子进程插入父进程运行队列，插入时让子进程怡好在父进程前面，因此而迫使子进程先于父进程运行。如果子进程刷新其地址空间，并在创建之后执行新程序，那么这种简单的处理会产生较好的性能。而如果我们让父进程先运行，那么写时复制机制将会执行一系列不必要的页面复制。

    > 当内核创建一个新进程时父进程有可能会被转移到另一个 CPU 上执行。

    c. 否则，如果子进程与父进程运行在不同的 CPU 上，或者父进程和子进程共享同一组页表（ CLONE_VM 标志被设置），就把子进程插入父进程运行队列的队尾。  
&emsp;

6. 如果 CLONE_STOPPED 标志被设置，则把子进程置为 TASK_STOPPED 状态。  
&emsp;

7. 如果父进程被跟踪，则把子进程的 PID 存入 current 的 ptrace_message 字段并调用 `ptrace_notify()`。`ptrace_notify()` 使当前进程停止运行，并向当前进程的父进程发送 SIGCHLD 信号。子进程的祖父进程是跟踪父进程的 debugger 进程。SIGCHLD 信号通知 debugger 进程：current 已经创建了一个子进程，可以通过查找 `current->ptrace_message`字段获得子进程的 PID。  
&emsp;

8. 如果设置了 CLONE_VFORK 标志，则把父进程插入等待队列，并挂起父进程直到子进程释放自己的内存地址空间（也就是说，直到子进程结束或执行新的程序）。  
&emsp;

9. 结束并返画子进程的PID。  

