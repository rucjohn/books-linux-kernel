# 线性地址字段

下列宏简化了页表处理：  

* *PAGE SHIFT*  
  指定 Offset 字段的位数，当用于 80x86 处理器时，它产生的值为 12。由于页内所有地址都必须能放到 Offset 字段中，因此 80x86 系统的页的大小是 $$2^{12}=4096$$ 字节。PAGE_SHIFT 的值为 12 可以看作以 2 为底的页大小的对数。这个宏由 `PAGE_SIZE`使用以返回页的大小。最后，`PAGE_MASK`宏产生的值为 `Oxfffff000`，用以屏蔽 Offset 字段的所有位。  
&emsp;

* *PMD SHIFT*  
  指定线性地址的 Offset 字段和 Table 字段的总位数：换句话说，是页中间目录项可以映射的区域大小的对数。`PMD_SIZE`宏用于计算由页中间目录的一个单独表项所映射的区域大小，也就是一个页表的大小。`PMD_MASK`宏用于屏蔽 Offset 字段与 Table 字段的所有位。  

  

  当PAE被禁用时，`PMD_SHIFT`产生的值为 22（来自 Offset 的 12 位加上来自 Table 的 10 位），`PMD_SIZE`产生的值为 $$2^{22}$$ 或 4MB，`PMD_MASK` 产生的值为 `0×ffc00000`。相反，当 PAE 被激活时，`PMD_SHIFT` 产生的值为 21（来自 Offset 的 12 位加上来自 Table 的 9 位），`PMD_SIZE`产生的值为 $$2^{21}$$ 或 2MB，`PMD_MASK` 产生的值为`0xffe00000`。  

  

  大型页不使用最后一级页表，所以产生大型页尺寸的 `LARGE_PAGE_SIZE` 宏等于 PMD_SIZE（2PMD_SHIFT），而在大型页地址中用于屏蔽 Offset 字段和 Table 字段的所有位的 `LARGE_PAGE_MASK` 宏，就等于 `PMD_MASK`。  
&emsp;

* *PUD SHIFT*  
  确定页上级目录项能映射的区域大小的对数。`PUD_SIZE`宏用于计算页全局目录中的一个单独表项所能映射的区域大小。`PUD_MASK` 宏用于屏蔽 Offset 字段、Table 字段、MiddleAir 字段和 UpperAir 字段的所有位。  

  

  在 80x86 处理器上，`PUD_SHIFT` 总是等价于 `PMD_SHIFT`，而 `PUD_SIZE` 则等于 4MB 或 2MB。  
&emsp;

* *PGDIR_SHIFT*  
  确定页全局目录项能映射的区域大小的对数。`PGDIR_SIZE` 宏用于计算页全局目录中一个单独表项所能映射区域的大小。`PGDIR_MASK` 宏用于屏蔽 Offset、Table、Middle Air及Upper Air 字段的所有位。  

  

  当PAE被禁止时，`PGDIR_SHIFT` 产生的值为 22（与 `PMD_SHIFT` 和 `PUD_SHIFT` 产生的值相同），`PGDIR_SIZE` 产生的值为 2 或 4MB，以及 `PGDIR_MASK` 产生的值为 `0×ffc00000`。相反，当 PAE 被激活时，`PGDIR_SHIFT` 产生的值为 30（12位 Offset 加 9 位 Table 再加 9 位 Middle Air），`PGDIR_SIZE` 产生的值为 $$2^{30}$$ 或 1GB 以及 `PGDIR_MASK`产生的值为 `0×C0000000`。  
&emsp;

* *PTRS_PER_PTE，PTRS_PER_PMD，PTRS_PER_PUD 以及 PTRS_PER_PGD*  
  用于计算页表、页中间目录、页上级目录和页全局目录表中表项的个数。当 PAE 被禁止时，它们产生的值分别为 1024，1，1 和 1024。当 PAE 被激活时，产生的值分别为 512，512，1 和 4。
