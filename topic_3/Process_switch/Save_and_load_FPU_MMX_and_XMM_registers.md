# 保存和加载FPU、MMX及XMM寄存器

从 Intel80486DX 开始，算术浮点单元（floating-point unit，FPU）已被集成到CPU中。数学协处理这个名词使人想起使用昂贯的专用芯片执行浮点计算的岁月。然而，为了维持与旧模式的兼容，浮点算术函数用 ESCAPE指令来执行，这个指令的一些前缀字节在 `0×d8` 和 `0xdf` 之间。这些指令作用于包含在 CPU 中的浮点寄存器集。显然，如果一个进程正在使用 ESCAPE 指令，那么，浮点寄存器的内容就属于它的硬件上下文，并且应该被保存。

在最近的 Pentium 模型中，Intel 在它的微处理器中引入一个新的汇编指令集，叫做 MMX 指令，用来加速多媒体应用程序的执行。MMX 指令作用于 FPU 的浮点寄存器。选择这种体系结构的明显缺点是编程者不能把浮点指令与MMX指令混在一起使用。优点是操作系统设计者能忽视新指令集，因为保存浮点单元状态的任务切换代码可以不加修改地应用到保存 MMX 状态。

MMX 指令加速了多媒体应用程序的执行，因为它们在处理器内部引入了单指令多数据（single-instruction multiple-data，SIMD）流水线。Pentium III 模型扩展了这种 SIMD能力：它引入 SSE 扩（Streaming SIMD Extensions），该扩展为处理包含在 8 个 128 位寄存器（叫做 XMM 寄存器）的浮点值增加了功能。这样的寄存器不与 FPU 和 MMX 寄存器重叠，因此 SSE 和 FPU/MMX 指令可以随意地混合。Pentium 4 模型指令还引入另一种特点：SSE2 扩展，该扩展基本上是 SSE 的一个扩展，支持高精度浮点值。SSE2 与 SSE 使用同一 XMM 寄存器集。

80x86 微处理器并不在 TSS 中自动保存 FPU、MMX 和 XMM 寄存器。不过，它们包含某种硬件支持，能在需要时保存这些寄存器的值。硬件支持由 cr0 寄存器中的一个 TS（Task-Switching）标志组成，遵循以下规则：
- 每当执行硬件上下文切换时，设置TS标志。
- 每当 TS 标志被设置时执行 ESCAPE、MMX、SSE 或 SSE2 指令，控制单元就产生一个 “Device not available” 异常（参见第四章）。

TS 标志使得内核只有在真正需要时才保存和恢复 FPU、MMX 和 XMM 寄存器。为了说明它如何工作，让我们假设进程 A 使用数学协处理器。当发生上下文切换时，内核置 TS 标志并把浮点寄存器保存在进程 A 的 TSS 中。如果新进程 B 不利用协处理器，内核就不必恢复浮点寄存器的内容。但是，只要 B 打算执行 ESCAPE 或 MMX 指令，CPU 就产生一个 “Device not available” 异常，并且相应的异常处理程序用保存在进程 B 中的 TSS 的值装载浮点寄存器。

现在，让我们描述为处理 FPU、MMX 和 XMM 寄存器的选择性装入而引入的数据结构。它们存放在进程描述符的 thread.i387 子字段中，其格式由 i387_union 联合体描述：  
```
union i387_union {
  struct i387_fsave_struct    fsave;
  struct i387_fxsave_struct   fxsave;
  struct i387_soft_struct     soft;
}
```

正如你看到的，这个字段只可以存放三种不同数据结构中的一种。
- i387_soft_struct 结构由无数学协处理器的 CPU 模型使用；Linux 内核通过软件模拟协处理器来支持这些老式芯片。不过，我们不打算进一步讨论这种遗留问题。
- 1387_fsave_struct 结构由具有数学协处理器、也可能有 MMX 单元的 CPU 模型使用。
- i387_fxsave_struct 结构由具有 SSE 和 SSE2 扩展功能的 CPU 模型使用。

进程描述符包含两个附加的标志：
- 包含在 thread_info 描述符的 status 字段中的 TS_USEDFPU 标志。它表示进程在当前执行的过程中是否使用过 FPU、MMX 和 XMM 寄存器。
- 包含在 task_struct 描述符的 flags 字段中的 PF_USED_MATH 标志。这个标志表示 thread.i387 子字段的内容是否有意义。该标志在两种情况下被清 0（没有意义），如下所示：
  - 当进程调用 `execve()` 系统调用（参见第二十章）开始执行一个新程序时。因为控制权将不再返回到前一个程序，所以当前存放在 thread.i387 中的数据也不再使用。
  - 当在用户态下执行一个程序的进程开始执行一个信号处理程序时（参见第十一章）。因为信号处理程序与程序的执行流是异步的，因此，浮点寄存器对信号处理程序来说可能是毫无意义的。不过，内核开始执行信号处理程序之前在 thread.i387 中保存浮点寄存器，处理程序结束以后恢复它们。因此，信号处理程序可以使用数学协处理器。

## 保存FPU寄存器

如前所述，`__switch_to()` 函数把被替换进程 prev 的描述符作为参数传递给 `_unlazy_fpu` 宏，并执行该宏。这个宏检查 prev 的 TS_USEDFPU 标志值。如果该标志被设置，说明 prev 在这次执行中使用了 FPU、MMX、SSE 或 SSE2 指令；因此内核必须保存相关的硬件上下文：  
```
if (prev->thread_info->status & TS_USEDPPU)
    save_init_fpu(prev);
```

`save_init_fpu()` 函数依次执行下列操作：  
1. 把 FPU 寄存器的内容转储到 prev 进程描述符中，然后重新初始化 FPU。如果 CPU 使用 SSE/SSE2 扩展，则还应该转储 XMM 寄存器的内容，并重新初始化 SSE/SSE2 单元。一对功能强大的嵌入式汇编语言指令处理每件事情，如果 CPU 使用 SSE/SSE2扩展，则：  
```
asm volatile( "fxsave %0; fnclex"
    : "=m" (tsk->thread.1387.fxSave) );
```
否则：  
```
asm volatile( "fnsave %0; fwait"
    : "=m" (tsk->thread.1387.fsave) );
```

2. 重置 prev 的 TS_USEDFPU 标志：  
```
prev->thread_info->status &= ~TS_USEDFPU;
```

3. 用`stts()` 宏设置 cr0 的 TS 标志，实际上，该宏产生下面的汇编语言指令：  
```
movl %cr0，eax
orl $8,%eax
movl %eax，%cr0
```

## 装载FPU寄存器

当 next 进程刚恢复执行时，浮点寄存器的内容还没有被恢复，不过，cr0 的 TS 标志位已由 `__unlazy_fpu()` 设置。因此，next 进程第一次试图执行 ESCAPE、MMX 或 SSE/SSE2 指令时，控制单元产生一个 “Device not available” 异常，内核（更确切地说，由异常调用的异常处理程序）运行 `math_state_restore()` 函数。处理程序把 next 进程当作 current 进程。  
```
void math_state_restore()
{
    asm volatile ("clts"); /* clear the TS flag of cr0 */
    if (!(current->flags & PF_USED_MATH))
        init_fpu(current);
    restore_fpu(current);
    current->thread.status != TS_USEDFPU;
```

这个函数清 cr0 的 TS 标志，以便进程以后执行 FPU、MMX 或 SSE/SSE2 指令时不再触发 “设备不可用” 的异常。如果 thread.i387 子字段中的内容是无效的，也就是说，如果 PF_USED_MATH 标志等于 0，就调用`init_fpu()` 重新设置 thread.i387 子字段，并把 PF_USED_MATH 标志的当前值置为 1。`restore_fpu()` 函数把保存在 thread.i387 子字段中的适当值载入 FPU 寄存器。为此，根据 CPU 是否支持 SSE/SSE2 扩展来使用 fxrstor 或 frstor 汇编语言指令。最后，`math_state_restore()` 设置 TS_USEDFPU 标志。
