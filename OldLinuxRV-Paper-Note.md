



# Contents

[TOC]

|              | 例子 |
| :----------: | :--: |
|  Linux 0.01  |      |
|  Linux 0.10  |      |
|  Linux 0.11  |      |
|  Linux 0.12  |      |
|  Linux 0.95  |      |
| Linux 0.96c  |      |
|  Linux 0.97  |      |
| Linux 0.97.6 |      |
|  Linux 0.98  |      |
| Linux 0.98.6 |      |
|  Linux 0.99  |      |
|  Linux 1.00  |      |



# Operation System

From: [^01]  P1

> Note that while the term “Linux” is often used to denote the whole system that is built up from the basic kernel and all the system and user applications that are usually running on top of the kernel, in the context of this thesis only the kernel itself is considered. The portability of system and user applications is here considered a totally different issue, even though there are obviously some common concerns.



# Memory Layout



## 研究的问题

From: [^01]  P2

> Linux did not even try to avoid using the x86 features available to those early versions — quite the opposite. Linux started out as a project to find out exactly what you could do with the CPU, and as such it used just about every feature of the CPU you could find ranging from using segments for inter-process protection to hardware assisted process switching.

- Memory Layout (时间划分)

  - Not Early Version

    只存在Virtual Address和Physical Address的概念, 所利用的硬件结构能做到非恒等偏移的映射:

    | 架构    | 映射方法                                                 |
    | ------- | -------------------------------------------------------- |
    | 80x86   | page table(段机制仅满足体系要求)                         |
    | Alpha   | page table                                               |
    | PowerPC | hash table                                               |
    | MIPS    | not have any architecture-specified page tables, BUT TLB |

    而段机制只能做到恒等偏移, 所以即使是早期的Linux也在使用x86段机制的基础上使用了页表机制。

    在研究早期版本Linux时, 由于使用了段机制来分离用户态和内核态, 导致Programer Counter和实际在整个virtual Address space寻址使用的“指针”并不相同。仅用virtual Address和Physical Address不足以描述问题。

    所以引入Abstract Virtual Address和PC related Address来研究问题。

  

  

  - Early Version

    Linux 1.0以及之前的版本都使用段机制来隔离内核和用户态程序。所以分析问题分两个步骤:

    - 首先分析内核和用户态程序在Abstract Virtual Address和PC related Address视角分别是如何分布的。
      - 段机制视角
      - 分页机制视角(页表的私有化问题)
    - 接着分析用户态和内核态空间中各子项的分布情况。



## 不同早期版本的Linux的内存布局

任务号以 nr 表示

仅讨论段机制造成的Abstract Virtual Address空间上的布局

|              |     内核起点      | 内核范围(长度) | 内核-注释  | 进程起点                              | 进程起点-注释 | 进程范围(长度)                                               | 进程范围-注释                                         |
| :----------: | :---------------: | -------------- | ---------- | ------------------------------------- | ------------- | ------------------------------------------------------------ | ----------------------------------------------------- |
|  Linux 0.01  |        0x0        | 8MiB 可扩展    | head.s末尾 | nr*0x400_0000                         | 64MiB         | code_limit = 参数相关<br />data_limit = 0x400_0000           | code选择子0x0f<br />data选择子0x17                    |
|  Linux 0.10  |        0x0        | 16MiB 可扩展   | head.s末尾 | nr*0x400_0000                         |               | 同上                                                         |                                                       |
|  Linux 0.11  |        0x0        | 16MiB 可扩展   | head.s末尾 | nr*0x400_0000                         |               | 同上                                                         |                                                       |
|  Linux 0.12  |        0x0        | 16MiB 可扩展   | head.s末尾 | nr*TASK_SIZE (TASK_SIZE = 0x400_0000) |               | code_limit = data_limit = TASK_SIZE                          |                                                       |
|  Linux 0.95  |        0x0        | 16MiB 可扩展   | head.s末尾 | nr*TASK_SIZE (TASK_SIZE = 0x400_0000) |               | code_limit = data_limit = TASK_SIZE                          |                                                       |
| Linux 0.96c  |        0x0        | 16MiB 可扩展   | head.s末尾 | nr*TASK_SIZE (TASK_SIZE = 0x400_0000) |               | code_limit = data_limit = TASK_SIZE                          |                                                       |
|  Linux 0.97  |        0x0        | 16MiB 可扩展   | head.s末尾 | nr*TASK_SIZE (TASK_SIZE = 0x400_0000) |               | code_limit = data_limit = TASK_SIZE                          |                                                       |
| Linux 0.97.6 | 0xc0000000 (3GiB) | 16MiB 可扩展   | head.s末尾 | 0x0                                   |               | code_limit = data_limit = TASK_SIZE<br />(TASK_SIZE = 0xC000_0000) | 3GiB                                                  |
|  Linux 0.98  | 0xc0000000 (3GiB) | 1GiB           | head.s末尾 | 0x0                                   |               | code_limit = data_limit = TASK_SIZE<br />(TASK_SIZE = 0xC000_0000) |                                                       |
| Linux 0.98.6 | 0xc0000000 (3GiB) | 1GiB           | head.s末尾 | 0x0                                   |               | code_limit = data_limit = TASK_SIZE<br />(TASK_SIZE = 0xC000_0000) |                                                       |
|  Linux 0.99  | 0xc0000000 (3GiB) | 1GiB           | head.s末尾 | 0x0                                   |               | code_limit = data_limit = TASK_SIZE<br />(TASK_SIZE = 0xC000_0000) |                                                       |
|  Linux 1.00  | 0xc0000000 (3GiB) | 1GiB           | head.s末尾 | 0x0                                   | GDT而非LDT    | code_limit = data_limit = TASK_SIZE<br />(TASK_SIZE = 0xC000_0000) | #define USER_CS    0x23 <br />#define USER_DS    0x2B |

### 关于内核范围可扩展性:

Linux 0.01 ~ Linux 0.97在GDT的内核范围设计上默认为16MiB, 在页表可能冲突的层面上最大可以接受64MiB的范围。由`TASK_SIZE`所影响。Linux早期将`TASK_SIZE`设为64。(4GiB / TASK_SIZE = 64MiB ) 同时仅有一份(总)页表, 前端的内存空间只能由一级页表的前端做恒等映射。这一段虚拟地址并不能跟task1的段虚拟地址冲突。

Linux 0.97.6在GDT的内核范围设计上默认为16MiB, 理论上可以拥有1GiB的空间。

PS: 内核内存范围大小和实际物理内存的大小不能划等号。Linux 0.97.6及以前, 物理内存的大小与head.S中的页表设置和GDT设置相关。Linux 0.98中物理内存的大小与head.S中的页表设置相关。而到了Linux 0.98.6中, 物理内存大小才与head.S中的设置解耦。

> 在注释中提及Linus自己的电脑只有16Mb, 所以如果要做出更改, 需要(search for "16Mb")并做修改即可使用更大的内存。

尽管从Linux 0.98.6到Linux 1.00的时间段内, 物理内存大小才与head.S中的设置解耦, 但是开发者们关注的仍然是16MiB、32MiB这种级别的物理内存大小的问题。简言之, 早期的内核远没有到3:1分配下x86架构考虑896MiB的MAXMEM问题的时候[^06]。 



### 用户态进程起点与范围

Linux 1.00及之前用户态进程的起点与范围的问题, 可划分为几个阶段。

Linux 1.00开始用户态进程代码段和数据段开始使用GDT选择子, 而非之前版本所使用的LDT选择子。这体现在用户态进程(实际)使用的段选择子上。( `INIT_TASK`宏中装在的段选择子并不是最后实际使用的段选择子, 这点体现在``move_to_user_mode()``宏中 ) ( Linux 1.00中存在很多与LDT相关的代码, 这点我们在后面在做说明 )

在使用LDT作为用户态选择子的版本中, 用户态进程起点与范围集中体现在 `INIT_TASK`宏、fork系统调用和exec系统调用的相关代码上。

首先, 可以以用户态是否拥有3GiB的段长度为划分。或者等价于由于用户态拥是否有3GiB段长而导致的(总)页表(即页表目录)是否私有为划分。这一变化出现在Linux 0.97与Linux0.97.6之间。在Linux 0.97.6及之后的版本, 用户态的段机制便都以0x0为起点, 段长为3GiB。在Linux 0.97及之前的版本中, 段起点与nr号相关, nr*0x400_0000。(nr为)0号进程的段长为640KiB, 所有fork于它的子进程的段长在由exec系统调用更改前均继承于(nr为)0号进程的长度。

而exec中为修改的新段limit中, 可以从code_limit和data_limit是否相等来做进一步细节上的区分: 0.01~0.97中数据段的长度都为64MiB, 而代码段的长度设置上却有些区别。0.01、0.10和0.11中, 代码段的长度来由`change_ldt` 函数中的 `text_size`参数决定; 在此之后(直至0.99), 都是直接在`change_ldt`中令code_limit = data_limit = TASK_SIZE(只是TASK_SIZE宏的数值有区别, 如同前文所述, 0.97及之前为64MiB, 0.97.6及之后为3GiB)。(此期间`change_ldt` 函数保留了 `text_size`参数, 但是实际上并未在函数内使用这个参数)

PS: `change_ldt`存在于我们所列出的所有早期版本中。但是在Linux 1.00彻底脱离了重设用户态进程LDT的用途。(因为Linux 1.00使用GDT为用户态所用) (且Linux 1.00中 `change_ldt` 与`sys_modify_ldt`的实现无关)



### Linux 1.00中LDT相关的解释

Linux 1.00中, 依旧保有LDT结构、进程struct拥有LDT属性( 面向对象编程的属性的概念 )、进程私有的LDT可以在fork时被子进程继承, 有修改LDT的系统调用。但与Linux 1.00之前版本中, fork继承/部分继承父亲进程的LDT ( 0.97及之前的版本只继承limit部分, base字段与nr号相关不继承 ), exec中更改LDT的行为不同。Linux 1.00中fork继承父进程的LDT, 但是在exec时会恢复为默认LDT。

一般情况下, Linux 1.00中所有进程私有的LDT表项都实际指向`&default_ldt`, 只有在调用`sys_modify_ldt`的`write_ldt`操作时, 会分配一个新的空间来存放一个新的LDT表。( 注: 子进程fork拥有非默认LDT的父进程时, 对LDT的复制采取的是重新分配空间的深拷贝 )

但是无论是`default_ldt`还是新分配的LDT, 进程都没有实际的使用它们。这一点很好地体现在进程的段选择子上。

















# 1  Introduction

- 本文是什么
  - 操作系统是什么
  - 内核 + 部分软件
- 内核
  - 硬件侧和软件侧
  - 硬件侧分为 CPU subsystem (including memory management, caches and optionally multiprocessing details) and the architecture of the I/O interfaces
  - 软件侧 子集 + 符合现代体系

This paper is an introduction to '以Linux 1.0为蓝本的在RISC-V 上运行的操作系统', OldLinuxRV.

~~There is no completely adequate definition of *operating system*. A more common definition, and the one that I follow in this paper, is that the *operating system* can be equated to the *kernel*[^02]. Another viewpoint is that the definition of operating system should not be limited to the term *kernel*. It should also include system programs along with the kernel. If the latter definition is adopted, then defining what is a system program itself will become the next difficult problem, because the boundaries between system programs and application programs are blurred. This paper will focus on implementing the kernel. In addition, some system programs and application programs, which are not the focus of this paper, will also be implemented to demonstrate the various functions of this operating system, OldLinuxRV.~~

~~Linus Torvalds pointed out that *the kernel itself is defined by both the hardware it runs on and the software it supports*[^01]. 从硬件侧定义OldLinuxRV, 最鲜明的特点在于它运行在RISC-V体系结构的一个子集上面。从软件侧来讲, 它会去兼容Linux1.0阶段~~

- *Hence, we often use the term "operating system" as a synonym for "kernel".* [^05] P8
- *用户见面是操作系统的外在表象, 内核才是操作系统的内在核心。*[^06cn] P4







RISC-V Architecture 作为一种新兴的体系结构, ...。虽然已经发展了几年, 当下RISC-V architecture仍处于发展的初期。仍有许多标准处于crafting的状态, 如, crafting中的CLIC被用于补足曾经被设计用于处理外设中断的PLIC。

但尽管如此, 得益于





## 1.1 ???	

Linus先生是这样描述操作系统内核的: *"As such, the kernel itself is defined by both the hardware it runs on and the software it supports."*。



# 2 Basic Kernel Framework

​	Linus先生是这样描述操作系统内核的: *"As such, the kernel itself is defined by both the hardware it runs on and the software it supports."*。

*"Memory is one of the most fundamental resources ~~in the system~~"* of a computer.如何分配内存(memory management问题)是及其重要的, 但是在这之前, 必须先确定一个明确的Memory Layout. 在使用virtual memory机制的体系结构上, Memory Layout问题本质上就是提出一个合理的physical memory 到virtual memory的mapping scheme.

​	

​	

Memory Layout 问题在RISC-V体系架构上 [^01]

## 2.1 Memory Layout

​	Memory Layout问题本质上就是提出一个合理的physical memory 到virtual memory的mapping scheme.

​	显然, 我们需要研究的子问题自然是virtual memory和physical memory的使用情况。其中virtual memory的使用情况是研究内核程序和用户态进程程序在virtual address space([^01] 4.1.1 指的是抽象的)中的布局情况。physical memory则是研究内核究竟如何管理和使用physical address space。

​	此处的virtual address和virtual memory

​	此处的virtual address和physical address都是研究内存问题时的抽象概念。virtual address是 一种CPU 理论上能合法寻址的能力。而physical address则是*Used to address memory cells in memory chips* ([^05] P36). 而如何实现他们之间的映射的细节, 我们在这个视角层面上并不关心。



### 2.1.1 kernel and user in virtual address space

​	请注意virtual address space和virtual address的区别。virtual address的一词在本文中的定义是由页表

内存布局问题是研究内核程序和用户态进程程序在内存地址空间中如何布局的问题。

为了避免歧义, 需要特别说明此处“内存地址空间”和“程序”两个术语。

此处“内存地址空间”是指某体系结构下, 理论上能合法寻址的一个寻址空间。例如在80x86体系结构的保护模式中, 合理利用段机制和分页机制, 逻辑地址理论上能获得0~4GiB的寻址能力。而在x64体系结构的IA-32e模式下, 线性地址的位宽的64位中低48位用于线性地址寻址、高16位为符号扩展, 理论上的合法寻址范围是0x00000000,00000000~0x00007FFF,FFFFFFFF和0x00008000,00000000~0x FFFFFFFF,FFFFFFFF。

- 此处讨论的Memory Layout

- 内存布局问题简单而言就是kernel和进程分别从内存的何处开始, 何处结束的问题。

### 内存有关术语名称问题

- 页表名称问题。Linux0.01 - Linux-1.00中, 分别使用了“页表目录”和“页表”的概念。本文则不做此区分, 只区分页表的级数。例如x86的“页表目录”本文成为“一级页表”, 而x86的“页表”本文称为“二级页表”。本文中“页表”指的是由所有级数的页表组成的、用于虚拟内存转换到物理内存的整体结构。

- 关于i386的内存相关术语有些混乱, 分别是线性地址、逻辑地址、虚拟地址、物理地址。我们依据[^03] (Volume 3 3.1)的表述segmentation and paging。其中, 由segmentation机制联系着线性地址、逻辑地址两个概; 将最终用由paging机制联系着虚拟地址和物理地址。线性地址被看作是段内偏移(offset), 其寻址空间是理论最大内存空间的子集。仅当处于Protected Flat Model([^03] 3.2.2)时, 其寻址空间和逻辑内存相同, 即整个理论最大内存空间。因此, 如果本文提及i386的“地址”这一概念, 其指的是由逻辑地址计算出的在整个内存空间中的地址。i386的“地址”再由是否开启页表分为虚拟地址和物理地址。

  (并不采用[^04]6.2.1中的说法, 即*虚拟地址(VirtualAddress)是抽象地址的统称，它们大多不能独立转换为物理地址，像逻辑地址、有效地址、线性地址和平坦地址皆属于虚拟地址的管理范畴*。 )

- i386相较RISC-V而言, 多了segmentation机制。

  - 从整个内存空间or内存物理取址 => logic
  - 编译器角度or程序执行角度or PC(Program Counter) 角度 => Linear

- 

### 内存模型区别

"段式存储器模型"的英文术语是 "Segmented Memory Model"，而"分页式存储器模型"的英文术语是 "Paged Memory Model"。



sched_init() in sched.c



```
set_ldt_desc
#define _LDT(n)
#define TASK_SIZE
```



```C
// Linux 1.00
// In sys_fork
// 第一次对p->ldt检测, 实际检测的是父ldt
if 父ldt != NULL:
	分配空间
  if 分配成功:
			拷贝
  else:
			失败留到稍后处理

...
if p->ldt != NULL: // 之前分配成功
	设置
else:
	设置默认的ldt

```







# Reference

[^01]: Linux a Portable Operating System
[^02]: [Operating.System.Concepts(9th,2012.12)].Abraham.Silberschatz.文字版
[^03]: Intel® 64 and IA-32 Architectures Software Developer’s Manual Combined Volumes: 1, 2A, 2B, 2C, 2D, 3A, 3B, 3C, 3D, and 4
[^04]: 一个64位操作系统的设计与实现
[^05]: 深入理解Linux内核第三版(英文版)
[^05cn]:Linux内核设计与实现(原书第三版)(中文版)
[^06]: 深入Linux内核架构

