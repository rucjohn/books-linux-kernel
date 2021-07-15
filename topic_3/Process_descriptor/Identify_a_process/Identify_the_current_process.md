# 标识当前进程

从效率的观点来看，刚才所讲的 thread_info 结构与内核态堆栈之间的紧密结合提供的主要好处是：内核很容易从 esp 寄存器的值获得当前在 CPU 上正在运行进程的 thread_info 结构的地址。事实上，如果 thread_union 结构长度是 8K（$$2^{13}$$字节），则内核屏蔽掉 esp 的低 13 位有效位就可以获得 thread_info 结构的基地址；而如果 thread_union 结构长度是 4K，内核需要屏蔽掉 esp 的低 12 位有效位。这项工作由 `current_thread_info()` 函数来完成，它产生如下一些汇编指令：
```
movl $0xffffe000, %ecx  /* 或者是用于 4K 堆栈的 0xfffff000 */
andl %esp, %ecx
movl %ecx, p
```

这三条指令执行以后，p 就包含在执行指令的 CPU 上运行的进程的 thread_info 结构的指针。

进程最常用的是进程描述符的地址而不是 thread_info 结构的地址。为了获得当前在 CPU 上运行进程的描述符指针，内核要调用 `current` 宏，该宏本质上等价于 `current_thread_info()->task`，它产生如下汇编语言指令：
```
movl $0xffffe000, %ecx  /* 或者是用于 4K 堆栈的 0xfffff000 */
andl %esp, %ecx
movl (%ecx), p
```

因为 task 字段在 thread_info 结构中的偏移量为 0，所以执行完这三条指令后，p 就包含在 CPU 上运行进程的描述符指针。

`current` 宏经常作为进程描述答字段的前缀出现在内核代码中，例如，`current->pid` 返回在 CPU 上正在执行的进程的 PID。

用栈存放进程描述符的另一个优点体现在多处理器系统上：如前所述，对于每个硬件处理器，仅通过检查栈就可以获得当前正确的进程。早先的 Linux 版本没有把内核栈与进程描述符存放在一起，而是强制引入全局静态变量 `current` 来标识正在运行进程的描述符。在多处理器系统上，有必要把 `current` 定义为一个数组，每一个元素对应一个可用 CPU。
