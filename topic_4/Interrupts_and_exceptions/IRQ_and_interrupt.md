# IRQ和中断

每个能够发出中断请求的硬件设备控制器都有一条名为 IRQ（Interrupt ReQuest）的输出线。**所有现有的 IRQ 线（*IRO line*）都与一个名为可编程中断控制器（*Programmable Interrupt Controuer, PIC*）的硬件电路的输入引脚相连**，可编程中断控制器执行下列动作：

1. 监视 IRQ 线，检查产生的信号（raised signal）。如果有条或两条以上的 IRQ 线上产生信号，就选择引脚编号较小的IRQ线。
2. 如果一个引发信号出现在IRQ线上：
    a. 把接收到的引发信号转换成对应的向量。  
    b. 把这个向量存放在中断控制器的一个 I/O 端口，从而允许 CPU 通过数据总线读此向量。  
    c. 把引发信号发送到处理器的 INTR 引脚，即产生一个中断。  
    d. 等待，直到 CPU 通过把这个中断信号写进可编程中断控制器的一个 I/O 端口来确认它；当这种情况发生时，清 INTR 线。  
3. 返回到第 1 步。


> 复杂一些的设备有几条 IRQ线 ，例如，PCI 卡可能使用多达 4 条 IRQ 线。

IRQ 线是从 0 开始顺序编号的，因此，第一条 IRQ 线通常表示成 IRQ0。与 IRQ*n* 关联的 Intel 的缺省向量是 *n+32* 。如前所述，通过向中断控制器端口发布合适的指令，就可以修改 IRQ 和向量之间的映射。

可以有选择地禁止每条 IRQ 线。因此，可以对 PIC 编程从而禁止 IRQ，也就是说，可以告诉 PIC 停止对给定的 IRQ 线发布中断，或者激活它们。禁止的中断是丢失不了的，它们一且被激活，PIC 就又把它们发送到 CPU。这个特点被大多数中断处理程序使用，因为这充许中断处理程序逐次地处理同一类型的 IRQ。

有选择地激活/禁止 IRQ 线不同于可屏蔽中断的全局屏蔽/非屏蔽。当 eflags 寄存器的 IF 标志被清 0 时，由 PIC 发布的每个可屏蔽中断都由 CPU 暂时忽略。cli 和 sti 汇编指令分别清除和设置该标志。

传统的 PIC 是由两片 8259A 风格的外部芯片以 “级联” 的方式连接在一起的。每个芯片可以处理多达 8 个不同的 IRQ 输入线。因为从 PIC 的 INT 输出线连接到主 PIC 的 IRQ2引脚，因此，可用 IRQ 线的个数限制为 15。
