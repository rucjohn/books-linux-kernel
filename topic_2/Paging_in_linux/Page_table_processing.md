# 页表处理

pte_t、pmd_t、pud_t 和 pgd_t 分别描述页表项、页中间目录项、页上级目录和页全局目录项的格式。当 PAE 被激活时它们都是 64 位的数据类型，否则都是 32 位数据类型。pgprot_t 是另一个 64 位（PAE 激活时）或 32 位（PAE 禁用时）的数据类型，它表示与一个单独表项相关的保护标志。

五个类型转换宏（`__pte`、`__pmd`、`__pud`、`__pgd` 和 `__pgprot`）把一个无符号整数转换成所需的类型。另外的五个类型转换宏（`pte_val`、`pmd_val`、`pud_val`、`pgd_val` 和 `pgprot_val`）执行相反的转换，即把上面提到的四种特殊的类型转换成一个无符号整数。

内核还提供了许多宏和函数用于读或修改页表表项：
- 如果相应的表项值为0，那么，宏 `pte_none`、`pmd_none`、`pud_none` 和 `pgd_none` 产生的值为 1，否则产生的值为 0。
- 宏 `pte_clear`、`pmd_clear`、`pud_clear` 和 `pgd_clear` 清除相应页表的一个表项，由此禁止进程使用由该页表项映射的线性地址。`ptep_get_and_clear()` 函数清除一个页表项并返回前一个值。
- `set_pte`、`set_pmd`、`set_pud` 和 `set_pgd` 向一个页表项中写人指定的值。`set_pte_atomic` 与 `set_pte` 的作用相同，但是当 PAE 被激活时它同样能保证 64 位的值被原子地写人。
- 如果 a 和 b 两个员表项指向同一页并直指定相同的访同优先级，那么 `pte_same(a, b)` 返回 1，否则返回 0。
- 如果页中间目录项 e 指向一个大型页（2MB 或 4MB），那么 `pmd_large(e)` 返回 1，否则返回 0。

宏 `pmd_bad` 由函数使用并通过输人参数传递来检查页中间目录项。如果目录项指向一个不能使用的页表，也就是说，如果至少出现以下条件中的一个，则这个宏产生的值为 1：
- 页不在主存中（Present 标志被清除）。
- 页只允许读访问（Read/write 标志被清除）。
- Acessed 或者 Dirty 位被清除（对于每个现有的页表，Linux 总是强制设置这些标志）。

`pua_bad` 宏和 `pgd_bad` 宏总是产生 0。没有定义 `pte_bad` 宏，因为页表项引用一个不在主存中的页、一个不可写的页或一个根本无法访问的页都是合法的。

如果一个页表项的 Present 标志或者 PageSize 标志等于 1，则 `pte_present` 宏产生的值为 1，否则为 0。前面讲过页表项的 PageSize 标志对微处理器的分页单元来讲没有意义，然而，对于当前在主存中却又没有读、写或执行权限的页，内核将其 Present 和 PageSize 分别标记为 0 和 1。这样，任何试图对此类页的访问都会引起一个缺页异常，因为页的 Present 标志被清 0，而内核可以通过检查 PageSize的 值来检测到产生异常并不是因为缺页。

如果相应表项的 Present 标志等于 1，也就是说，如果对应的页或页表被载入主存，`pmd_present` 宏产生的值为1。`pud_present` 宏和 `pgd_present` 宏产生的值总是 1。

表 2-5 中列出的函数用来查询页表项中任意一个标志的当前值；除了 `pte_file()` 外，其他函数只有在 `pte_present` 返回 1 的时候，才能正常返回页表项中任意一个标志。

**表 2-5：读页标志的函数**

函数名称 | 说明
--- | ---
pte_user() | 读 User/Supervisor 标志
pte_read() | 读 User/Supervisor 标志（表示 80x86 处理器上的页不受读的保护）
pte_write() | 读 Read/Write 标志
pte_exec() | 读 User/Supervisor 标志（ 80x86 处理器上的页不受代码执行的保护）
pte_dirty() | 读 Dirty 标志
pte_young() | 读Accessed标志
pte_file() | 读 Dirty 标志（当 Present 标志被清除而 Dirty 标志被设置时，页属于一个非线性磁盘文件映射，参见第十六章）

表 2-6 列出的另一组函数用于设置页表项中各标志的值。

**表 2-6：设置页标志的函数**

函数名称 | 说明
--- | ---
mk_pte_huge() | 设置页表项中的 PageSize 和 Present 标志
pte_wrprotect() | 清除 Read/write 标志
pte_rdprotect() | 清除 User/supervisor 标志
pte_exprotect() | 清除 User/Supervisor 标志
pte_mkwrite() | 设置 Read/write 标志
pte_mkread() | 设置 User/supervisor 标志
pte_mkexec() | 设置 User/supervisor 标志
pte_mkclean() | 清除 Dirty 标志
pte_mkdirty() | 设置 Dirty 标志
pte_mkold() | 清除 Accessea 标志（把此页标记为未访问）
pte_mkyoung() | 设置 Accessed 标志（把此页标记为访问过）
pte_modify(p, v)| 把页表项 p 的所有访问权限设置为指定的值
ptep_set_wrprotect() | 与 pte_wrprotect() 类似，但作用于指向页表项的指针
ptep_set_access_flags() | 如果 Dirty 标志被设置为 1，则将页的存取权限设置为指定的值，并调用 `flush_tlb_page()` 函数（参见本章 “转换后援缓冲器（TLB）” 一节）
ptep_mkdirty() | 与 pte_mkdirty() 类似，但作用于指向页表项的指针
ptep_test_and_clear_dirty() | 与 pte_mkclean() 类似，但作用于指向页表项的指针并返回 Dirty 标志的旧值
ptep_test_and_clear_young() | 与 pte_mkold() 类似，但作用于指向页表项的指针并返回 Accessed 标志的旧值


现在，我们来讨论表 2-7 中列出的宏，它们把一个页地址和一组保护标志组合成页表项或者执行相反的操作，从一个页表项中提取出页地址。请注意这其中的一些宏对页的引用是通过 “页描述符” 的线性地址（参见第八章 “页描述符” 一节），而不是通过该页本身的线性地址。

**表 2-7：对页表项操作的宏**

宏名称 | 说明
--- | ---
pgd_index(addr) | 找到线性地址 addr 对应的目录项在页全局目录中的索引（相对位置）
pgd_offset(mm, addr) | 接收内存描述符地址 mm（参见第九章）和线性地址 addr 作为参数。这个宏产生地址 addr 在页全局目录中相应表项的线性地址；通过内存描述符 mm 内的一个指针可以我到这个页全局目录
pgd_offset_k(addr) | 产生主内核页全局目录中的某个项的线性地址，该项对应于地址 addr（参见稍后“内核页表”一节）
pgd_page(pgd) |通过负全局目录项pgd产生页上级目录所在负框的页描述符地址。在两级或三级分页系统中，该宏等价于 pud_page()，后者应用于页上级目录项
pud_offset(pgd, addr) | 接收指向页全局目录项的指针pgd和线性地址 addr 作为参数。这个宏产生员上级目录中目录项 addr 对应的线性地址。在两级或三级分页系统中，该宏产生 pgd，即一个页全局目录项的地址
pud_page(pud) | 通过页上级目录项pud产生相应的页中间日录的线性地址。在两级分页系统中，该宏等价于 pma_page()，后者应用于页中间目录项
pmd_index(addr) | 产生线性地址 addr 在页中间目录中所对应目录项的索引（相对位置）
pmd_offset(pud, addr) | 接收指向页上级目录项的指针pud和线性地址 addr 作为参数。这个宏产生目录项addr在页中间目录中的偏移地址。在两级或三级分页系统中，它产生 pud，即页全局目录项的地址
pmd_page(pmd) | 通过页中间目录项pmd产生相应页表的页描述符地址。在两级或三级分页系统中，pmd 实际上是页全局目录中的一项
mk_pte(p, prot) | 接收页描述符地址p和一组存取权限 prot 作为参数，并创建相应的页表项
pte_index(addr) | 产生线性地址addr对应的表项在页表中的索引（相对位置）
pte_offset_kernel(dir, addr) | 线性地址 addr 在页中间目录dir中有一个对应的项该宏就产生这个对应项，即页表的线性地址。另外，该宏只在主内核页表上使用（参见稍后的 “内核页表” 一节）
pte_offset_map(dir, addr) | 接收指向一个页中间目录项的指针dir和线性地址 addr 作为参数，它产生与线性地址 addr 相对应的页表项的线性地址。如果页表被保存在高端内存中，那么内核建立一个临时内核映射（参见第八章 “高端内存页框的内核映射” 一节），并用 pte_unmap 对它进行释放。`pte_offset_map_nested` 宏和 `pte_unmap_nested` 宏是相同的，但它们使用不同的临时内核映射
pte_page(x) | 返回页表项×所引用页的描述符地址
pte_to_pgoff(pte) | 从一个负表项的 pte 字段内容中提取出文件偏移量这个偏移意对应着一个非线性文件内存映射所在的页（参见第十六章 “非线性内存映射” 一节）
pgoff_to_pte(offset) | 为非线性文件内存映射所在的页创建对应页表项的内容

这里罗列最后一组函数来简化页表项的创建和撤消。  

当使用两级页表时，创建或删除一个页中间目录项是不重要的。如本节前部分所述，页中间目录仅含有一个指向下属页表的目录项。所以，页中间目录项只是页全局目录中的一项而已。然而当处理页表时，创建一个页表项可能很复杂，因为包含页表项的那个页表可能就不存在。在这样的情况下，有必要分配一个新页框，把它填写为 0，并把这个表项加入。  

如果 PAE 被激活，内核使用三级页表。当内核创建一个新的页全局目录时，同时也分配四个相应的页中间目录：只有当父页全局目录被释放时，这四个页中间目录才得以释放。  

当使用两级或三级分页时，页上级目录项总是被映射为页全局目录中的一个单独项。  

与以往一样，表2-8中列出的函数描述是针对 80x86 体系结构的。  

**表 2-8：页分配函数**
函数名称 | 说明
--- | ---
pgd_alloc(mm) | 分配一个新的页全局目录。如果 PAE 被激活，它还分配三个对应用户态线性地址的子页中间目录参数 mm（内存描述符的地址）在 80x86 体系结构上被忽略
pgd_free(pgd) | 释放页全局目录中地址为 pgd 的项。如果 PAE 被激活，它还将释放用户态线性地址对应的三个页中间目录
pud_alloc(mm, pgd, addr) | 在两级或三级分页系统下，这个函数什么也不做：它仅仪返回页全局目录项pgd的线性地址
pud_free(x)| 在两级或三级分页系统下，这个宏什么也不做
pmd_alloc(mm, pud, addr) | 定义这个函数以便普通三级分页系统可以为线性地址 addr 分配一个新的负中间自录。如果 PAE 未被激活，这个函数只是返回输入参数 pud 的值，也就是说，返回页全局目录中目录项的地址。如果 PAE 被激活，该函数返回线性地址 addr 对应的页中间目录项的线性地址。参数 mm 被忽略
pmd_free(x) | 该函数什么也不做，因为页中间目录的分配和释放是随同它们的父全局目录一同进行的
pte_alloc_map(mm, pmd, addr) | 接收页中间目录项的地址pmd和线性地址 addr 作为参数，并返回与 addr 对应的页表项的地址。如果页中间目录项为空，该函数通过调用函数 pte_alloc_one() 分配一个新页表。如果分配了一个新页表，addr 对应的项就被创建，同时 User/Supervisor 标志被设置为 1。如果页表被保存在高端内存，如内核建立一个临时内核映射（参见笔八章 “高端内存页框的内核映射” 一节），并用 pte_unmap 对它进行释放
pte_alloc_kernel(mm, pmd, addr) | 如果与地址 addr 相关的页中间目录项 pmd 为空，该函数分配一个新负表。然后返回与 addr 相关的负表项的线性地址。该函数仅被主内核页表使用（参见稍后 “内核页表” 一节）
pte_free(pte) | 释放与页描述符指针 pte 相关的页表
pte_free_kernel(pte) | 等价于 pte_free()，但由主内核页表使用
clear_page_range(mmu, start, end) | 从线性地址 start 到 end 通过反复释放页表和清除页中间目录项来清除进程页表的内容

