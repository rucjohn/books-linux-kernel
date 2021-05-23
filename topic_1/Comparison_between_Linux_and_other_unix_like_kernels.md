## Linux 与其他类 Unix 内核的比较

市场上各种类 Unix 系统在很多重要的方面有所不同，其中有些系统已经有很长的历史，并且显得有点过时。所有商业版本都是 SVR4 和 4.4BSD 的变体，并且都趋向于遵循某些通用标准，诸如 IEEE 的 POSIX（Portable Operating Systems based on Unix 基于 Unix 的可移植操作系统） 和 X/Open 的 CAE（Common Applications Enviroment，公共应用环境）。

现有标准仅仅指定了应用程序编程接口（application programming interface, API），即一个已经定义好的环境（用户程序应当运行在该环境下）。因此，这些标准并没有对内核的内部设计施加任何限制。

为了定义一个通用用户接口，类 Unix 内核通常采用相同的设计思想和特征。在这一点上，Linux 和其他的类 Unix 操作系统是一样的。因此，阅读本书并研读 Linux 内核也有助于你理解其他 Unix 变体。

Linux 内核 2.6 版本的目标是遵循 IEEE POSIX 标准。这意味着在 Linux 系统下，很容易编译和运行目前现有的大多数 Unix 程序，只需少许或根本无需为源代码打补丁。此外，Linux 包括了现代 Unix 操作系统的全部特点，诸如虚拟存储、虚拟文件系统、轻量级进程、Unix 信号量、SVR4 进程间通信、支持对称多处理器（Symmetric Multiprocessor，SMP）系统等。

Linus Torvalds 在写第一个内核的时候，参考了Unix 内幕方面一些经典的书，比如 Maurice Bach 的《The Design of the Unix Operation System》（Prentice Hall, 1986）。实际上，Linux 始终对 Bach 的书（即 SVR2）中所描述的 Unix 基准有些偏爱。但是，Linux 没有拘泥于任何一个特定的变体，相反，它尝试采纳了几种不同 Unix 内核中最好的特征和设计选择。

Linux 与一些著名的商用 Unix 内核到底如何竞争，下面给予描述：

*单块结构的内核（Monolithic kernel）*
&emsp;&emsp;它是一个庞大、复杂的自我完善（do-it-yourself）程序，由几个逻辑上独立的成分构成。在这一点上，它是相当传统的，大多数商用 Unix 变体也是单块结构。（一个显著的例外是 Apple 的 Mac OS X 和 GNU 的 Hurd 操作系统，它们都是从卡耐基 - 梅隆大学的 Mach 演变而来的，都遵循微内核的方法。）

*编译并静态连接的传统 Unix 内核*
&emsp;&emsp;大部分现代操作系统内核可以动态地装载和卸载部分内核代码（典型的例子如设备驱动程序），通常把这部分代码称为模块（Module）。Linux 对模块的支持是很好的，因为它能自动按需装载或卸载模块。在主要的商用 Unix 变体中，只有 SVR4.2 和 solaris 内核有类似的特点。
