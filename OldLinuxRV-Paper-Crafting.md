



# Contents

[TOC]



# 1 Introduce





## 跨度大, 资料多, 术语的额外说明

#### Virtual Address and Virtual Memoey

本文中, Virtual Address指的是在processor’s addressable memory space ( [^03] Volume3 3.1 )中寻找寻址的地址。Virtual Memoey指的是processor’s addressable memory space这一空间。

在这一定义下, IA-32中Virtual Address是linear address。

关于这两个术语, 常见的参考资料中有以下观点:

- 在[^04] P200 和[^08]P153 中都认为不能独立转换为物理地址的抽象地址都被称为Virtual Address。这里的不能独立转换指的是需要段机制的参与[^WBL]。但无论是i386手册[^10], 还是IA-32手册[^03]都没有这样的定义。

- i386手册[^10]P15,  [^08]P89、P153 用Virtual Address Space / Virtual Memory来表达了内存管理系统的潜在能力。

  例如, i386手册认为"The processor can address up to four gigabytes of physical memory and 64 terabytes ($2^{46}$ bytes) of virtual memory."它的计算方法是, 段选择子(段选择符)中13位用于在GDT或LDT中索引一个段选描述符, 由TI位来具体选择使用GDT还是LDT, 同时每个段描述符又可以指向一个页表, 每个页表可以寻址4GiB的空间[^WBL]。所以有${2}\times{2^{13}\times{2^32}={2^{46}}}$ 的虚拟地址空间。这一计算方式甚至忽略了GDT的首个段描述符必须为空, LDT必须由GDT间接引用这些影响。是非常理想化的计算方式。

- 在https://foldoc.org/Virtual+address与https://foldoc.org/MMU中认为, 被MMU转化为物理地址/内存的地址被称为Virtual Address / Memory。且将MMU这一概念直接与PMMU划等号。片面的将由分页机制转化为物理地址的地址称为虚拟地址。

- 在[^08] P154中, 认为"Virtual Memory是指计算机呈现出要比实际拥有的内存大得多的内存量。"在IA-32手册中没有这样的定义的描述, 但[^03] Volume3 3.1中描绘了一种"能力", 即"Paging supports a “virtual memory” environment where a large linear address space is simulated with a small amount of physical memory (RAM and ROM) and some disk storage."。

  ~~在支持PSE-36的IA-32处理器中, 处理器实际能寻址的范围只有32bits, 而物理地址的寻址却能达到36bits。~~

在[^01]中, Linus先生并未指明Virtual Address和Virtual Memory的定义。现代Linux中非分页机制仅保证体系结构的要求[^WBL], 但由于本文要讨论使用了分段机制的早期Linux, 不对Virtual Address作出明确的定义会产生歧义。



#### 进程相关

The term “process” is often used with several different meanings([^05]P79). Task, Processes, Lightweight Processes, and Threads, etc这些术语在特定的语境下也有可能指的是“process”这一术语。

From the kernel’s point of view, the purpose of a process is to act as an entity to which system resources (CPU time, memory, etc.) are allocated([^05]P79).更简单的来说, 一个进程享有一个独特的Process Descriptor, 它是一个`struct task_sturct`的数据结构。(正是因为这个命名导致task和process在大部分情况下是同义词)

早期内核在创建一个新进程时, 子进程只能获得父亲进程拥有的资源的深拷贝( COW机制延后了深拷贝的时机, 但最终结果还是深拷贝 )。The multiple execution flows of a multithreaded application were created, handled, and scheduled entirely in User Mode([^05]P80).

后来, Linux使用了Lightweight Processes的概念来, 不同轻量进程依旧享有不同的进程描述符, 两个轻量级进程间可以共享一些资源。实现多线程程序时, 将轻量进程与每个线程关联起来。

但复杂的是, 现代Linux中, 存在大量与thread这一名词相关的数据结构。其中与进程相关的最重要的是`thread_info`。进程描述符中必然直接或间接包含一个`thread_info`结构, `thread_info`和`task_sturct`具体如何关联与体系结构相关。在某些体系结构中, `thread_info`结构是进程描述符的第一个成员。在这些结构中, 内核会用一个寄存器存放当前进程的 thread_info 结构体的地址，通过这个寄存器既可以得到 thread_info 结构体的地址，也可以得到进程描述符的地址([^07]P27)。

这让提前指明Task, Processes, Lightweight Processes, and Threads的定义变得不可能, 请读者注意。



#### 页表这一术语的歧义

Page table 一词会产生歧义。Page table这一术语可以指 a data structure used by a virtual memory system in a computer to store mappings between virtual addresses and physical addresses. 

在Linux术语中讨论Multilevel page table时, 为每一级页表结构进行了命名,  page table 又是其中的一级的名称。

在RISC-V特权级手册中, 并未给每一级结构命名, 而是都统称为page table。

为此我们额外引入两种额外的描述页表的方法。

两种方式都将虚拟地址从MSB到LSB分为三个主要部分: 符号扩展, 分级索引号, 页内偏移。

其中符号扩展部分可能不存在, 或者根据体系结构的不同会有不同的规定。

第一种方法, 受到了[^01]中的启发。[^01]中分级索引号被称为Index Level, 从L1开始, 由MSB至LSB方向逐级升高。假使当前为Index L$x$, 则我们称当前页表结构为第$x$级页表。例如, x86的页表目录, 在这种方式下被称为第一级页表

第二种方式, 出自[^07], 也是xv6-riscv在编码中使用的描述方式。分级索引号被称为VPN ([^WBL] RV页表术语说法), 从VPN[0]开始, 由LSB至MSB方向逐级升高。这种描述方法一般会出现在本文的代码实现中。

本文涵盖了Linux不同的发展时期的资料, 对于页表的描述时可能会出现的歧义, 将在这三种描述方式选择合适的进行说明以避免歧义



# 2 体系结构相关

~~本文将体系结构相关的部分分为四个重点: Memory Layout、Entry Path and Exit Path、Context Switch和Kernel and Process Initialization。~~

... 

## Comparing the ISAs of RISC-V and x86



### Modular vs. Incremental ISAs (模块化ISA和增量型ISA)

x86是一种增量型ISA, RISC-V是一种模块化ISA. [^11]中比较了他们的优劣, 本文仅讨论两种ISA的同异之处。

RISC-V的ISA由一个base ISA (RV32I or RV64I)和各种optional standard extensions组成。The convention is to append the extension letters to the name to indicate which are included. For example, RV32IMFD adds the multiply (RV32M), single-precision floating point (RV32F), and double-precision floating point extensions (RV32D) to the mandatory base instructions (RV32I).

字母G有着特殊的含义, 它并不单指某一种可选的扩展, 而是用于标记“general-purpose”的ISA。它包含了a combination of a base ISA (RV32I or RV64I) plus selected standard extensions (IMAFD, Zicsr, Zifencei)。( Zicsr, Zifencei来自RISC-V的另一种扩展命名方式, 具体请参考 [^15] ISA Extension Naming Conventions )

Some ISA extensions depend on the presence of other extensions, e.g., “D” depends on “F” and “F” depends on “Zicsr”. These dependences may be implicit in the ISA name: for example, RV32IF is equivalent to RV32IFZicsr, and RV32ID is equivalent to RV32IFD and RV32IFDZicsr.

本文主要的移植目标就是RV64G。

x86作为一种增量型ISA, 从8086发展至今, 最近于2023年提出x86s ISA以简化其指令集[^16]。正如[^11]所说, x86架构有着“沉重的历史包袱”, 这可以从( [^WBL]  有关启动部分 16-bit...)看出, 但是有些时候我们可以借用x86的前辈处理器的思想简化我们的研究( [^WBL] 用8086解释段结构)。本文主要的讨论的是80386 (i386)处理器。早期Linux中实际上提及了80486处理器与80387数学协处理器芯片, 但我们不会过多讨论 ( [^WBL] 数学处理器部分)。



### Privilege Levels

x86的保护模式的权级从 Level 0 至 Level3 权限逐级降低([^03] Volume3  2.5 、5.5)。由于intel在手册中用使用了protection rings的概念, 一些文献也将其称为4层Ring, 标号规则与前面所述的一致。本文仅讨论用于内核态的 Level 0 与用于用户态的 Level3 。

> The processor’s segment-protection mechanism recognizes 4 privilege levels, numbered from 0 to 3. The greater numbers mean lesser privileges.

RISC-V的权级从 Level 0 (level of 0) 至 Level3 权限逐级升高([^09])。虽然设计了4个权级, 但权级为3的M模式是唯一一个所有标准 RISC-V 处理器都必须实现的特权模式。

在RISC-V的设想中, 实现不同数量的权级能实现不同的作用。

| Number of levels | Supported Modes   | Intended Usage                              |
| ---------------- | ----------------- | ------------------------------------------- |
| 1                | M                 | Simple embedded systems                     |
| 2                | M, U              | Secure embedded systems, RTOS               |
| 3                | M, S, U           | Systems running Unix-like operating systems |
| 4                | M, HS, VS, VU / U | Support Type-2 hypervisor execution         |

不同权级模式Abbreviation和对应的Level与名称之间的关系。

| Level | Encoding | Name                           | Abbreviation |
| ----- | -------- | ------------------------------ | ------------ |
| 0     | 00       | User/Application               | U            |
| 0     | 00       | Virtualized User               | VU           |
| 1     | 01       | Supervisor                     | S            |
| 1     | 01       | Virtualized Supervisor         | VS           |
| 2     | 10       | Hypervisor-Extended Supervisor | HS           |
| 3     | 11       | Machine                        | M            |

在RISC-V中, 内核并不运行在最高权级中。这与x86不同, 运行在RISC-V非最高权级的内核并不能拥有最高权限。包括访问CPU的编号与属性、响应定时器中断、核间通讯(IPI)、异常处理等都不能直接处理。而是依赖于The higher privilege software providing SBI (Supervisor Binary Interface) interface.例如在M、S、U情况下, 运行在M-mode的SBI为S-mode的操作系统提供接口。在使用Hypervisor Extension的情况下, 运行在M态的SBI为HS态的OS or hypervisor提供接口.An HS-mode hypervisor is expected to implement the SBI for its VS-mode guest.

> 曾经的设想中, 为HS提供接口的被称为HBI, 为VS和S提供接口的被称为SBI。但当下术语中, 无论是为HS、VS还是S提供的接口都称为SBI。

SBI可以被认为是一种runtime, 也可以被更简单的认为是一种固件, 如QEMU甚至直接称其为BIOS。

本文仅讨论使用M、S、U三种权级的情况。大多数情况下, 我们会忽略M态和SBI的存在。但在[^WBL]异常处理等章节, 我们会设计到一些SBI的实现, 这有助于我们理解。



### 通用寄存器

x86的寄存器分为3类([^13] 2.3 P29): General registers, Segment registers, Status and instruction registers.其中通用寄存器有8个, 分别为EAX, EBX, ECX, EDX, EBP, ESP, ESI, and EDI。其中instruction registers是instruction pointer register(EIP), the low-order 16 bits of EIP is named IP.而IP is analogous to the program counter (PC) ([^13] P2-7).

本文将RV32I / RV64I这样的Base ISA的从x0到x31寄存器称为RISC-V的通用寄存器 。其中与ARM(32位)架构将R15通用寄存器用于存储PC不同, RISC-V并不将PC寄存器归为通用寄存器, 也不直接将PC暴露给编程者。此外, 使用Control and Status Register (CSR)需要Zicsr扩展。

x86设计者和RISC-V设计者对“通用寄存器”这一术语的理解显然是不同的。x86的设计者认为, general-purpose registers are used primarily to contain operands for arithmetic and logical operations.RISC-V设计者认为的通用寄存器是不存在在某条指令中一定需要某个通用寄存器参与。

从RISC-V设计者的角度来看, x86的8个通用寄存器并不“通用”。x86中的一些通用寄存器是一些指令中有特殊用途的, 例如eax是默认的累加寄存器, ecx用于循环计数等等。事实上, 如果我们追溯i386的历史, 也就是8086会发现, 所有x86的8个通用寄存器在8086中的前身每一个都有特殊的用途。也正因如此, 从ISA的角度, x86的8个通用寄存器都有自己的名字: A for Accumulator, B for Base, C for Count, D for Data, SP for Stack Pointer, BP for Base Pointer, SI for Source Index, DI for Destination Index.

RISC-V将x0寄存器硬连线为0, 当这不仅没有破坏RISC-V寄存器设计的通用性, 反而有所加强。由于 ARM-32 和 x86-32 没 有零寄存器，它们需要通过原生指令实现这些操作。但对于 RISC-V，只需简单地将零寄存器作为其中一个操作数，即可通过 RV32I 指令实现相同操作( [^11cn] P18、P35)。例如: `nop`在RISC-V中的实现为`addi x0, x0, 0`。 再例如`jr rs1` (Jump Register)被实现为`jalr x0, 0(rs1)`, `jalr`会记录转跳的返回位置也就是当前指令的下一条指令, 但由于`jr`不需要记录, 所以将其丢入硬连线为0的x0中。

RISC-V的汇编器通过提供伪指令的形式为汇编语言程序员或编译器开发者提供便利。例如[^WBL]下一节即将提到的`ret`就是一条伪指令, 汇编器将其替换为 `jalr x0, x1, 0`。即, 将下一条指令送入x0寄存器中保存, 将x1寄存器中的值加上符号扩展的offset 0之后送入x1寄存器。请注意区别x86等架构实现的一条"指令"与RISC-V汇编器层面的“pseudoinstruction”的区别——前者是"instruction", 后者是"mnemonic"或更接近一种宏。



### ABI and Calling Conventions

~~操作系统中包含了许多需要使用汇编语言为高级语言( 主要是C语言 )准备运行环境( runtime )、保护上下文、以及汇编与高级语言的配合。这就需要了解ABI和调用规范。~~

~~不同ISA的设计思想会影响ABI和调用规范的设计, 而ABI和调用规范的设计又会影响内核数据结构上的设计。~~

~~IA-32有很多调用规范, 其中Linux内核主要使用的是cdecl调用规范。~~

操作系统中存在使用非编译器生成的汇编语言调用C语言函数和在C语言函数中调用非编译器生成的汇编语言的部分。这涉及到ISA的ABI和调用规范。

x86的调用规范有许多版本, 这与操作系统、所使用的高级语言等等各种因素息息相关。与Linux内核关联紧密的是cdecl调用规范, 它是GCC编译器默认的调用规范。而RISC-V的调用规范从设计之初就与Unix和GCC紧密相连, 它的版本暂时比较唯一。

内核中不同调用规范之间的差异有许多, 在研究内核时需要注意的有以下几点:

第一是参数和返回值的传递方式。

从宏观的来看, 早期的Linux x86-32的cdecl调用规范在传值的时候主要使用栈, x86-64和RISC-V指令集则更偏向于使用寄存器来传递参数。一开始Linux内核对于x86-32架构严格使用cdecl调用规范。但随着内核的发展, 对于x86-32架构开始使用x86-32下GCC中的特殊属性`__attribute__((regparm(n)))`来解决不同情况下的寄存器传值问题, 其中, n代表用于传值的。此变化可以从对应的linkage.h头文件中看出。需要额外说明的是, 在Linux 2.6中的一段时间, IA-32曾引入名为fastcall的宏, 其本质就是`__attribute__((regparm(3)))`。请区别这个宏名与名为fastcall的调用规范之间的区别。

有些参考文献如[^08]认为使用栈来传值意味着callee能拥有更多比实际caller传递过来的参数。[^08]中的举例Linux 0.12中的`_sysfork()`函数, 它自身认为自己有18个参数, 而距离它较近的汇编函数在调用它前后仅压入和弹出了5个参数。但本文认为这仅仅是利用cdecl下用于传参的栈由caller cleanup( 或称“由caller负责栈平衡操作” )的一种技巧。

第二是各个寄存器在调用函数时的作用。

展开来说涉及到寄存器在调用前后是否一致, 或者说某个寄存器是callee-save还是caller-save。以及ABI别名。

首先讨论ABI别名。在通用寄存器一节提到了x86主要的8个通用寄存器的命名本质上来自8086时期的寄存器的作用。而对于RISC-V的通用寄存器来说, 他们的别名则全部来自ABI与调用规范。RISC-V的ABI别名分别是: zero for hard-wired zero, ra for Return Address, SP for Stack Pointer, GP for Global Pointer, TP for Thread Pointer。剩下的寄存器分为Temporaries、Saved register和Functions arguments.其中s0同时兼顾frame pointer, 所以也能表示为fp, a0和a1兼顾return values.如果按RISC-V通用寄存器原始顺序来看, 剩下的三大类寄存器交错出现, 这是因为RISC-V设计了一种使用更少寄存器的方案, 在此本文不过多赘述。

其中zero, gp, tp 和 fp需要额外说明。

首先, 是fp寄存器, RISC-V不强求使用fp去维护函数调用栈, 但是fp指针和sp指针配合对unwind是有作用的。也是处于s0/fp有可能作为栈指针被使用, U-Boot启动时在保存由a1传入的参数时, 没有保存在s0中, 而是保存在s1中。

zero, gp, tp即不由caller维护也不由callee维护。zero由于硬布线零不需要维护。 gp作为链接器松弛的作用, 它的加载需要特殊操作[^TBF]==毕设这块直接省略==。tp指针在设计上被用于TLS (Thread-Local Storage), 这体现在RISC-V的测试用例中( https://github.com/riscv-software-src/riscv-tests/blob/master/benchmarks/common/crt.S#L119 )。但是在实践中, tp寄存器往往直接或间接指向当前的HART ID( ==RISC-V术语, 毕设此处省略== )。

tp储存hartid——xv6-riscv

tp间接储存hartid —— openSBI	tp指向sbi_scratch

https://github.com/riscv-software-src/opensbi/blob/d4d2582eef7aac442076f955e4024403f8ff3d96/include/sbi/sbi_scratch.h

tp间接储存hartid —— Linux	tp指向thread_info, 其中cpu间接映射到hartid





## Memory Layout

> 相关名词搭配:
>
> - ( Virtual or Physical ) Memory Layout	见2.1.1标题来源
> - ( Virtual or Physical ) Address Space	见[^01]和2.1.5标题来源
> - Virtual Address, Physical Address		见[^01]
> - kernel memory, user memory		见2.1.5标题来源
> - memory management				见[^01]
>
> 
>
> 特别注意:
>
> - VMA ( Virtual Memory Area ) 一般特别用于描述https://www.oreilly.com/library/view/linux-device-drivers/9781785280009/4759692f-43fb-4066-86b2-76a90f0707a2.xhtml
> - VAS ( Virtual address space ) wiki: https://en.wikipedia.org/wiki/Virtual_address_space
>   - 某些情况下仅指user的部分, 例如https://www.ibm.com/docs/en/zos-basic-skills?topic=storage-what-is-address-space
>   - 微软的手册认为VAS中应分为两个部分 https://learn.microsoft.com/en-us/windows/win32/memory/virtual-address-space



Memory Layout 问题可以被分为 Virtual Memory Layout 和 Physical Memory Layout 两个子问题。自然地, 我们会讨论将Virtual Address 映射到 Physical Address的方法([^01]P25)。需要注意的是, 现代的Linux已经支持不使用MMU[^TBC], 但是我们将不讨论这种情况。

在讨论 Virtual Memory Layout 问题时, 我们会讨论到 kernel memory 与 user memory 的划分(split)问题。这其中会涉及到权级相关问题。









### Virtual Memory Layout on 现代 Linux

> - 标题名命名来自https://docs.kernel.org/arch/riscv/vm-layout.html 《Virtual Memory Layout on RISC-V Linux》

总的来说, 现代Linux中, 拥有内核态权限的内存位于虚拟地址的高位, 而用户态权限



从布局上看, kernel memory在高地址user memory在低地址, 参考[^05cn]P73。[^07]P121。或Linux网页手册。

且仅使用页表/类页表结构。[^01]说明在几种架构采用的类分页机制的几种例子。

| 架构    | 映射方法                                                 |
| ------- | -------------------------------------------------------- |
| 80x86   | page table(段机制仅满足体系要求)                         |
| Alpha   | page table                                               |
| PowerPC | hash table                                               |
| MIPS    | not have any architecture-specified page tables, BUT TLB |

在此我们讲问题简化为仅讨论页表机制。





- 现代Linux分页机制

  Linux 0.01~1.0的时代, x86-32手册的术语来说, 整个分页机制分为页表目录、页表、页内偏移三个部分, 被视为一个2级页表。

  在[^01] P28中引入了kernel virtual page tables的概念。并使用了三级的页表, 分别为Page Directory、Mid Page Table、Page Table、 Offset in Page  ( [^01] P27 )。如果实际的硬件体系结构的页表级数少于3级, 则会通过机制隐藏其中的一级。

  Linux 2.6中, 内核使用的虚拟页表是一个4级页表[^06] P124~P125。

  Linux 4.11时, 页表被扩展为5级[^07] P219。

  正如[^01] P28中所说:

  > As three levels is enough to map on the order of 40–45 bits of virtual address space, and no current hardware supports more, that was the choice for the Linux kernel virtual machine.

  每一个级数的设计在当时的眼光看都是足够的。但从级数的不断增长来看, 似乎随着技术的发展, 一切都不会**足够**。

  本文在叙述时并未为每一级页表命名, 只是简单的至顶而下成为第?级目录。

  

  

  

- RV中页表相关术语

  RV特权级手册中, 仅使用了page table 与page table entrie (PTE)来描述页表这一数据结构。RV32中每page table consist of $2^{10}$ PTEs, RV64中每page table consist of $2^{9}$ PTEs。每一PTE存储的下一级的映射信息每一份页表仅使用了最高级(被放入`satp`寄存器的)的页表被称为root page table。

  

- 使用页表表项的权级区分U与S

  见[^05cn] P76

- 用(总)页表来区分不同进程(线程)的地址空间

  见[^05cn] P73。同时在部分设计中, 两个不同的进程描述符通过共享一份(总)页表的引用来达到某种资源共享的目的。这一点从Linux 1.00中开始, 之后的资源共享机制变得复杂, 



### Segmentation and Paging in 现代 x86 Linux

> - 标题命名来自[^05]第二章的两个小节标题

简言之, 现代x86-32中, 页表为主, 默认情况下段结构仅用于满足体系结构要求。依旧保留段机制的原因是, 分段在x86体系结构中是必须的。见 [^03] Volume3 3.1。

x86有三种地址相关的术语: 逻辑地址、线性地址与物理地址。他们的关系是: 线性地址 -分段-> 逻辑地址 -分页-> 物理地址。

```mermaid
flowchart LR
    线性地址 --分段--> 逻辑地址 --分页--> 物理地址
```



段机制来自8086, 在8086中, 仅仅是将段寄存器的值乘16(左移4位)然后加上对应的寄存器使用。i386中引入了GDT和LDT间接引用。

现代Linux中段机制所起的作用非常有限, 在设计时会保证线性地址与逻辑地址实际上一致。一方面许多其它的体系结构没有段机制, 他们更经常使用类页表机制。另一方面, x86-32和x86-64代码的合并也是一个原因。尽管x86-64中段机制也是必须开启的, 但是段必须以0为起点[^04]P229。

现代Linux的段机制仅保留了每个CPU一个GDT和一个默认的LDT( [^05cn] P50 ), 在必要的系统调用时才会临时增加LDT。这样的设计第一次出现是在Linux 1.00中。从Linux 1.00开始, 默认情况下, Linux 1.00中所有进程私有的LDT表项都指向`&default_ldt`, 只有在首次调用`sys_modify_ldt`的`write_ldt`操作时, 会分配一个新的空间来存放一个新的LDT表。( 注: 子进程fork拥有非默认LDT的父进程时, 对LDT的复制采取的是重新分配空间的深拷贝 )。

现代Linux中, 每个CPU对应的GDT中用户代码段、用户数据段、内核代码段、内核代码段的Base都为0x0, 长度都为4GiB。Linux 1.00在对GDT的设置上与现代Linux有些不同, 我们在后面接介绍[^WBL]。

现在Linux使用段机制仅为了满足x86体系结构的要求。既不会使用段机制来隔离不同的进程的地址空间, 也不使用段机制来隔离用户态与内核态的地址空间。这些都由分页机制管理。分页机制的使用正如上一小节所说, 使用页表表项的权级区分U与S, 用(总)页表来区分不同进程(线程)的地址空间。



### Segmentation and Paging in 早期 Linux

正如Linus在[^01]中所说, “Linux did not even try to avoid using the x86 features available to those early versions”. 段机制作为x86, 尤其是x86-32的硬件内存管理特色, 在早期的Linux中起关键作用。这个时期的Linux对其它架构的可移植性很弱, 它们仅适用于x86-32体系结构。

我们以Linux 0.01~1.00中的几个版本作为分析对象。

#### 段机制

在介绍这个时期Linux的段机制时, 我们必须从线性地址和逻辑地址两种视角来说明问题。~~此外Linux 0.01~0.99中的0号进程和1号进程的长度都是640KiB, 为了简洁性, 后面的介绍中我们不会特别提及这一点。~~

从线性地址地址角度来看, 由于分段机制, 无论是出于用户态的地址空间还是内核态的地址空间都是从0开始的。由于分段机制仅影响了对应线性地址和逻辑地址的起始点(base 地址), 所以他们的限长是统一的。

从逻辑地址的视角看, 内核地址空间与用户态地址空间的起点与范围的问题, 可划分为几个阶段。
$$
\text{Kernel Version}\begin{cases}

  \text{用户态进程默认使用GDT}\begin{cases}
  \text{Linux 1.00}\\
  \end{cases}\\
  
  \\

  \text{用户态进程默认使用LDT}\begin{cases}

    \text{使用页机制隔离不同进程}\begin{cases}
    \text{Linux 0.99}\\
    \text{Linux 0.98.6}\\
    \text{Linux 0.98}\\
    \text{Linux 0.97.6}\\
    \end{cases}\\
    
    \\

    \text{使用段机制隔离不同进程}\begin{cases}
    
      \text{代码段和数据段长度相等}\begin{cases}
      \text{Linux 0.97}\\
      \text{Linux 0.96c}\\
      \text{Linux 0.95}\\
      \text{Linux 0.12}\\
      \end{cases}\\
      
      \\
    
    	\text{代码段和数据段长度不等}\begin{cases}
      \text{Linux 0.11}\\
      \text{Linux 0.10}\\
      \text{Linux 0.01}\\
      \end{cases}\\

    \end{cases}\\


  \end{cases}\\

\end{cases}\\
$$
Linux 1.00开始用户态进程代码段和数据段默认开始使用GDT选择子, 而非之前版本所使用的LDT选择子。这体现在用户态进程(实际)使用的段选择子上。( `INIT_TASK`宏中装在的段选择子并不是最后实际使用的段选择子, 这点体现在``move_to_user_mode()``宏中 ) ( Linux 1.00中存在很多与LDT相关的代码, 这点我们在后面在做说明 )

在默认使用LDT作为用户态选择子的版本中, 用户态进程起点与范围集中体现在 `INIT_TASK`宏、fork系统调用和exec系统调用的相关代码上。

在这其中, 有三项等价的特征可以用于划分。分别是: 以用户态进程的起点是否与nr无关为划分、以用户态是否拥有3GiB的段长度为划分、以用户态拥是否拥有私有的(总)页表(即页表目录)为划分。这一变化出现在Linux 0.97与Linux0.97.6之间。在Linux 0.97.6及之后的版本, 用户态的段机制便都以0x0为起点, 段长为3GiB, 也因此每个进程都需要自己的私有页表。在Linux 0.97及之前的版本中, 段起点与nr号相关, nr*0x400_0000。(nr为)0号进程的段长为640KiB, 所有fork于它的子进程的段长在由exec系统调用更改前均继承于(nr为)0号进程的长度。而在执行exec时则将代码段和数据段设置成对应的长度。

而exec中为修改的新段limit中, 可以从code_limit和data_limit是否相等来做进一步细节上的区分: 0.01~0.97中数据段的长度都为64MiB, 而代码段的长度设置上却有些区别。0.01、0.10和0.11中, 代码段的长度来由`change_ldt` 函数中的 `text_size`参数决定; 在此之后(直至0.99), 都是直接在`change_ldt`中令code_limit = data_limit = TASK_SIZE(只是TASK_SIZE宏的数值有区别, 如同前文所述, 0.97及之前为64MiB, 0.97.6及之后为3GiB)。(此期间`change_ldt` 函数保留了 `text_size`参数, 但是实际上并未在函数内使用这个参数)

`change_ldt`存在于我们所列出的所有早期版本中。但是在Linux 1.00中`change_ldt`彻底脱离了重设用户态进程LDT的用途, 且与之前提到的从Linux 1.00中开始支持的`sys_modify_ldt`的实现无关。

#### 页机制

用户态进程与(总)页表的关系主要通过`invalidate()`宏和fork中的系统调用中的`copy_page_tables`体现。

页表的使用因是否使用段机制分离用户态进程而划分。

Linux 0.01~0.97使用段机制分隔不同进程, 整个Linux内核与进程公用一份一级页表。此时`invalidate()`宏仅是将0重新写入CR3寄存器来完成页表的刷新(一级位于0x0处)。`copy_page_tables`本质上是对二级页表的复制。

Linux 0.97.6~1.00使用页表来隔离不同的进程, 设计上一个进程私有一份一级页表。此时`invalidate()`宏会通过将CR3读出再写入的方式完成对页表的刷新。`copy_page_tables`会为每个进程的一级页表和二级页表重新分配空间, 完成页表的深拷贝。

在使用页表来隔离不同的进程的版本中, Linux 1.00又是一个特例。它不仅提供了`copy_page_tables`还提供了`clone_page_tables`这个函数。`clone_page_tables`仅将子进程的一级页表指向父进程的一级页表, 并未重新分配空间。

子进程的页表是父进程页表的一份引用。此时内核对于“进程”的描述还停留在process (例: `copy_process`)与task一词。如果从现代角度回看, 我们不难看出其行为与“线程”的关系。尽管此时的clone机制仍是相当简陋的。

#### 段机制与分页机制的相互制约

早期Linux中段机制与分页机制的相互作用制约了所能安装的物理内存大小。Linux 0.97.6及以前, 物理内存的大小与head.S中的页表设置和GDT设置相关。Linux 0.98中物理内存的大小与head.S中的页表设置相关。而到了Linux 0.98.6中, 物理内存大小才与head.S中的设置解耦。

Linux 0.01 ~ Linux 0.97理论上只能安装最大64MiB的内存。

在GDT的内核范围设计上默认为16MiB, 在页表可能冲突的层面上最大可以接受64MiB的范围。由`TASK_SIZE`所影响。Linux早期将`TASK_SIZE`设为64。(4GiB / TASK_SIZE = 64MiB ) 同时仅有一份(总)页表, 前端的内存空间只能由一级页表的前端做恒等映射。这一段虚拟地址并不能跟task1的段虚拟地址冲突。

Linux 0.97.6 ~ 1.00期间, 开发者希望用户态进程能拥有更大的用户态地址空间, 且能支持安装了更大物理内存的机器。但是开发者们关注的仍然是16MiB、32MiB这种级别的物理内存大小的问题。换言之, 早期的内核没有考虑到拥有和GDT限长相等的安装1GiB物理内存的情况。现代Linux能处理好这一点, [^WBL]。



### Physical Memory Layout

#### x86的物理内存

处于兼容性考虑, 物理地址0x000a0000到0x000FFFFF (640KiB~1MiB)处留给BIOS以及一些图形卡[^05cn]P71。

Linux 0.01~ Linux 0.99中, 内核实际被放置在物理地址0x0~640KiB区间中。

Linux 1.00中内核如果不使用压缩启动模式, 内核将被加载到0x00001000处。使用compressed boot则会被放置到0x00101000处。(见head.S)

这是因为Linux 0.01 ~ Linux 1.00的一级页表(Linux 0.01 ~ 0.97为全局一级页表`_pg_dir`, Linux 0.97.6 ~ 1.00为临时一级页表`_swapper_pg_dir`)都是紧挨内核头部放置的, 内核最前端代码将在执行完后被页表数据覆盖。Linux为了保证内核的NULL异常也能被捕获, 所以让第一个页不存在。

> .org加载的位置是LD时推断的位置, 与entry入口的位置无关。
>
> 如果一段代码在链接的时候用了 -Ttext 1000 参数, 然后这段实际被加载到了0x00101000。
> 紧跟在.org 0x1000汇编后的符号会被加载到0x00101000。因为这个符号认为自己处在0x1000 ( 也就是与指定的入口是同一个位置 )。因为加载时整体偏移了0x00100000, 所以实际位于0x00101000。

由于Linux 1.00的代码本身还是以0x1000为起点, 本文中我们仅在这一篇章提及这1MiB的偏移。

Linux 2.6中, 内核被加载到了物理地址的0x00100000处。此时不再需要保留0x1000( 第一个页 )来捕获内核访问NULL指针的异常。此时系统的内核态和用户态用页表权限的方式分离, 保留0x0开始的第一个页的工作留给了用户态的内存布局来实现。

#### RISC-V的物理内存

当下RV会把M-mode的runtime, 即SBI放在物理的内存开头处。Linux RV的启动手册要求内核被放置在与PMD (Page Middle Directory, 页中间目录)的边界上(rv64就是2MiB, rv32是4MiB)。当下SBI runtime的大小往往远小于2MiB。Linux内核代码( head.S )则更直接, 如果是rv64就偏移2MiB, 如果是rv32就偏移4MiB ( PS: 在M-mode下内核不偏移 )。

另外, 使用SBI的话, a1有可能包含设备树在内存中的位置( 之后补充严谨 )。



### Virtual Memory Split

> - 标题名来自https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/tree/arch/x86/Kconfig#n1434  `prompt "Memory split" if EXPERT`

总的来说, 虚拟内存的划分有两种情况。部分体系结构要求的虚拟地址位宽会小于最大寻址的位宽, 它们之间的差距要求以虚拟地址位宽的最高位进行符号扩张, 这会在虚拟地址空间中制造出一个无法合法寻址的空洞区。如果虚拟内存空间中没有空洞, 系统会手动划分。如果虚拟内存空间中有空洞, 则自然的以由空洞分割的两个子空间来划分。

前者的例子是x86-32。它使用`CONFIG_PAGE_OFFSET`来在虚拟内存划分内核态和用户态。[^06]中P145给出了不用划分比例的情况。

x86-64, AArch64, RV64则都符合后者的情况。[^04]P227, [^07]P115分别介绍了x86-64和AArch64的情况。



## Entry Path and Exit Path

### 中断与异常分类



#### x86-32中的中断与异常分类

i386手册规定, There are two sources for external interrupts and two sources for exceptions:
$$
\text{Interrupts}\begin{cases}
\text{Maskable interrupts}\\
\text{Nonmaskable interrupts (NMI)}\\
\end{cases}\\

\\

\text{Exceptions}\begin{cases}
  \text{Processor detected}\begin{cases}
  \text{faults}\\
  \text{traps}\\
  \text{aborts}\\
  \end{cases}\\
  
  \\
  
  \text{Programmed}\begin{cases}
  \text{INTO}\\
  \text{INT 3}\\
  \text{INT n}\\
  \text{BOUND}\\
  \end{cases}\\
\end{cases}\\
$$
其中, INTO, INT 3, INT n, and BOUND这些指令被称为"software interrupts", 但是它们被视为异常来处理。

其中, fault发生时, 会记录引发fault的指令。trap发生时, 会记录指令流中的下一步指令, 例如如果trap发生在jmp, 会记录jmp所转跳的地址, 而不是紧接在jmp后面的指令的地址。



#### RISC-V中的中断与异常分类

RISC-V特权手册规定, 中断和异常都被统称为trap。
$$
\text{Traps}\begin{cases}
  \text{Interrupts}\begin{cases}
  \text{Software interrupt}\\
  \text{Timer interrupt}\\
  \text{External interrupt}\\
  \text{CLIC相关 (Drafting)}\\
  \end{cases}\\

  \\

  \text{Exceptions}\begin{cases}
    \text{Processor detected}\\

    \\

    \text{Programmed}\begin{cases}
    \text{ecall}\\
    \text{ebreak}\\
    \end{cases}\\
  \end{cases}\\
\end{cases}\\
$$
有关CLIC相关的规范正在制定中, 本文并不对齐进行讨论。

其中的Software interrupt对应的是x86或Linux中核间中断的概念(IPI)。

RISC-V仅会记录发生trap时pc的值。但由于RISC-V在非压缩指令集下指令长都为32位, 所以可以用pc+4软件的处理这个问题。压缩指令集下指令长为16位, 本文不讨论这种情况。



### 中断与异常相关的硬件结构

#### x86中的中断与异常的硬件结构

在x86中, 与中断与异常处理关系最紧密的结构是IDT。在x86-32中IDT可以存在三种类型的门描述符、中断门（Interrupt gate）描述符、陷阱门（Trap gate）描述符、任务门（Task gate）描述符。x86-64中已经不在支持任务门描述符, 在IDT中仅能存放中断门描述符、陷阱门描述符。

任务门被设计为用于和TSS配合以完成任务切换。Linux仅用它来处理x86-32中的双重错误。

中断门和陷阱门的差异在于, 当处理器穿过中断门时, 会关闭中断, 而穿过陷阱门时则不会。在初始化这两种门的时候需要设置其中的DPL位。只有利用INT n, INT 3 或 INTO指令产生指令时才会检查当前权级与DPL的关系。这是为了防止用户通过这些指令来模拟非法的中断或异常。如果我们希望用户能发出一个编程异常, 需要将对应中断门或陷阱门描述符的DPL设置为3(也就是x86中用户态全级)。否则, DPL应该被设置为0。

x86-64中, 中断门和陷阱门引入了IST位, 它可以无条件的进行栈的切换[^WBL]。

x86中中断指向一个中断向量表, 当发生对应中断时, 会转跳到对应地址去执行。

x86还有一种门叫调用门(call gate), 它被设计为允许低特权级安全的请求高特权级服务。部分Linux版本设置了lcall7和lcall27调用门, 但Linux仅用它执行对BSD和Solaris的二进制仿真。本文并不讨论调用门。



#### x86 Linux中有关门的术语

| Linux中有关门的术语  | 实际使用的门   | DPL  | IST  |
| -------------------- | -------------- | ---- | ---- |
| trap_gate            | trap gate      | 0    | N    |
| system_gate          | trap gate      | 3    | N    |
| system_trap_gate     | trap gate      | 3    | N    |
| intr_gate            | Interrupt gate | 0    | N    |
| intr_gate_ist        | Interrupt gate | 0    | Y    |
| system_intr_gate     | Interrupt gate | 3    | N    |
| system_intr_gate_ist | Interrupt gate | 3    | Y    |
| task_gate            | Task gate      | 0    | N    |

本文中在使用Linux术语体系的时候, 本文会用下划线连接。否则, 则指的是Intel手册中的术语。

总的来说, x86 Linux逐步放弃使用trap gate而改用Interrupt gate。

从0.01到0.96c中, 都还存在有中中断源使用trap gate的情况。从0.97开始, 仅异常源会使用trap gate。在2008年10月13日v2.6.28-rc1的一份名为"i386: prepare to convert exceptions to interrupts"的`61aef7d`提交正式拉开了从trap gate几乎全面转向Interrupt gate的序幕。在2.6版本的最后一个小版本中, 除双重错误外, 仅剩下x86-32的系统调用还在使用trap gate。而到本文所处的6.8时代, 除了双重错误外, 一切中断和异常都使用Interrupt gate。

一开始Linux中并没有使用Task gate, v0.01- v1.00都是如此。但是至少从2.6开始, Linux一直在x86-32中的双重错误时使用Task gate。x86-64则由于取消了Task gate, 仅能使用带IST的Interrupt gate。



#### RISC-V中的中断与异常的硬件结构

RISC-V的中断控制器仍在发展当中, 现阶段主要的4中控制器时CLINT、PLIC、CLIC、APLIC。本文仅讨论使用CLINT和PLIC的情况。CLINT在中断控制上非常初级, 面对需要抢断、嵌套等需求时显得力不从心。CLIC的出现试图解决这些问题。尽管有的实体芯片已经支持CLIC, 但是由于它和APLIC的规范都还在crafting状态, 本文不再进一步讨论。

但无论是CLINT还是CLIC, RISC-V对异常的处理是没有变的。RISC-V中可以通过配置`xtvec`CSR寄存器来设置trap的入口和是否使用中断向量。但是暂时的实现中, 很少在RISC-V中使用中断向量。首先, 因为无论是否配置使用中断向量, 异常发生时pc都会指向`xtvec`所指向的地址, 然后再分类讨论。其次, 不使用CLIC的情况下, 每权级仅有3种中断情况, 所有的外部中断还是需要从PLIC种软件的获取中断号然后分类讨论。CLIC或许会改变这一格局。

`xcause`是RISC-V中用于存储trap类型的CSR寄存器。`xcause`的最高位为1时, 代表其为中断, 否则则为异常。Linux中使用了一个技巧, 就是将`xcause`中的值看作有符号数然后和0比较。由于最高位为1的有符号数小于0, 用一条汇编指令即可判断是中断还是异常。



### 中断与异常时栈的切换

Linux用栈隔离不同的运行环境。最经典的例子是用户态和内核态的代码流拥有不同的栈。现代Linux拥有更多的栈环境。例如, 在某些配置下处理中断时会专门为中断处理程序分配一份临时的栈; 再例如在处理部分异常时, 会使用`set_system_intr_gate_ist`来设置单独的栈。本文仅在不考虑内核进程的情况下讨论每一个进程结构体拥有的用户态和内核态两种栈。

总的来说, 内核中两种栈的切换遵循以下原则, 当中断或异常导致权级切换时, 会切换为对应的栈。如果发生中断或异常导致权级不变, 则继续使用原来的栈。

x86结构中, 在TSS中设置好不同权级的栈后, 当发生中断或异常时会硬件的进行保存与切换。在RISC-V中, 栈的切换则全是由软件完成的。

RISC-V将用户栈和内核栈的信息存放在`thread_info`结构体中, 在和RISC-V一样使用`CONFIG_THREAD_INFO_IN_TASK`配置的体系架构中, 这同时意味着指向进程描述符的指针同时指向`thread_info`结构体。RISC-V用tp指针指向当前所运行的进程描述符或者说`thread_info`结构体, 正如我们之前谈及的, tp寄存器在函数调用前后保持不变。当中断或异常发生时, 通过操作tp寄存器和`xscratch`CSR寄存器, 能软件的切换栈。



### Linux中的Entry Path and Exit Path的演变

总的来说, Linux在尝试归不同中断与异常的Entry Path和Exit Path, 但在尝试的过程中同时也追求对不同中断和异常处理的差异性。

Linux中重要的Entry Path有3个: 系统调用、中断和异常的Entry Path。

Linux中重要的Exit Path有4个: 系统调用、从fork返回(的新进程)、中断和异常的Exit Path。

Linux 0.01~ Linux1.00中不存在从fork返回的Exit Path, 这一点我们在下一节说明[^WBL]。
$$
\begin{cases}
  \text{sys\_call.S and asm.S}
  \rightarrow
    \begin{cases}
    \text{中断与系统调用在sys\_call.S中}\\
    \text{异常在asm.S中}\\
    \end{cases}
  \rightarrow
    \begin{cases}
			\text{入口栈中的顺序不一致}
        \begin{cases}
        \text{Linux 0.01}\\
        \text{Linux 0.10}\\
        \text{Linux 0.11}\\
        \text{Linux 0.12}\\
        \end{cases}
      \\
      \text{入口栈中的顺序一致}
      	\begin{cases}
        \text{仅系统调用使用ret\_from\_sys\_call}\rightarrow\text{Linux 0.95}\\
        \text{系统调用、异常、一部分中断共用返回}\rightarrow\text{Linux 0.96c}\\
        \end{cases}
    \end{cases}

  \\

  \text{sys\_call.S and irq.h}
    \rightarrow
    \begin{cases}
    \text{异常与系统调用在sys\_call.S中}\\
    \text{中断在irq.h中}\\
    \end{cases}
  \rightarrow
    \text{系统调用、异常、一般中断共用返回; 快速中断除外}\begin{cases}
      \text{Linux 0.97}\\
      \text{Linux 0.97.6}\\
      \text{Linux 0.98}\\
      \text{Linux 0.98.6}\\
      \text{Linux 0.99}\\
      \text{Linux 1.00}\\
    \end{cases}
\end{cases}\\
$$
下图为2.6.12中内核路径, Linux 0.97~1.00相较而言, 少了ret_from_fork的路径, 多了FAST_INTR的路径。注意, 由于使用中断向量表, 本质上异常和中断都有多个入口点, 但由于其中代码实际上一样, 我们仅展示一条线路。

![2_6_12内核路径.drawio](./asset/2_6_12内核路径.drawio.png)

下图展示的是6.8 x86中的内核路径, 其中`\cfunc`为具体函数名, 正如我们之前所说, 此时的内核几乎仅使用中断门。同样实际存在多个入口, 但都由同一个宏展开。

![6_8_x86内核路径.drawio](./asset/6_8_x86内核路径.drawio.png)

下图展示的是6.8 RISC-V中的内核路径, 此版本的RISC-V内核没有使用中断向量, 使用事实上还是就只有一条Entry Path。

![6.8_RV_路径.drawio](./asset/6.8_RV_路径.drawio.png)



### fork后子新进程的Entry Path and Exit Path

fork后新的子进程在第一次背调度器选中并调度时是特殊的。总的来说, fork后新的子进程Entry Path and Exit Path模拟了其父进程的过程。我们以进入Entry Path前、do fork、退出Exit Path后三个时间点。

早期Linux中, 从进入路径来看, fork的子进程没有进入路径。 它直接复制了其父亲进程进入Entry Path前的状态, 而忽略其父亲进程进入Entry Path到do fork之间的状态。这个时期的fork的子进程也无需退出路径, 它会直接返回到退出Exit Path后这个时间点。 从进程视角看, 这仅是发出fork系统调用而进入Entry Path的后一时刻。

现代Linux的fork后的子进程不能这样做。所有可能被调度的进程在它背调度切换到指令流的那一刻, 都需要检查是否要移除将控制器交给它的进程是否处于需要被清理并移除的状态 [^WBL]。于是, 现代Linux为fork后的子进程添加了一个特殊的Exit Path, 即`ret_from_fork`。但fork后的子进程依旧不存在entry path, 所以在do fork时, 改为复制刚刚进入Entry Path后的父进程状态。





### pt_regs在Entry Path and Exit Path中的作用

`struct pt_regs`是用于实现ptrace系统调用的数据结构。从Linux 0.01 ~ Linux 0.12, ptrace系统调用属于未实现的状态。ptrace系统调用从0.95开始正式实装, 这也正是Linux在x86体系中统一了系统调用、异常与中断发生时栈内的顺序。

现代Linux为了实现内核核心代码与体系结构的解耦, `struct pt_regs`是特定于体系结构的[^06]P52。在系统系统调用、异常与中断发生的C语言代码中, 实际上传递的就是`struct pt_regs`。新架构的移植很大程度上依赖于这些类似`struct pt_regs`结构体和函数。它们使现代的Linux系统框架化, 且易于移植。







### 由信号处理产生的额外路径

信号的处理发生在内核将要返回用户态的时间点。对于一个信号的处理, 内核可能可以在内核中就完成处理。但在另一些情况下, 内核需要返回用户态执行用户态的信号处理程序, 再由用户重新再入内核态来完成之前被打断的过程。

C库会在当需要用户态代码参与内核的重入时起作用。至于用户态C库具体如何实现, 本文不做讨论, 仅是说明会有这样一种情况打断原有的路径。



## Context Switch

### current在上下文切换中的作用

Linux内核用名为`current`的变量/宏/函数来直接或间接获取当前CPU上的当前进程描述符, 它在两个进程上下文切换中起到了关键作用。

在单CPU系统中这可以通过在内存中设置一个全局`current`变量来实现。多核系统中, 一个简单的实现方式则是在`per_CPU`( 每CPU变量, 具体实现方式不多赘述 )中存放一个`current`变量。`per_CPU`实现依赖于对当前代码所运行的CPU的标识。对于可以获取当前CPU标识的体系结构来说, 这并不困难。但对于不对系统暴露当前CPU的标识的体系结构来说, 仍然需要一个变量来存储当前CPU的标识, 且这个标识显然不可能存放在`per_CPU`中。

于是Linux使用了最最常驻在当前CPU运行流中的结构进程描述符来直接或间接地存储这一结构。所以参与上下文切换的两个进程必须保证切换过程中一直能正确的直接或间接地保存当前CPU的标识。这是现代Linux上下文切换流程与早期Linux上下文的切换有着巨大不同的原因。

它导致了参与上下文切换核心代码的参数从一个变为了两个。它导致了进程的彻底销毁必须由接管当前CPU的进程进行, 这又进一步影响了另外两个步骤。这导致了上下文切换核心代码必须将切换发生前的进程作为返回值返回。同时导致了fork系统调用( 包括fork、vfork、clone )产生的子进程不能直接返回到原上下文, 而必须插入`ret_from_fork`的执行。



### Linux中switch_to的演变过程

`switch_to`宏一直被包含于`schedule`函数中。现代Linux的`__schedule`函数变得及其复杂, 但是其进行上下文切换的思想没有改变。即, 挑出下一个将要运行的进程`next`, 将它和当前进程`current`进行上下文切换。

早期Linux中, `switch_to`宏本质上是一段內联汇编。此时的 `switch_to`宏仅有一个`next`参数, 它能独自完成两个进程间全部的上下文切换工作。现代Linux由`context_switch`来调用一系列体系结构无关函数/宏来完成上下文的切换。`context_switch`中使用了名为`switch_to`宏来包装体系结构相关的`__switch_to`函数。有的体系结构`__switch_to`全部由汇编编写, 有的体系结构`__switch_to`函数除汇编层面的函数外还包含了C函数和。

#### 核心参数由1个变为2个

正如本文前文提及的, `current`作为全局变量存在于内存之中, 使用早期Linux的`switch_to`仅需要`next`一个参数。而`context_switch`需要两个参数`prev`和`next`两个参数, `prev`用于`switch_to`事实上完成进程切换后的清理工作。为了实现这一目标, `switch_to`需要3个参数 `prev`, `next`和`last`, 并接受`context_switch`以 `prev`, `next`和`prev`传入参数。

例如, 我们研究进程A。在某一时刻A会切换到X; 在另一时刻, 会从Y切换到A。在此我们假定进程X, Y是不是进程A的任意进程。如果X, Y是进程A并不影响最终的结论, 因为没有实际发生切换, 切换过程会更简单。我们将`context_switch`中`switch_to`之前的部分称为$A_1$, $X_1$和$Y_1$, 将`switch_to`之后的部分称为$A_2$, $X_2$和$Y_2$。对于$A_1$而言, `prev`是自身, 而`next`是$X_2$。对于$Y_1$而言, `prev`是自身, 而`next`是$A_2$。所以, 对于$A_2$而言, `prev`是$Y_1$, 而`next`是自身。穿越`switch_to`前后 , `prev`的内容发生了变化。所以我们利用`switch_to`将`prev`指向正确的进程。( 注: `switch_to`是宏, 所以`prev`本身的值被改变。假设 `switch_to`是函数, 则只会改变深拷贝后`prev` 的拷贝值)

`__switch_to`和`cpu_switch_to`都通过接收`prev`和`next`并返回`last`参数的方式在 `switch_to`宏中起作用。所以从最终汇编代码视角来看, 仅因为不能使用常驻全局变量`current`而导致参数由1个变为2个。

#### switch_to与thread_struct

Linux 0.01~1.00中`switch_to`宏仅用了不到10行内联汇编。能如此简洁是因为使用了x86-32的TSS结构。x86-32的TSS结构几乎存储了进程切换所需的一切底层信息, 且能用几行汇编简单地从一个进程的TSS切换到另一进程的TSS。

体系结构都会实现一个硬件结构用于进程的切换是昂贵的, 且x86-32的TSS结构体在使用时也仅使用了一部分结构, 存在大量的浪费。大多数体系架构不会设计一个专用的硬件结构用于进程的切换, x86-64的TSS结构体也被更改为仅能存储IO位图和各种栈。出于对体系结构的解耦, 现代Linux引入了`thread_struct`结构体( 它在进程描述符中声明为`struct thread_struct thread` )。在`switch_to`切换时, 将`prev`中必要的上下文保存在`thread_struct`结构体中, 并从`next`的`thread_struct`结构体中恢复上下文。



### 进程删除时机与上下文切换的关系

现代Linux中[^05cn]将进程的撤销(Destroying )过程分为了进程终止( Termination )和进程删除( Removal )两个阶段。所有进程的终止都是由`do_fork`函数来处理的, 这个函数并不会删除进程描述符的全部结构。正如我们之前所说, 上下文的切换仰赖进程描述符中的一些结构, 所以进程描述符的彻底删除由调度程序在`switch_to`发生后执行。

早期Linux则无需分开处理, `current`一直在单一的内存全局变量中。因而没有必要标识当前CPU, 自然也不用担心进程描述符所有的内容被清空。



### 浮点寄存器的上下文切换

不是所有体系结构都支持浮点操作。同时部分架构也不仅只有浮点寄存器需要保存与恢复, 例如x86-32中除了FPU外还有MMX和SSE/SSE2单元。我们仅用浮点寄存器的上下文保存与恢复来说明一个通用的思路。

同时, Linux也能支持模拟这些非常规指令的软件方式。Linux 0.11中便加入了数学协处理器软件模拟的代码, 在此本文不讨论软件模拟的情况。

以fpu为例, 总的来说, fpu的上下文处理的思路基本一致。仅在必要的时候保存, 仅在必要的时候恢复。不同的体系结构与不同版本的Linux都有着不同的处理方式。

在x86-32中, TSS中留有给i387协处理器的空间。随着改用thread_struct替代TSS结构存储进程上下文, 其上下文空间很自然的移动到了thread.i387字段中( thread是进程描述符中的thread_struct结构体的变量名 )。

各个体系结构中, 有专门的硬件结构用于指示这些类似fpu的处理器的状态, 如果在上下文切换时发现前一进程使用了fpu, 便会在切换过程中保存之前进程fpu的上下文。

各体系结构对fpu上下文的恢复的处理则有不同的策略。下面我们分别介绍x86-32、ARM64和RISC-V恢复fpu上下文的策略。

x86-32的策略从早期Linux版本延续至今, 它的策略是将fpu上下文的恢复推迟到下一次进程使用了fpu后。它在切换时设置cr0寄存器的TS位, 这会导致当有进程试图使用fpu时发生`device_not_available`异常。在异常处理程序中, 才会真正恢复当前进程的fpu上下文[^05cn]P117。

ARM64的策略是微微延后fpu的恢复到进程准备返回用户模式的时候。它会在需要恢复时设置thread_info结构体中浮点状态相关的标志位TIF_FOREIGN_FPSTATE, 然后在进程准备返回用户模式时调用对应函数恢复fpu的上下文[^07]P71。

RISC-V的恢复策略最为直接, 如果发现被切换的下一个进程有fpu不关闭禁用fpu, 就为下一进程恢复fpu的上下文。



### fork后新的子进程的上下文

我们在上一章中讨论过这个问题的一部分[^WBL]。

在依旧使用TSS作为进程切换主要手段的Linux版本和现代Linux有着明显的不同。 以Linux 0.12为例, 在do_fork的核心copy_process中, 会刚进入entry path所保存的进入entry path之前的状态存入TSS中 ( 实际上还有修改返回值的过程 )。正如我们之前所说, 在新fork生成的进程第一次被调度时, 会回到用户态中进入entry path之前状态的下一条代码。

但由于现代Linux中不使用TSS, 同时添加了`ret_from_fork`的退出路径, 所以要在新fork生成时模拟它进入了系统调用的状态。调用copy_thread利用刚进入内核时所保存的pt_regs来初始化子进程的内核栈( 并修改其中作为返回值的部分 )[^05cn]P125, 因为exit path需要利用内核栈上的这些值来退出。

同时, 还需要模拟被切换上下文的状态, 通过新fork出的子进程的thread_struct结构体中的相应寄存器, 将`ret_from_fork`设为子进程的函数返回地址。这样, 当switch_to第一次切换到这个进程时, 就会转跳到`ret_from_fork`执行退出路径。



## Kernel and Process Initialization

### 进程描述符和内核栈关系的演变

Linux中, 进程描述符和内核栈始终保持着紧密的关系。

第一阶段, 进程描述符和内核栈在连续的物理空间的两端。例如, Linux 0.01~Linux0.98中, 进程描述符和内核栈处于同一个4KiB页的两端。

第二阶段, 进程描述符通过自身元素指向内核栈。例如Linux 0.98.6~1.00中, 进程描述符位于一个4KiB的页中, 进程描述符中的kernel_stack_page==成员==指向一个单独的4KiB的内核栈。==(记得全文统一格式!!!!!!!!!)==

第三阶段, 将内核栈和thread_info结构体放置在连续的物理空间的两端[^05cn] P90。此时current会指向的是thread_info, thread_info中的task指向进程描述符, 进程描述符中的thread_info又指向thread_info而间接的指向了栈。

如2.6的x86-32中, 内核栈和thread_info结构体放置在8KiB的两页中, thread_union共用体定义如下。

```c
union thread_union {
	struct thread_info thread_info;
	unsigned long stack[THREAD_SIZE/sizeof(long)];
};
```

第四阶段, 引入了新的thread_info布局。 可以通过设置`CONFIG_THREAD_INFO_IN_TASK`将结构体 thread_info设为是进程描述符的第一个成员而不将其放在内核栈中。但无论哪种布局, 进程描述符中的stack元素指向内核栈(指向内核栈所处内存空间的地址最低端, 例如不定义`CONFIG_THREAD_INFO_IN_TASK` 时本质上还是指向了thread_info ) [^07] P26。

例如, 4.12的ARM64 Linux使用将thread_info设为是进程描述符的第一个成员的布局, thread_union共用体定义如下。

```c
<include/linux/sched.h>
union thread_union {
#ifndef CONFIG_THREAD_INFO_IN_TASK
	struct thread_info thread_info; 
#endif
	unsigned long stack[THREAD_SIZE/sizeof(long)]; 
};
```

第五阶段, 将进程描述符和内核栈放置在连续的物理空间的两端成为一个可由`CONFIG_ARCH_TASK_STRUCT_ON_STACK`配置选项。例如, 可以从4.18.0版本的thread_union共用体定义看出: 

```c
union thread_union {
#ifndef CONFIG_ARCH_TASK_STRUCT_ON_STACK
	struct task_struct task;
#endif
#ifndef CONFIG_THREAD_INFO_IN_TASK
	struct thread_info thread_info;
#endif
	unsigned long stack[THREAD_SIZE/sizeof(long)];
};
```

第六阶段,进程描述符和内核栈放置在连续的物理空间的两端重新变为不可配置的布局方式。例如, 6.8版本的thread_union共用体定义: 

```c
union thread_union {
	struct task_struct task;
#ifndef CONFIG_THREAD_INFO_IN_TASK
	struct thread_info thread_info;
#endif
	unsigned long stack[THREAD_SIZE/sizeof(long)];
};
```

需要说明的是, 无论哪种布局方式, 进程描述符中的`kernel_stack_page`, `thread_info` 或 `stack`成员都指向的是栈所在空间的低端地址。在初始化内核栈时, 本质上是使用, 如6.8中RISC-V的初始化进程0的进程描述符和内核栈时:

```c
la tp, init_task
la sp, init_thread_union + THREAD_SIZE
```

其中, 进程0的thread_union, 即init_thread_union, 隐式地通过链接脚本(linux/include/asm-generic/vmlinux.lds.h)和进程0的进程描述符init_task指向同一地址:

```c
#define INIT_TASK_DATA(align)						\
	. = ALIGN(align);						\
	__start_init_task = .;						\
	init_thread_union = .;						\
	init_stack = .;							\
	KEEP(*(.data..init_task))					\
	KEEP(*(.data..init_thread_info))				\
	. = __start_init_task + THREAD_SIZE;				\
	__end_init_task = .;
```

同样也能通过以下这段代码看出: ==别写进论文!!!==

```c
#ifdef CONFIG_THREAD_INFO_IN_TASK
# define task_thread_info(task)	(&(task)->thread_info)
#elif !defined(__HAVE_THREAD_FUNCTIONS)
# define task_thread_info(task)	((struct thread_info *)(task)->stack)
#endif
```



### 内核与进程0的初始化

#### 内核与进程0的关系

所有进程的祖先叫进程0( process 0 ), idle进程( idle process )或因历史原因称为swapper进程( swapper process )。它是Linux在初始化阶段从无到有创建的进程。无论是早期Linux还是现代Linux中, 这个祖先进程都是特别的。它使用的很多数据结构都是静态分配的, 而所有其他进程/线程的数据结构都是动态分配的。

内核的代码从head.S开始, 一些体系结构在head.S前还会有一些额外的工作需要做, 本文不讨论这种情况。

如果用现代Linux的术语来分析内核与0号进程的关系, 我们会称0号进程是一个内核线程( kernel thread )。它会分支出其他的内核线程, 其他的内核线程进一步分支出用户态进程。因此我们不难讲整个操作系统运行流程分为3个部分: 内核服务, 内核任务和用户态任务。内核线程在内核服务, 内核任务两者中切换。用户进程在内核服务和用户态任务中切换。内核在进入head.S的执行流后不久便会将进程0的相关数据结构安置妥当, 在此之后的内核初始化指令流本质上就是进程0的内核服务的代码。

早期Linux中情况会变得复杂。第一、早期内核不存在内核线程的概念, 进程0是用户态进程。第二、内核流执行一段时间后才会装载进程0的相应数据结构。三、从线性地址的角度看, 内核执行流和进程0的用户态执行流是紧密接续的。从内核执行流到进程0用户态执行流的切换是通过切换x86-32段选择子来完成的。这导致了在除了用户态进程的内核服务和用户态任务执行流之外, 还有一段无法被忽视的, 仅属于内核程序初始化的执行流。

严谨的说, 现代内核也有这么一段仅属于内核初始化程序的执行流。但是这段执行流通常很短, 且用不依赖于栈的汇编语言编写。



#### 内核与进程0的栈

理论上不同的C语言运行环境依赖于不同的栈。正如我们之前在“中断与异常时栈的切换”一节讨论的一样, 我们仅讨论导论因权级不同而导致C语言运行环境不同的内核栈和用户栈。

从进程角度来看, 每个进程的进程描述符中都有自己的内核栈和用户栈字段。每个进程的内核栈和其进程描述符紧密联系, 因此, 每个进程的内核栈也和它的进程描述符一样具有唯一性这一点我们稍后说明[^WBL]。用户栈则仅能保证进程描述符里有它的字段。不同进程描述符的用户栈可能指向不同的地址空间, 也有可能因为clone的原因指向同一地址空间的引用, 还有可能是内核线程而不使用内核栈。

不考虑进入head.S之前可能使用的代码和栈。早期Linux有3种栈, 内核初始化使用的栈、用户态栈和内核态栈。而现代内核只有用户态栈和内核态栈。

Linux 0.01~1.00有3种栈, 并将内核初始化使用的栈作为进程0的用户栈。这样做的原因是两者的执行流的线性地址上看是连续的, 只用切换栈的段选择子即可继续运行。

无论是早期Linux还是现代Linux, 进程0的内核栈都是特殊的。正如前文所说, 它们都是系统通过`TASK_INIT`等一些列宏设置出来的。

余下其他所有进程的栈则依赖内核的其他机制来处理, 初始化过程不用再为其操心了。

![进程栈的分配过程.drawio](./asset/进程栈的分配过程.drawio.png)

#### ~~6.8中RISC-V与ARM64在初始化内核栈上的对比 (不写入论文)~~

在[v6.5-rc1](https://github.com/torvalds/linux/releases/tag/v6.5-rc1)中的[commit c7cdd96](https://github.com/torvalds/linux/commit/c7cdd96eca2810f5b69c37eb439ec63d59fa1b83)中, Linux RV以非常随意的理由在内核栈上留出了一个PT_SIZE_ON_STACK的空间。

ARM64在初始化时也留有一个PT_REGS_SIZE大小, 以6.8中的代码为例:

```c
SYM_CODE_END(preserve_boot_args)

	/*
	 * Initialize CPU registers with task-specific and cpu-specific context.
	 *
	 * Create a final frame record at task_pt_regs(current)->stackframe, so
	 * that the unwinder can identify the final frame record of any task by
	 * its location in the task stack. We reserve the entire pt_regs space
	 * for consistency with user tasks and kthreads.
	 */
	.macro	init_cpu_task tsk, tmp1, tmp2
	msr	sp_el0, \tsk

	ldr	\tmp1, [\tsk, #TSK_STACK]
	add	sp, \tmp1, #THREAD_SIZE
	sub	sp, sp, #PT_REGS_SIZE

	stp	xzr, xzr, [sp, #S_STACKFRAME]
	add	x29, sp, #S_STACKFRAME

	scs_load_current

	adr_l	\tmp1, __per_cpu_offset
	ldr	w\tmp2, [\tsk, #TSK_TI_CPU]
	ldr	\tmp1, [\tmp1, \tmp2, lsl #3]
	set_this_cpu_offset \tmp1
	.endm
```

但其明确说明了这个PT_REGS_SIZE是为了winder追踪内核所用的。

相较之下, RISC-V的理由显得过于随意。





### 分页机制的初始化

分页机制的初始化与系统结构和内存管理策略息息相关。初始化阶段涉及到两个重要映射机制, 恒等映射(identity mapping)和直接映射(direct mapping)。请注意直接映射和固定映射(fix-map)的区别, 它们都是偏移固定常量的线性映射。直接映射线性偏移量通常在编译内核时就已经确定, 且常常与体系结构相关。直接映射的虚拟地址和物理地址在内核生命周期中永远存在且保持一致, 通过`__pa(vaddr)`和`__va(paddr)`相互转化[^06] P142。固定映射的线性偏移量不是预设的, 且固定映射本身可以被动态的创建和销毁。

恒等映射往往用于内核初始化阶段的过渡, 当这种映射不再必要时, 内核会清除对应的页表项[^05cn] P76。恒等映射也不是必须建立的,  6.8版本的RISC-V Linux就没有建立恒等映射, 而是用异常转跳的方式完成的过渡。

下面具体说明一些例子:

Linux 0.01 ~ Linux 1.00中:

0.01~0.97版本中, 会做机器内存大小的恒等映射。因为此时内核的起点在线性地址和逻辑地址的0x00000000处。如果机器安装了更大的物理内存, 则需要手动更改内核源码设置更多的页表来初始化。

0.97.6和0.98中, 仅做内核前4MiB的恒等映射, 与机器内存大小的直接映射。这4MiB内存已经包含了内核所有的代码部分。直接映射部分则同理, 如果机器安装了更大的物理内存要修改内核源代码以适应。

0.98.6~1.00中, 仅做内核前4MiB的恒等映射和直接映射, 随后内核会在`paging_init`函数中映射所有机器物理内存。

Linux 2.6版本的x86-32中, 会做内核前8MiB的恒等映射和直接映射。此时要求内核使用的段、临时页表和用于存放动态数据结构的128KiB都能被容纳于这8MiB空间里[^05cn] P74。

Linux ARM (不是ARM64)中(到当前6.8版本都遵循), 会恒等映射head.S中的一小部分代码和直接映射内核部分。恒等映射部分从head.S中`__enable_mmu`到`__turn_mmu_on`, 其中的代码负责开启内存管理单元[^07] P10。

Linux RISC-V中(到当前6.8版本都遵循)没有恒等映射。其他体系结构中恒等映射的部分被`trampoline_pg_dir`页表和异常处理的转跳机制的配合所替代, `trampoline_pg_dir`中只有直接映射。这套机制同时被用于多核启动或CPU热插拔的启动。此外, 除了体系结构无关的`swapper_pg_dir`外, RISC-V还创建了一个静态分配的`early_pg_dir`供(主)启动核使用。`early_pg_dir`会在`setup_bootmem()`探明内存后被`setup_vm_final`函数融入`swapper_pg_dir`中[^18]。



### fork中新进程的初始化

Linux0.01~1.00中, fork系统调用全部由sys_fork进行。Linux 1.00中加入了sys_clone, 它本质上是`#define sys_clone sys_fork`。

6.8版本中, 所有与fork行文相关的系统调用, 最终都由父进程的kernel_clone函数执行, 区别仅是传递给kernel_clone的参数不同。进程会经过一系列数据结构的初始化, 并最终在kernel_clone所调用的wake_up_new_task唤醒开始正式开始执行。

#### 进程描述符的初始化

(2.6之类的之后再考古吧, 也可能不考了)

Linux0.01~1.00中, 包含进程描述符的初始化的整个fork过程都在sys_fork函数中进行。6.8版本Linux的进程描述符的创建由kernel_clone中的copy_process完成。

#### 内核栈的创建

前文描述了内核栈和进程描述符的关系。

Linux 0.01~Linux0.98中, 进程描述符和内核栈在连续的物理空间的两端, 在分配进程描述符空间时, 栈的空间也就同时被分配了。

Linux 0.98.6~1.00中, sys_fork为kernel_stack_page成员分配内核栈空间。

6.8版本中, 不管使用何种布局, 都由copy_process调用的dup_task_struct中的alloc_thread_stack_node进行分配。

#### thread_info的复制

早期Linux没有thread_info结构体。

6.8中由copy_process调用的dup_task_struct中的thread_info的复制由setup_thread_stack完成。此时仅是复制并设置了thread_info相关的结构, 与栈内容的复制无关。

#### 内核栈内容的复制

正如前文所述, 早期Linux不存在内核栈的复制, 因为早期Linux的fork的退出路径返回点是进入fork调用前。而早期Linux规定, 当用户态切换为内核态时, 内核栈是空的, 所以并没有复制过程。(注意, 早期Linux会对进程描述符结构体进行复制, 在这个过程中, 不会复制内核栈, 这是C语言共用体的语法决定的)

现代Linux的新进程内核栈的设置在copy_process中的copy_thread中完成, 其复制了父进程内核栈上最先入栈的pt_regs结构体到自己的内核栈中。

#### 内核栈的初始化

早期Linux复制父亲进程TSS的内容到子进程的TSS中, 通过设置子进程上下文的方式设置了用户栈的位置。

现代Linux的用户栈的引用的复制也在copy_process中完成。如果是用户态进程, 现代Linux中父亲进程的用户栈显然包含在内核栈上最先入栈的pt_regs结构体中, 在copy_thread中已经复制。内核线程不使用用户态内存空间, 自然不使用用户栈, 在fork时也不会为子进程分配用户栈。

以上操作都是复制栈的引用, 栈空间的真正分配则要留到返回用户态后, COW写时复制机制为第一个像栈空间写的进程分配新的栈内存空间。

#### 子进程执行流的初始化

正如前文所述, 早期Linux和现代Linux在fork后新进程的Exit Path上有很大的区别。在此我们仅讨论用户进程的情况。早期Linux用TSS直接设置, 而现代Linux则是在copy_thread中将pc设置为ret_from_fork, 栈指针设置为其自己栈中所复制的pt_regs结构体的位置上。



### execve中新程序的初始化

本文此阶段仅讨论体系结构相关的问题。因此对execve的讨论仅限于对其运行最基本环境的设置。将中断使能设置好, 将pc寄存器设置好, 将栈设置好。如有FPU之类的, 根据体系结构的不同完成对FPU等设备的初始化。



### fork与execve在早期Linux中对段机制的初始化

在Linux0.01~1.00期间, 进程0, 即init_task在段机制上与其他进程有所区别。fork与execve参与了各子进程段机制的初始化。

```
# Linux 0.01 ~ Linux 0.99 中研究用户态进程起点与范围的主要相关函数/宏名

1、只列出了主要相关函数/宏的函数/宏名
2、函数/宏的参数可能在各个版本中有变化

sys_fork
	└── copy_process
				└── copy_mem
							└── set_base
							
sys_execve
	└── do_execve
				└── change_ldt
							├── set_base
							└── set_limit
```

Linux0.01 ~ 0.99的init_task的段限长都是640KiB。

首先要说明的是`set_base`和`set_limit`都只更改对应表项的对应部分。例如, 在`copy_mem`中调用`set_base`时, 对应表项的原limit属性并没有改变。

- Linux 0.01、0.10、0.11

  在`copy_mem`中调用`set_base`宏设置fork出的子进程的base为`nr * 0x4000000` ( 64MiB )。

  在`change_ldt`中依据传入的`text_size`参数用`set_limit`宏设置代码段限长, 并将数据段长度设为`0x4000000` ( 64MiB )。虽然调用了`set_base`但是实际没有改变当前进程的base参数。

- Linux 0.12、0.95、0.96c、0.97

  引入了`TASK_SIZE`宏, 此时的`TASK_SIZE`宏为`0x4000000` ( 64MiB )。

  在`copy_mem`中调用`set_base`设置fork出的子进程的base为`nr * TASK_SIZE`。( 仅引入了宏, 本质与Linux 0.01、0.10、0.11没有区别 )

  在`change_ldt`函数保留了`text_size`参数, 但未在函数体内使用。使用`set_limit`宏同时设置代码段限长和数据段限长为`TASK_SIZE`。虽然调用了`set_base`但是实际没有改变当前进程的base参数。

- Linux 0.97.6、0.98、0.98.6、0.99

  此时的`TASK_SIZE`宏为`0xC0000000` ( 3GiB )。

  在`copy_mem`中调用`set_base`设置fork出的子进程的base为0。

  在`change_ldt`函数保留了`text_size`参数, 但未在函数体内使用。使用`set_limit`宏同时设置代码段限长和数据段限长为`TASK_SIZE`。虽然调用了`set_base`但是实际没有改变当前进程的base参数。

- Linux 1.00

  Linux 1.00使用了GDT中的表项作为自己的段描述符, 其数据段和代码段限长应该都是3GiB。尽管在Linux 1.00的`INIT_TASK`前的注释中写道`Base=0, limit=0x1fffff (=2MB)`, 但是从具体使用的段选择子来看, init_task数据段和代码段限长应为3GiB。



### 进程目标处理器的初始化

(之后再慢慢补充吧)

早期Linux是单核的, 现代Linux中不同架构中对进程的目标处理器初始化在具体细节上也各有千秋。所以我们仅讨论现代的通用思想和RISC-V的一些细节。Linux内核在调度等问题上使用是cpuid, 它是与处理器实际的编号无关的。Linux会维护一份cpuid和处理器实际编号的相互映射关系, 在RISC-V中, 这个映射就是cpuid和hartid的相互映射

#### 进程0目标处理器的初始化

在系统启动时, 第一个开始启动的处理器会被内核映射为cpuid为0。内核将在这个处理器上运行进程0。

参考[^18], 在现在的RISC-V系统上, 有两种进入内核的方式。如果使用`RISCV_BOOT_SPINWAIT`方式启动, 意味着硬件将一次性释放所有hart开始运行内核代码。此时通过在内核初始化流程中通过原子指令来选中一个核作为启动核, 其他和在某个symbol处不停spin等待被启动核放行。`Ordered booting`方式下, 硬件仅会释放一个核用于执行初始化流程。其他核通过SBI的HSM扩展进行启动, 这意味着内核将支持处理器的热插拔。

#### idle线程的初始化

进程0是第一个idle线程, 其他处理器的idle线程都将由它分叉产生。启动处理器会探测其他处理器的存在, 并以适当方式在它们上面初始化并运行每个处理器的idle线程。在初始化idle线程时, 便会设置好其目标处理器为当前运行的处理器。idle线程的目标处理器不会改变, 除非idle线程所在运行的处理器被热插拔卸载。

RISC-V通过devicetree or ACPI tables来探知其他CPU的存在。==.............困的不行了, 反正论文里也不会写这个, 之后再来补吧。==在https://github.com/torvalds/linux/blob/master/arch/riscv/kernel/smpboot.c里。

#### 其他进程的目标处理器设置

进程的cpuid往往被放置在thread_info的cpu成员中。一个新的进程第一次被设置目标处理器是在fork行为时的copy_process中通过对thread_info结构体的复制完成的。但由copy_process所复制出来的目标处理器不会维持太久。内核会为了实现负载均衡在fork/clone行为和execve装载程序时进行负载均衡, 这个过程中内核调度的核心代码(linux/kernel/sched/core.c)中的wake_up_new_task和sched_exec会为进程重新安排目标处理器。



#### ~~RISC-V曾经在上下文切换中显式的切换thread_info::cpu~~

见https://github.com/torvalds/linux/commit/8aa0fb0fbb82a4d2395be7eaeb994653b2d869fc 

==论文中别出现这个。==

但我们估计会采用这个方案, 因为我不打算实现复杂的调度。



#### cpuid与hartid的映射

hartid与Linux内部cpuid有一组映射关系。

CPUID 到 HARTID 的映射由`cpuid_to_hartid_map()`提供。HARTID 到 CPUID 的映射由`riscv_hartid_to_cpuid();`提供。

他们的核心是`__cpuid_to_hartid_map`数组 (在https://github.com/torvalds/linux/blob/master/arch/riscv/include/asm/smp.h中)。

```c
/*
 * Mapping between linux logical cpu index and hartid.
 */
extern unsigned long __cpuid_to_hartid_map[NR_CPUS];
#define cpuid_to_hartid_map(cpu)    __cpuid_to_hartid_map[cpu]
```

它在https://github.com/torvalds/linux/blob/master/arch/riscv/kernel/smp.c被初始化为全-1。

```c
unsigned long __cpuid_to_hartid_map[NR_CPUS] __ro_after_init = {
	[0 ... NR_CPUS-1] = INVALID_HARTID
};
```

其中有一对使用`static inline`声明的函数, 他们用于单核的情况。其中`boot_cpu_hartid`在启动时被设置(head.S ), 它也是启动所使用的hart的hartid。任何CPUID都会映射到`boot_cpu_hartid`  (在https://github.com/torvalds/linux/blob/master/arch/riscv/include/asm/smp.h中)。

```c
static inline int riscv_hartid_to_cpuid(unsigned long hartid)
{
	if (hartid == boot_cpu_hartid)
		return 0;

	return -1;
}
static inline unsigned long cpuid_to_hartid_map(int cpu)
{
	return boot_cpu_hartid;
}
```

多核的情况下: (https://github.com/torvalds/linux/blob/master/arch/riscv/kernel/smp.c)

```c
void __init smp_setup_processor_id(void)
{
	cpuid_to_hartid_map(0) = boot_cpu_hartid;
}

int riscv_hartid_to_cpuid(unsigned long hartid)
{
	int i;

	for (i = 0; i < NR_CPUS; i++)
		if (cpuid_to_hartid_map(i) == hartid)
			return i;

	return -ENOENT;
}
```

另一个函数在上方的宏中:

```c
#define cpuid_to_hartid_map(cpu)    __cpuid_to_hartid_map[cpu]
```









# Idea暂存处



内核依旧保有LDT结构、进程描述符拥有LDT属性、进程私有的LDT可以在fork时被子进程继承, 有修改LDT的系统调用。但与Linux 1.00之前版本中, fork继承/部分继承父亲进程的LDT ( 0.97及之前的版本只继承limit部分, base字段与nr号相关不继承 ), exec中更改LDT的行为不同。Linux 1.00中fork继承父进程的LDT, 但是在exec时会恢复为默认LDT。



在这一点上, Linux 1.00已经与后期版本有异曲同工之妙[^05cn] P50。





### 分页的实现:

且我们希望使用一个确定大小的空间来完成初始化阶段的分页初始化。

我们将分页机制的初始化分解成2个问题: 是否使用大页(huge page), 是否映射全部的内核代码。

如果我们采取最小颗粒度分页的分页机制, 且只映射部分内核代码。优点在于我们能确定其使用的空间大小。缺点是不一定能保证后期用于将所有内核代码都完成映射的函数也被包括在初始化时的范围内。使用连接脚本将对应代码放在尽量靠前的位置或许能够解决, 但是这会给后续版本迭代的兼容性造成潜在的威胁。

如果我们采取最小颗粒度分页的分页机制, 且映射全部内核代码。优点便是一定能保证内核的安全运行。坏处便是内核使用的页表空间不确定。如果设置一个更大的范围, 来确保内核被包含进去, 则有可能引发浪费。如果直接在连接脚本中计算内核代码大小然后计算则会极为复杂, 更重要的是, 日后迭代中如果引入5级的虚拟页表结构可能会直接使得在编译阶段计算出需要多少页表空间称为不可能。

使用巨页的好处是, 每一个页表项能映射非常大的范围。这个范围远超过内核代码当前和未来可能的大小, 自然能确保内核代码空间全被包含。好处是能使用确定大小的空间完成初始化。坏处是当不实现巨页管理机制时, 映射了巨页的页表可能会被丢弃, 造成浪费。

修改Linux RISC-V的启动流程是一个好方案, `trampoline_pg_dir`被用于多核启动, 它不需要回收, 因为可能涉及到CPU的热插拔, 所以不存在浪费。

然后我们延后设置`swapper_pg_dir`的时机到



# 废弃的Idea

进程所运行的CPU由内核调度的核心代码(linux/kernel/sched/core.c)设置。早期Linux仅支持单核, 所以不存在这个概念。现代Linux的调度子系统是及其强大的, 从RISC-V Linux的注释中我们能看出这个设置过程已经是复制的调度、抢占、负载均衡等机制的一部分。总之, 现代Linux能为子进程设置合适的CPU去运行。





# Reference

[^01]: Linux a Portable Operating System
[^02]: [Operating.System.Concepts(9th,2012.12)].Abraham.Silberschatz.文字版
[^03]: Intel® 64 and IA-32 Architectures Software Developer’s Manual Combined Volumes: 1, 2A, 2B, 2C, 2D, 3A, 3B, 3C, 3D, and 4
[^04]: 一个64位操作系统的设计与实现
[^05]: 深入理解Linux内核第三版(英文版)
[^05cn]: Linux内核设计与实现(原书第三版)(中文版)
[^06]: 深入Linux内核架构

[^07]: Linux内核深度解析_ARM64
[^08]: Linux内核完全注释 5.0
[^09]: RV特权级手册

[^10]: i386手册
[^11]: The RISC-V Reader: An Open Architecture Atlas  (RISC-V简明手册的英文版, 正式中文出版物是《RISC-V开放架构设计之道》)
[^11cn]:RISC-V开放架构设计之道
[^12]: 汇编程序设计与计算机体系结构 软件工程师教程
[^13]: 8086手册
[^14]:RISC-V汇编手册 https://github.com/riscv/riscv-asm-manual
[^15]: RV非特权级手册
[^16]: intel有关x86S的宣传 https://www.intel.com/content/www/us/en/developer/articles/technical/envisioning-future-simplified-architecture.html
[^17]: RISC-V  SBI手册

[^18]: RISC-V Kernel Boot Requirements and Constraints https://github.com/torvalds/linux/blob/master/Documentation/arch/riscv/boot.rst









[^TBC]: 待补充对应资料/出处
[^WBL]: Will be Linked, 与文章其他部分可能相关联

[^TBF]: To be filled, 待填充内容
