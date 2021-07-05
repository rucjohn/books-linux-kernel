# 进程资源限制

每个进程都有一组相关的资源限制（*resource limit*），限制指定了进程能使用的系统资源数据。这些限制避免用户过分使用系统资源（CPU、磁盘空间等）。Linux 承认以下表 3-7 中的资源限制。

对当前进程的资源限制存放在 `current->signal->rlimit` 字段，即进程的信号描述符的一个字段（参见第十一章 “与信号相关的数据结构” 一节）。该字段是类型的 rlimit 结构的数组，每个资源限制对应一个元素：  
```
struct rlimit {
    unsigned long rlim_cur;
    unsigned long rlim_max;
};
```

- rlim_cur 字段是资源的当前资源限制。例如 `current->signal->rlim[RLIMIT_CPU]` 中 rlim_cur 表示正在运行进程所占用 CPU 时间的当前限制。
- rlim_max 字段是资源限制所允许的最大值。利用 `getrlimit()` 和 `setlimit()` 系统调用，用户总能把一些资源的 rlim_cur 限制增加到 rlim_max。然而，只有超级用户（或更确切地说，具有 `CAP_SYS_RESOURCE` 权能的用户）才能改变 rlim_max 字段，或者把 rlim_cur 字段设置成大于相应 rlim_max 字段的一个值。

**表 3-7：资源限制**
字段名 | 说明
RLIMIT_AS | 进程地址空间的最大数（以字节为单位）。当进程使用 `malloc()` 或相关函数扩大它的地址空间时，内核检查这个值（参见第九章 “进程的地址空间” 一节）
RLIMIT_CORE | 内存信息转储文件的大小（以字节为单位）。当一个进程异常终止时，内核在进程的当前目录下创建内存信息转储文件之前检查 空上值（参见第十一章 “传递信号之前所执行的操作” 一节）
RLIMIT_CPU | 进程使用 CPU 的最长时间（以秒为单位）。如果进程超过了这个限制，内核就向它发一个 `SIGXCPU` 信号，然后如时进程还不终止，再发一个 `SIGKILL` 信号（参见第十一章）
RLIMIT_DATA | 堆大小的最大值（以字节为单位）。在扩充进程的堆之前，内核检查这个值（参见第九章 “堆的管理” 一节）
RLIMIT_FSIZE | 文件大小的最大值（以字节为单）。如果进程试图把一个文件的大小扩充到大于这个值，内核就给这个进程发 `SIGXFSZ` 信号
RLIMIT_LOCKS | 文件锁的最大值（上当是非强制的）
RLIMIT_MEMLOCK | 非交换内存的最大值（以字节为单）。当进程试图通过 `mlock()` 或 `mlockall()` 系统调用锁住一个页框时，内核检查这个值（参见第九章 “分配线性地址区间” 一节）
RLIMIT_MSGQUEUE | POSIX 消息队列中的最大字节数（参见第十九章 “POSIX 消息队列” 一节）
RLIMIT_NOFILE | 打开文件描述符的最大数。当打开一个新文件或复制一个文件描述符时，内核检查这个值（参见第十二章）
RLIMIT_NPROC | 用户能拥有的进程最大数（参见本章 “clone()、fork() 及 vfork() 系统调用” 一节）
RLIMIT_RSS | 进程所拥有的页框最大数（目前是非强制的）
RLIMIT_SIGPENDING | 进程挂起信号的最大数（参见第十一章）
RLIMIT_STACK | 栈大小的最大值（以字节为单）。内核在扩充进程的用户态堆栈之前检查这个值（参见第九章 “异常处理” 一节）

大多数资源限制包含值 RLIMIT_INFINITY(Oxffffffff)
