### 进程实现

为了让内核管理进程，每个进程都由一个二进制描述符（*process descriptor*）表示，这个描述符包含有关进程当前状态的信息。

当内核暂停一个进程的执行时，就把几个有关处理器寄存器的内容保存在进程描述符中。这些寄存器包括：

- 程序计数器（PC）和栈指针（SP）寄存器
- 通用寄存器
- 浮点寄存器
- 包含 CPU 状态信息的处理器寄存器（处理器状态字，Process Status Word）
- 用来跟踪进程对 RAM 访问的内存管理寄存器

当内核决定恢复执行一个进程时，它用进程描述符中合适的字段来装载 CPU 寄存器。因为程序计数器中所存的值指向下一条将要执行的指令，所以进程从它停止的地方恢复执行。

当一个进程不在 CPU 上执行时，它正在等待某一事件。Unix 内核可以区分很多等待状态，这些等待状态通常由进程描述符队列实现。每个（可能为空）队列对应一组等待特定事件的进程。