# 前言

在1997年春季的第一学期，我们讲授了基于 Linux 2.0 操作系统这门课程。其主导思想是鼓励学生阅读源代码。为了达到这一目的，我们按小组分配项目，这些项目对内核进行修改并对所修改的版本进行测试。对于诸如任务切换和任务调度这样一些 Linux 的主要特点，我们也为学生写下课程笔记。

除了这些工作，还有来自 O'Reilly 编辑 Andy Oram 的很多支持，这就促成了《深入理解 Linux 内核》这本书的第一版，那时是 2000 年底，该版涵盖了 Linux 2.2 以及对 Linux 2.4 的一些展望。这本书的成功鼓励我们继续沿这一思路走下去，在 2002 年底，我们完成了涵盖 Linux 2.4 的第二版。现在你看到的第三版则涵盖了 Linux 2.6。

在以往所经历的一样，我们这次又阅读了数千行的代码，并努力搞清其含义。在做了所有这些工作以后，可以说我们的努力是完全值得的。我们学到很多你无法从书本中找到的东西，因此我们希望自己已经成功地在后面的内容中涵盖了这些信息。

## 本书的读者对象

如果你对 Linux 如何工作、其性能又为什么如此之高怀有强烈的好奇心，你将会从这里找到答案。阅读本书之后，你会通过上千行代码找到自己的方式来区别重要数据结构和次要数据结构的不同，简而言之，你将成为一名真正的 Linux 高手。

可以把我们的工作看作是畅游 Linux 内核的向导：我们讨论了在内核中使用的很多重要的数据结构、算法和编程技巧。在很多例子中，我们逐行讨论了有关代码片段。当然，你手头应当备有 Linux 源代码，你还应当乐于花一些功夫去解读那些为简洁起见而未完整描述的函数。

另一方面，如果你想更多地了解现代操作系统中主要设计问题，那么本书将提供颇有价值的见解。本书不是专门针对系统管理员或编程人员的，而是主要针对那些想探究机器内部到底是如何工作的人们的！与任何好向导一样，我们试图透过现象看其本质。我们还提供了背景材料，例如主要特点的历史及使用它们的理由。

## 材料的组织

开始写这本书时，我们面临重大的抉择：是应该涉及特定的硬件平台，还是跳过硬件相关的细节集中于纯粹与硬件无关的内核部分？

有关 Linux 内核内幕的其他书选择后一种方式；因为下述理由，我们决定采用前一种方式：
- 高效的内核充分利用硬件可利用的特点，诸如寻址技术、高速缓存（cache）、处理器异常（exception）、专用指令、处理器、控制寄存器等等。如果我们想使你相信，内核在执行一个特殊的任务时确实工作得相当好，那我们必须首先告诉你内核工作在一个什么样的硬件平台上。
- 即使 Unix 内核大部分源代码是独立于处理器的，并且用C语言编写，但也有少数重要的部分是用汇编语言编写的。因此，为了充分理解内核，就需要学习一些与硬件打交道的汇编语言片段。

当涉及硬件特征时，我们策略非常简单：对全部由硬件驱动的特征给予简单描述，而对于需要软件支持的特征给予详细描述。事实上，我们感兴趣的是内核的设计而不是计算机的体系结构。

我们下一步就是选择所描述的计算机系统。尽管 Linux 目前已运行在很多种类的个人计算机（PC）和工作站上，但我们决定把主要精力放在非常流行的且便宜的 IBM PC 兼容机上，其中微处理器是 Intel 80x86 及 PC 中所支持的一些芯片。在以后的章节中，术语“Intel 80x86 微处理器”将表示 Intel 80386、80486、Pentium、Pentium Pro、Pentium II、Pentium III、Pentium 4 微处理器。在少数情况下，对于特殊的模型会给出明确的说明。

在研究 Linux 各组件时，我们还必须对所遵循的顺序做出选择。我们尝试的是一种自底向上的方式：从硬件相关的主题开始，以完全与硬件无关的主题结束。事实上，在本书的初始部分我们将多次引用 Intel 80x86 微处理器，而其他部分相对来说与硬件无关。不过，第十三章和第十四章是个种例外。实际上，遵循自底向上的方法并不像看起来那样简单，这是因为存储器管理、进程管理和文件系统这几部分相互渗透；少数向前引用（即引用还待解释的主题）是不可避免的。

每章以所涵盖内容的理论概述开始，然后按自底向上的方式组织材料。我们以描述每章内容所需要的数据结构开始，然后，我们通常从描述最低级功能移到描述较高级功能，最后说明用户应用程序所发出的系统调用是如何得到支持的。

## 描述级别

支持各种体系结构的 Linux 源代码包含在 14000 多个C语言和汇编语言的文件中，这些文件存放在大约1000个子目录中。源代码大约由600万行代码组成，占 230MB 以上的磁盘空间。当然，这本书只涵盖源代码非常少的一部分。考虑一下你所读的书的全部源代码只占不到 3MB 的磁盘空间，就能想像出 Linux 源代码有多么的庞大了。因此，即使不对源代码进行解释，只列出所有的代码，75本书也写不完！

因此，我们必须对阐述的内容做出选择，我们的决策大致情况如下：
- 我们相当全面描述了进程管理和内存管理。
- 我们涵盖了虚拟文件系统以及 Ext2 和 Ext3 文件系统，不过，很多功能仅仅是提及而已，并没有对其代码进行详尽描述；我们不讨论 Linux 所支持的其他文件系统。
- 我们描述了占内核 5% 左右的设备驱动程序；但仅涉及有关的内核接口，而并不试图分析每个具体的驱动程序。

本书描述的是 Linux 内核 2.6.11 的正式版，可以从 http://www.kernel.org 试点下载。

注意，很多 GUN/Linux 发行版都对正式版内核进行了修改，以实现新的特点或提高其效率。在少数情况下，由你喜爱的发布版所提供的源代码可能与本书所描述的源代码有很大的不同。

在很多实例中，我们展示了以易读但低效的方式重写的原始代码的片段。这些代码出现在关键时间上，在这些点上，程序片段是用手工优化的C语言和汇编代码混合在一起编写的。再次声明，我们的目的是为研究 Linux 原始的人提供一些帮助。

在讨论内核代码时，我们常常同时描述 Unix 程序员熟悉的很多基础知识（共享内存和映射内存、信号、管道、符号链等等），也许他们听说过这些内容，但可能还想进一步了解。

## 本书概述

为了对全书有一个大体了解：
- 第一章 “绪论” 中对 Unix 内核内部结构给出了一个般性描述，并说明 Linux 如何与其他著名的 Unix 系统展示竞争。
- 任何 Unix 内核的核心都是内存管理。第二章 “内存寻址” 说明 Intel 80x86 处理器包含有对内存数据进行寻址的特殊电路，并解释 Linux 如何充分利用它们。
- 进程是 Linux 所提供的一种基本抽象，这在第三章 “进程” 中进行介绍。在这一章，我们了解释了每个进程如何在非特权的用户态下运行，又如何在有特权的内核态下运行。用户态与内核态之间的转换只能通过已建立的所谓中断和异常处理硬件机制实现，这些内容将在第四章 “中断和异常” 中介绍。
- 在很多情况下，内核必须处理来自不同设备和处理器的突发性中断。因此，就需要同步机制，以便所有这些请求能由内核以交错方式去处理：这些将在第五章 “内核同步” 中进行讨论，其中既涉及单处理器系统，也涉及多处理器系统。
- 定时中断使 Linux 能够处理已经经历的时间，是一种重要的中断类型；更详细的内容将在第六章 “定时测量” 中介绍。
- 第七章 “进程调度” 说明 Linux 如何轮流执行系统中的每个活动进程，以便所有的进程都能顺利地执行完。
- 接下来我们再一次集中讨论内存。第八章 “内存管理” 描述用严重处理系统中最宝贵的资源 --- 可用内存（当然除了处理器） --- 所需要的复杂技术。这种资源必须同时满足 Linux 内核和用户应用程序的需要。第九章 “进程地址空间” 讲述内核如何处理应用程序对内存发出的“贪婪（greedy）”请求。
- 第十章 “系统调用” 说明用户态下运行的进程如何对内核发出请求；第十一章 “信号” 描述进程如何给其他进程发送同步信号。现在我们准备进入另一个实质性的主题，即 Linux 如何实现文件系统。很多章节涉及到这个主题。第十二章 “虚拟文件系统” 介绍支持很多种不同文件系统的通用层。某些 Linux 文件比较特殊，这是因为它们能提供到达硬件设备的陷阱门；第十三章 “I/O体系结构和设备驱动程序” 以及第十四章 “块设备驱动程序” 进一步考察了这些特殊的文件和相应的硬件设备驱动程序。
- 另一个值得考虑的问题是磁盘访问时间，第十五章 “页高速缓存” 说明灵活地利用 RAM 可以减少磁盘的访问时间，因而极大地提高系统的性能。在前几章内容的基础上，我们将在第十六章 “访问文件” 中讨论用户应用程序如何访问常规文件。在第十七章 “回收页框” 完成对 Linux 内存管理的讨论，并说明 Linux 确保总是有足够的内存所使用的技术。第十八章 “Ext2和Ext3文件系统” 是讨论文件系统的最后一章，阐述了 Linux 最常用的文件系统，即 Ext2 及最新改进的 Ext3。
- 最后两章结束我们对 Linux 内核的详细游览：第十九章 “进程通信” 介绍通信机制而不是用户态进程使用的信号；第二十章 “程序的执行” 说明用户应用程序是如何执行的。
- 最后，但必不可少的，就是附录：附录一 “系统启动” 大致描述了 Linux 如何启动；而附录二 “模块” 描述怎样动态地重新配置正在运行的内核，即按需增加或删除有关功能。源代码索引（Source Code Index）包含了在本书中引用的所有 Linux 符号；你将在这里找到宣言每个符号的 Linux 文件名，以及对这个符号进行解释的正文所在的页码。我们认为你会发现它非常方便实用。

## 背景知识

除了一些C语言编程技巧和汇编语言的知识外，理解这些内容不需要任何先决条件。

## 排版约定

下面是本书在英文字体上的两个约定：

等宽字体（Constant Width）  
&emsp;用来说明代码文件的内容或命令输出的内容，也表示出现在代码中的源代码关键字。

斜体（*Italic*）  
&emsp;用来说明文件名、目录名、程序名、命令名、命令行选项和URL

## 如何与我们联系

请把有关本书的评论和问题告知出版社：

美国：  
&emsp;O'Reilly Media, Inc  
&emsp;1005 Gravenstein Highway North  
&emsp;Sebastopol, CA 95472

中国：  
&emsp;北京市海滨区知春路49号希格玛公寓B座809室（100080）  
&emsp;奥莱理软件（北京）有限公司
  
本书的网页上列出了勘误表、示例和任何额外的信息。可登录以下网址查询：  
&emsp;*http://www.oreilly,com/catalog/understandlk*  
&emsp;*http://www.oreilly.com.cn/book.php?bn=978-7-5083-5394-4*

如果想要发表关于本书的评论或咨询有关技术问题，请发邮件至：  
&emsp;*bookquestions@oreilly.com*  
&emsp;*info@mail.oreilly.com.cn*

关于图书、会议、资源中心和O'Reilly网络的更多信息，请查看我们的站点：  
&emsp;*http://www.oreilly.com*  
&emsp;*http://www.oreilly.com.cn*

## 致谢

如果没有罗马大学 Tor Vergata 分校工程学校很多学生的尽力帮助，这本书不可能完成，他们不但上了这门课，还试着解读了 Linux 内核的讲稿。他们不懈的努力紧紧抓住了源代码的真正含义，使得我们对讲稿不断改进，并改正了很多错误。

Andy Oram 是 O'Reilly Media 的优秀编辑，非常值得信任。他是 O'Reilly 第一位对此项目给予信任的人，他花费了很多时间和精力阅读我们的初稿。他提出的很多建议使本书的可读性更强，同时他还写出了不少出色的介绍性段落。

还有一些颇具声望的技术审校，他们非常认真地阅读了本书的内容。第一版的审校人为（名字按字母顺序排列）：Alan Cox、Michael Kerrisk、Paul Kinzelman、Raph Levien 和 Rik van Riel。

第二版的审校人为Erez Zadok、Jerry Cooperstein、John Goerzen、Michael Kerrisk、Paul Kinzelman、Rik van Riel 和 Walt Smith。

第三版的审校人为Charles P. Wright、Clemens Buchacher、Erez Zadok、Raphael Finkel、Rik van Riel 和 Robert P. J. Day。他们的建议，加之世界各地很多读者的参与，帮助我们去掉了很多错误和不准确的地方，使得本书更具有说服力。

<p align="right">- Dainel P. Bovet</p>
<p align="right">Macro Cesati</p>
<p align="right">July 2005</p>
