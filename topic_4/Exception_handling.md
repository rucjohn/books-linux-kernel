# 异常处理

**CPU 产生的大部分异常都由 Linux 解释为出错条件**。当其中一个异常发生时，内核就向引起异常的进程发送一个信号尚它通知一个反常条件。例如，如果进程执行了一个被 0 除的操作，CPU 就产生一个 “Divide error” 异常，并由相应的异常处理程序向当前进程发送一个 SIGFPE 信号，这个进程将采取若干必要的步骤来（从出错中）恢复或者中止运行（如果没有为这个信号设置处理程序的话）。

但是，在两种情况下，Linux 利用 CPU 异常更有效地管理硬件资源。
- 第一种情况已经在第三章 “保存和加载 FPU、MMX 及 XMM 寄存器” 一节描述过，“Device not availeble” 异常与 cr0 寄存器的 TS 标志一起用来把新值装人浮点寄存器。  
- 第二种情况指的是 “Page Fault” 异常，该异常推迟给进程分配新的页框，直到不能再推迟为止。

相应的处理程序比较复杂，因为异常可能表示一个错误条件，也可能不表示一个错误条件（参见第九章 “缺页异常处理程序” 一节）。

异常处理程序有一个标准的结构，由以下三部分组成：  
1. 在内核堆栈中保存大多数寄存器的内容（这部分用汇编语言实现）。  
2. 用高级的C函数处理异常。  
3. 通过 `ret_from_exception()` 函数从异常处理程序退出。  

为了利用异常，必须对 IDT 进行适当的初始化，使得每个被确认的异常都有一个异常处理程序。`trap_init()` 函数的工作是将一些最终值（即处理异常的函数）插入到 IDT 的非屏蔽中断及异常表项中。这是由函数 `Set_trap_gate()`、`set_intr_gate()`、`set_system_gate()`、`set_system_intr_gate()` 和 `set_task_gate()` 来完成的。

```
set_trap_gate(0, &divide_error);
set_trap_gate(1, &debug);
Set_intr_gate(2, &nmi);
set_system_intr_gate(3, &int3);
set_system_gate(4, &overflow);
set_system_gate(5, &bounds);
set_trap_gate(6, &invalid_op);
set_trap_gate(7, &device_not_available);
Set_task_gate(8, 31);
Set_trap_gate（9, &coprocessor_segment_overrun);
set_trap_gate(10, &invalid_TSS);
set_trap_gate(11, &segment_not_present);
set_trap_gate(12, &stack_segment);
set_trap_gate(13，&general_protection);
set_intr_gate(14，&page_fault);
set_trap_gate(16，&coprocessor_error);
Set_trap_gate(17，&alignment_check);
set_trap_gate(18，amachine_check);
set_trap_gate(19,&simd_coprocessor_error);
set_system_gate(128, &systemcal1);
```

由于 “Double fault” 异常表示内核有严重的非法操作，其处理是通过任务门而不是陷阱门或系统门来完成的，因而，试图显示寄存器值的异常处理程序并不确定 esp 寄存器的值是否正确。产生这种异常的时候，CPU 取出存放在 IDT 第 8 项中的任务门描述符，该描述符指向存放在 GDT 表第 32 项中的 TSS 段描述符。然后，CPU 用 TSS 段中的相关值装载 eip 和 esp 寄存器，结果是：处理器在自己的私有栈上执行 `doublefault_fn()` 异常处理函数。

现在我们要考察一旦一个典型的异常处理程序被调用，它会做些什么。由于篇幅所限，我们对异常处理仅做粗略的描述，尤其是我们不涉及下面的内容：  
1. 由一些处理函数发送给用户态进程的信号码（见第十一章中的表 11-8 ）。
2. 内核运行在 MS-DOS 虚拟模式（VM86 模式）时产生的异常，它们的处理是不同的。
3. “Debug” 异常。

## 为异常处理程序保存寄存器的值

让我们用 handler_name 来表示一个通用的异常处理程序的名字。（所有异常处理程序的实际名字都出现在前一部分的宏列表中。）每一个异常处理程序都以下列的汇编指令开始：
```
handler_name：
    pushl $0        /* only for some exceptions */
    pushl $do_handler_name
    jmp error_code
```

当异常发生时，如果控制单元没有自动地把一个硬件出错代码插入到栈中，相应的汇编语言片段会包含一条 `pushl $0` 指令，在栈中垫上一个空值。然后，把高级 C 函数的地址压进栈中，它的名字由异常处理程序名与 do_ 前缓组成。

标号为 error_code 的汇编语言片段对所有的异常处理程序都是相同的，除了 “Device not available” 这一个异常（参见第三章的 “保存和加载 FPU、MMX 及 XMM 寄存器” 一节）。这段代码执行以下步骤：  
1. 把高级 C 函数可能用到的寄存器保存在栈中。  
&emsp;

2. 产生一条 cld 指令来清 eflags 的方向标志 DF，以确保调用字符串指令时会自动增加edi和esi寄存器的值。  
&emsp;

3. 把栈中位于 esp+36 处的硬件出错码拷贝到 eax 中，给栈中这一位置存上值-1，正如我们将在第十一章的 “系统调用的重新执行” 一节中所看到的那样，这个值用来把 0×80 异常与其他异常隔离开。  
&emsp;

4. 把保存在栈中 esp+32 位置的 `do_handler_name()` 高级 C 函数的地址装入 edi 寄存器中，然后，在栈的这个位置写入 es 的值。  
&emsp;

5. 把内核栈的当前栈顶援贝到 eax 寄存器。这个地址表示内存单元的地址，在这个单元中存放的是第1步所保存的最后一个寄存器的值。  
&emsp;

6. 把用户数据段的选择符拷贝到 ds 和 es 寄存器中。  
&emsp;

7. 调用地址在 edi 中的高级 C 函数。  
&emsp;

被调用的函数从 eax 和 edx 寄存器而不是从栈中接收参数。我们已经遇见过一个从 CPU 寄存器获取参数的函数 `__switch_to()`，在第三章 “执行进程切换” 一节我们讨论过这个函数。

## 进入和离开异常处理程序

如前所述，执行异常处理程序的 C 函数名总是由 do_ 前缓和处理程序名组成。其中的大部分函数把硬件出错码和异常向量保存在当前进程的描述符中，然后，向当前进程发送一个适当的信号。用代码描述如下：
```
current->thread.error_code = error_code;
current->tnread.trap_no = vector;
force_sig(sig_number, current);
```

异常处理程序刚一终止，当前进程就关注这个信号。该信号要么在用户态由进程自己的信号处理程序（如果存在的话）来处理，要么由内核来处理。在后面这种情况下，内核一般会杀死这个进程（参见第十一章）。异常处理程序发送的信号已在表 4-1中列出。

异常处理程序总是检查异常是发生在用户态还是在内核态，在后一种情况下，还要检查是否由系统调用的无效参数引起。我们将在第十章 “动态地址检查：修正代码” 一节描述内核如何防御自己受无效的系统调用参数攻击。出现在内核态的任何其他异常都是由于内核的 bug 引起的。在这种情况下，异常处理程序认为是内核行为失常了。**为了避免硬盘上的数据崩溃，处理程序调用 `die()` 函数，该函数在控制台上打印出所有 CPU 寄存器的内容（这种转储就叫做 *kernel oops*），并调用 `do_exit()` 来终止当前进程**（参见第三章“进程终止”一节）。

当执行异常处理的 C 函数终止时，程序执行一条 jmp 指令以跳转到 `ret_from_exception()` 函数。这个函数将在后面的 “从中断和异常返回” 一节中进行描述。


