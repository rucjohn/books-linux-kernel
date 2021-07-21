# 异常

80×86 微处理器发布了大约 20 种不同的异常。内核必须为每种异常提供一个专门的异常处理程序。对于某些异常，CPU 控制单元在开始执行异常处理程序前会产生一个硬件出错码（*hardware error code*），并且压入内核态堆栈。

> 精确的数字依赖于处理器模型。

下面的列表给出了在 80x86 处理器中可以找到的异常的向量、名字、类型及其简单描述。更多的信息可以在 Intel 的技术文挡中找到。

* *0 - “Divide error”（故障）*  
当一个程序试图执行整数被 0 除操作时产生。  
&emsp;

* *1 - “Debug”（陷阱或故障）*  
产生于：①设置 eflags 的 TF 标志时（对于实现调试程序的单步执行是相当有用的），②一条指令或操作数的地址落在一个活动 debug 寄存器的范围之内（参见第三章的 “硬件上下文” 一节）。  
&emsp;

* *2 - 未用*  
为非屏蔽中断保留（利用 NMI 引脚的那些中断）。  
&emsp;

* *3 - “Breakpoint”（陷阱）*  
由 int3（断点）指令（通常由 debugger 插入）引起。  
&emsp;

* *4 - “Overflow”（陷阱）*  
当 eflags 的 OF（overflow）标志被设置时，into（检查溢出）指令被执行。  
&emsp;

* *5 - “Bounds check”（故障）*
对于有效地址范围之外的操作数，bound（检查地址边界）指令被执行。  
&emsp;

* *6 - “Invalid opcode”（故障）*  
CPU 执行单元检测到一个无效的操作码（决定执行操作的机器指令部分）。  
&emsp;

* *7 - “Device not available”（故障）*  
随着 cr0 的 TS 标志被设置，ESCAPE、MMX 或 XMM 指令被执行（参见第三章的 “保存和加载 FPU、MMX 及 XMM 寄存器”一节）。  
&emsp;

* *8 - “Double fault”（异常中止）*
正常情况下，当 CPU 正试图为前一个异常调用处理程序时，同时又检测到一个异常，两个异常能被串行地处理。然而，在少数情况下，处理器不能串行地处理它们，因而产生这种异常。  
&emsp;

* *9 - “Coprocessor segment overrun”（异常中止）*  
因外部的数学协处理器引起的问题（仅用于 80386 微处理器）。  
&emsp;

* *10 - “Invalid TSS”（故障）*  
CPU 试图让一个上下文切换到有无效的 TSS 的进程。  
&emsp;

* *11 - “Segment not present”（故障）*  
引用一个不存在的内存段（段描述符的 Segment-Present 标志被清 0）。  
&emsp;

* *12 - “Stack segment fault”（故障）*  
试图超过栈段界限的指令，或者由 SS 标识的段不在内存。  
&emsp;

* *13 - “General protection”（故障）*  
违反了 80x86 保护模式下的保护规则之一。  
&emsp;

* *14 - “Page fault”（故障）*
寻址的页不在内存，相应的页表项为空，或者违反了一种分页保护机制。  
&emsp;

* *15 - 由 Intel 保留*  
&emsp;

* *16 - “Floating point error”（故障）*  
集成到CPU芯片中的浮点单元用信号通知一个错误情形，如数字溢出，或被 0 除。  
&emsp;

* *17 - “Alignment check”（故障）*
操作数的地址没有被正确地对齐（例如，一个长整数的地址不是 4 的倍数）。  
&emsp;

* *18 - “Machine check”（异常中止）*  
机器检查机制检测到一个 CPU 错误或总线错误。  
&emsp;

* *19 - “SIMD floating point exception”（故障）*  
集成到 CPU 芯片中的 SSE 或 SSE2 单元对浮点操作用信号通知一个错误情形。  
&emsp;

> 80x86 微处理器也产生这个异常，这发生在执行一个带符号的除法运算，而运算结果不能以带符号整数存放的时候（例如 `-2147483648` 到 `-1` 之间的一个除法运算）。

20 ~ 31 这些值由 Intel 留作将来开发。如表 4-1 所示，每个异常都由专门的异常处理程序来处理（参见本章后面的 “异常处理” 一节），它们通常把一个 Unix 信号发送到引起异常的进程。

**表 4-1：由异常处理程序发送的信号**

编号 | 异常 | 异常处理程序 | 信号
--- | --- | --- | ---
0 | Divide error | divide_error() | SIGFPE
1 | Debug | debug() | SIGTRA
2 | NMI | nmi() | None
3 | Breakpoint | int3() | SIGTRAE
4 | Overflow | Overflow(）| SIGSEGV
5 | Bounds check | bounds() | SIGSEGV
6 | Invalid opcode | invalid_op() | SIGILL
7 | Device not available | device_not_available() | None
8 | Double fault | doublefault_fn() | None
9 | Coprocessor segment overrun | coprocessor segment_overrun() | SIGFPE
10 | Invalid TSS | invalid_tss() | SIGSEGV
11 | Segment not present | segment_not_present() | SIGBUS
12 | Stack exception | stack_segment() | SIGBUS
13 | General protection | general_protection() | SIGSEGV
14 | Page fault | page_fault() | SIGSEGV
15 | Intel reserved | None | None
16 | Floating point error | coprocessor_error() | SIGFPE
17 | Alignment check | alignment_check() | SIGSEGV
18 | Machine check | machine_check() | None 
19 | SIMD floating point | simd_coprocessor_error() | SIGFPE

