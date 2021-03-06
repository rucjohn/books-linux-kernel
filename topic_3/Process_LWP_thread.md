# 进程、轻量级进程和线程

术语 “进程” 在使用中常有几个不同的含义。在本书中，我们遵循 OS 教科书中的通常定义：**进程是程序执行时的一个实例**。你可以把它看作 **充分描述程序已经执行到何种程度的数据结构的汇集**。

进程类似于人类：它们被产生，有或多或少有效的生命，可以产生一个或多个子进程，最终都要死亡。一个微波的差异是进程之间没有性别差异 ------ 每个进程都只有一个父亲。

从内核观点看，进程的目的就是担当分配系统资源（CPU 时间、内存等）的实例。

当一个进程创建时，它几乎与父进程相同。它接受父进程地址空间的一个（逻辑）拷贝，并从进程创建系统调用的下一条指令开始挂靠与父进程相同的代码。尽管父子进程可以共享含有程序代码（正文）的页，但是它们各自有独立的数据拷贝（栈和堆），因此，**子进程对一个内存单元的修改对父进程是不可见的（反之亦然）**。

尽管早期  Unix 内核使用了这种简单模式，但是，现代 Unix 系统并没有如此使用。它们支持多线程应用程序 --- 拥有很多相对独立执行流的用户程序共享应用程序的大部分数据结构。在这样的系统中，**一个进程由几个用户线程（或简单地说，线程）组成，每个线程都代表进程的一个执行流**。现在，大部分多线程应用程序都是由 *pthread（POSIX thread）* 库的标准库函数集编写的。

Linux 内核的早期版本没有提供多线程应用的支持。从内核观点看，多线程应用程序仅仅是一个普通进程。多线程应用程序多个执行流的创建、处理、调度整个都是在用户态进程的（通常使用 POSIX 兼容的 *pthread* 库）。

但是，这种多线程应用程序的实现方式不那么令人满意。例如，假设一个象棋程序使用两个线程：其中一个控制图形化横盘，等待人类选手的移动并显示计算机的移动，而另一个思考棋的下一步移动。尽管第一个线程等待选手移动时，第二个线程应当继续运行，以此利用选手的思考时间。但是，如果象棋程序仅是一个单独的进程，第一个线程就不能简单地发出等待用户行为的阻塞系统调用；否则，第二个线程也被阻塞。相反，第一个线程必须使用复杂的非阻塞技术来确保进程仍然是可运行的。

**Linux 使用轻量级进程（*lightweight process，LWP*）对多线程应用程序提供更好的支持**。两个轻量级进程基本上可以共享一些资源，诸如地址空间、打开的文件等等。只要其中一个修改共享资源，另一个就立即查看这种修改。当然 ，当两个线程访问共享资源时就必须同步它们自己。

实现多线程应用程序的一个简单方式就是把轻量级进程与每个线程关联起来。这样，线程之间就可以通过简单地共享同一内存地址空间、同一打开文件集等来访问相同的应用程序数据结构集；同时，每个线程都可以由内核独立调度，以便一个睡眠的同时另一个仍然 是可运行的。POSIX 兼容的 *pthread* 库使用 Linux 轻量级进程有 3 个例子：
- *LinuxThreads*
- *Native Posix Thread Library (NPTL)*
- IBM 的下一代 Posix 线程包 NGPT（*Next Generation Posix Threading Package*）

POSIX 兼容的多线程应用程序由支持 “线程组” 的内核来处理最好不过。**在 Linux 中，一个线程组基本上就是实现了多线程应用的一组轻量级进程**，对于像 `getpid()`、`kill()` 和 `_exit()` 这样的一些系统调用，它像一个组织，起整体的作用。在配音随后，我们将对其进行详细描述。
