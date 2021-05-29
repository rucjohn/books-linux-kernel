## 硬件的依赖性

Linux 试图在硬件无关的源代码与硬件相关的源代码之间保持清晰的界限。为了做到这点，在 *arch* 和 *include* 目录下包含了23个子目录，以对应 Linux 所支持的不同硬件平台。这些平台的标准名字如下：

* *alpha*  
HP 的 Alpha 工作站，最早属于 Digital 公司，后来属于 Coompaq 公司，现在不再生产。  
&emsp;

* *arm, arm26*
基于 ARM 处理器的计算机（如 PDA）和嵌入式设备。  
&emsp;

* *cris*  
Axis 在它的瘦服务器中使用的 “代码精简指令集”（Code Reduced Instruction Set）CPU，用在诸如 Web 摄像机或开发主板中。  
&emsp;

* *frv*
基于 Fujitsu FR-V 系列微处理器嵌入式系统。  
&emsp;

* *h8300*  
Hitachi h8/300 和 h8S 的 8 位和 16 位 RISC 微处理器。  
&emsp;

* *i386*  
基于 80x86 微处理器的IBM 兼容个人计算机。  
&emsp;

* *ia64*  
基于 64 位 Itanium 微处理器的工作站。  
&emsp;

* *m32r*  
基于 Renesas M32R 系列微处理器的计算机。  
&emsp;

* *m68k, m68knommu*  
基于 Motorola MC680x0 微处理器的个人计算机。  
&emsp;

* *mips*  
基于 MIPS 微处理器的工作站，如 Silicon Graphics 公司销售的那些工作站。  
&emsp;

* *parisc*  
基于 HP 公司 HP 9000 PA-RISC 微处理器的工作站。  
&emsp;

* *ppc, ppc64*  
基于 Motorola-IBM PowerPC 32 位和 64 位微处理器的工作站。  
&emsp;

* *s390*  
IBM ESA/390 及 zSeries 大型机。  
&emsp;

* *sh, sh64*  
基于 Hitachi 和 STMicroelectronics 联合开发的 SuperH 微处理器的嵌入式系统。  
&emsp;

* *sparc, sparc64*  
基于 Sun 公司 SPARC 和 64 位 Ultra SPARC 微处理器的工作站。  
&emsp;

* *um*  
用户态的 Linux --- 一个允许开发者在用户态下运行内核的虚拟平台。  
&emsp;

* *v850*  
集成了基于 Harvard 体系结构的 32 位 RISC 核心的 NEC V850 微控制器。  
&emsp;

* *x86_64*  
基于 ADM 的 64 位微处理器的工作站，如 Athlon 和 Opteron，以及基于 Intel 的 ia32e/EM64T 64 位微处理器的工作站。  
&emsp;
