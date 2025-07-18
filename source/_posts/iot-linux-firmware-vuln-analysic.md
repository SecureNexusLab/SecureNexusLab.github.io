---
title: 看不懂系列——分析检测基于Linux的物联网固件中的漏洞
date: 2024-07-18 14:29
author: SecureNexusLab
cover: true
sidebar: []
readmore: true
tags: 
- 公众号推文
- 二进制安全
categories:
- 公众号推文
---

论文阅读笔记-《更快更好：通过优化的到达定义分析检测基于Linux的物联网固件中的漏洞》

# 1 概述

本文提出了一个高效的污点分析解决方案，即`HermeScan`，用于发现物联网设备固件程序中污点式的漏洞，该方案利用到达定义分析(RDA)进行污点分析，并具有更少的漏报、误报和时间成本。

# 2 背景

## 2.1 应用的场景

![](/images/iot-linux-firmwork-vuln-analysic/1.png)

本文专注于基于Linux的物联网设备中的污点式漏洞，图1描述了污点式漏洞的场景：攻击者可以通过局域网或广域网访问目标设备，并向设备发送任意数据，没有任何限制。数据首先在固件提供的前端文件（如JavaScript和HTML文件）中被解析。然后在设备端，数据被传播到后端的Web服务程序。Web服务程序进一步将数据传输给操作设备的处理程序。在这个过程中，一些库文件被加载以提供必要的支持。后端提供Web服务的二进制程序们通常被称为边界二进制（border binaries），它们通常包括Web服务器和处理程序。Web服务器和处理程序也可以集成到单个二进制文件中，例如路由器供应商经常自定义的`httpd`。

## 2.2 动机示例

![图2 ASUS路由器中0-day漏洞的简化示例](/images/iot-linux-firmwork-vuln-analysic/2.png)

触发漏洞的过程如下：httpd在接收到HTTP请求后查询其注册的`mime_handler`数组。当数组索引指向函数`appGet.cgi`时，它进一步调用函数`do_ej`，在那里它检查HTTP请求中输出字段的值是否存在于由`ej_handler`存储的函数列表中。当值与字符串`bwdpi_monitor_info`匹配时，调用函数`ej_bwdpi_monitor_info`，并通过`websGetVar`函数从用户输入中获取type和event的值。type和event的值最终传递给位于私有库`libbwdpi_sql.so`中的`bwdpi_monitor_info`。在不适当的字符串连接操作之后，可以通过不受限制的用户输入设置命令执行函数的参数。

# 3 挑战

1、全面的控制流图恢复；

2、精确的污点源识别；

3、采用符号执行进行污点分析具有较高的时间开销。

# 4 设计

为了解决这些挑战，`HermeScan`采用了三个关键方案：增强的控制流图恢复、精确的污点源识别和高效的数据流分析。

如图3所示，

`HermeScan`按照以下五个步骤分析固件：

1、边界二进制查找器；

2、增强的控制流图恢复；

3、污点源识别；

4、高效的数据流分析；

5、污点检查引擎。

![图3 HermeScan的概述](/images/iot-linux-firmwork-vuln-analysic/3.png)

## 4.1 增强的控制流图恢复

本文解决了现有方案在函数边界识别、符号名称查找、调用约定恢复等方面的不足，为HermeScan的反汇编数据流分析提供尽可能坚实的基础。

识别的函数越多，分析范围可扩展得越广。因此，除了反汇编器自动识别的函数外，HermeScan还扫描整个二进制文件的反汇编代码，通过匹配不同架构下的函数序言（例如堆栈操作）的特征来识别反汇编器漏掉的函数边界。

符号名称用于识别二进制文件加载了哪些库函数。SaTC和KARONTE都依赖angr通过分析节头表来恢复二进制程序中外部库函数的符号名称。然而，这种方法对于那些剥离了节头表的二进制文件来说是不足的，导致一些函数符号信息被遗漏。HermeScan解析程序头表以恢复符号。首先遍历所有程序头表并识别出与p_type对应的PT_DYNAMIC段，然后在PT_DYNAMIC段中搜索DT_SYMTAB标签的值以定位ELF符号表的地址，最后从ELF符号表中获取更多的符号信息。

HermeScan扩展了angr的默认调用约定并实施了更激进但更完整的恢复策略。对于angr未成功获取的函数调用约定，HermeScan根据程序的架构（例如，MIPS架构的默认参数调用是使用a0-a3寄存器）分配预设的调用约定。为调用约定提供默认值确保了随后的数据流分析更接近可能性分析。

HermeScan利用这些信息增强了CFG的构建。与SaTC和KARONTE采用的CFG构建不同，HermeScan将每个函数视为主节点来构建独立的控制流子图，然后通过跳转或调用指令将它们连接起来。同时，HermeScan为二进制文件加载的共享库建立了Lib-CFG，并利用符号名称将Bin-CFG与Lib-CFG连接起来

## 4.2 污点源识别

包括两个部分。第一部分采用模糊匹配策略来获取更多的候选源函数。第二部分通过检查候选函数，为源函数的返回值或参数分配适当的污点初始值。

HermeScan采用模糊匹配策略筛选前端和后端文件中共享的字符串，同时考虑单词形态相似性和语义相似性匹配关键字。HermeScan识别出引用这些字符串的候选函数，这些函数很可能对外部输入进行解析，并将这些输入存储在函数的参数中或者作为返回值

在识别出候选函数后，先前的工具会假设它们的所有参数和返回值都是存储外部输入的变量，即把它们标记为污点源。然而，将所有参数都视为污点源会引入误报，而将那些不会被后续操作使用的返回值标记为污点源则会导致额外的污点分析开销。在每个候选函数内部，HermeScan通过定义-使用分析跟踪每个参数的数据流。如果指向参数的内存被赋予了某些值，它可能作为接收外部输入的污点源。否则，该参数不是污点源。在候选函数外部，跟踪其返回值的数据流，并检查它是否会被后续操作使用。如果是，返回值将被标记为污点源。为了进一步提高分析的精确度，本文采用了约束推理通过对污点源进行定义-使用分析和值集分析来收集约束信息。

![列表1 用于说明约束推理的伪代码](/images/iot-linux-firmwork-vuln-analysic/4.png)

以列表1为例，更好地阐述约束推理方法。首先，通过定义-使用分析，发现函数dlink_webGetVarN的第三个参数流到了strncpy函数的第三个参数中，而strncpy的目标地址最终流到了函数的返回值中。然后，通过值集分析（VSA）为dlink_webGetVarN函数构建了一个值流图（VFG）。通过获取位于strncpy调用处（第6行）的VFG节点状态，看到strncpy的第三个寄存器值表达为uninitialized_initial_r2+0xffffffff，其中uninitialized_initial_r2代表参数len，而0xffffffff代表-1。相似地，在退出语句的基本块（第9行），存储函数返回值的寄存器表达为v1_(len-1)+00，意味着返回值等于变量v1的值，且值约束为长度为len-1的字符串与一个截断字符的连接。因此，本文可以推断dlink_webGetVarN的污点源约束受到其第三个参数减一的限制。

最终，HermeScan 通过启发式方法根据漏洞类型和收集到的长度约束，在污点源的适当参数或返回寄存器上定义输入污点值。

## 4.3 高效的到达定义分析

本文根据三个原则设计了独特的污点分析方案（LCO Interprocedural Analysis），并采用路径合并策略来缓解路径爆炸问题。

1）LCO跨进程分析

（1）轻量级原则（Lightweight principle）。HermeScan利用基于到达定义分析（RDA）的污点分析，而不是在SaTC和Karonte中使用的基于符号执行的污点分析方法。具体来说，angr的RDA模块为HermeScan提供了最基本的过程内分析。它将汇编指令提升到VEX IR，并在构建的控制流图上使用经典的工作列表算法执行RDA分析。对于感兴趣的变量，生成一个间接的定义-使用（Def-Use）图，其中每个节点代表该变量的一个定义（Def），每条边表示在一个定义之后变量被使用（Use）。在具体实现中，给定的变量被区分为五个类别：临时变量、全局变量、栈、堆和寄存器，这样可以更好地标记在特定地址处变量的定义或使用。

（2）上下文敏感原则（Context-sensitive principle）。数据流分析通常涉及函数调用，因此跨过程分析是必要的。不幸的是，基于angr现成的RDA方法不能应用于实际的固件分析，因为它没有考虑跨过程调用的上下文。因此，HermeScan将angr的过程内RDA扩展到跨过程，并考虑上下文信息以实现细粒度的数据流分析。在程序中，当一个函数被调用时，该函数的参数会被临时存储起来。在这一过程中，函数参数被视为临时变量。这些临时变量的值是从调用该函数的函数（称为调用函数）中获得的，根据特定的调用约定从寄存器或栈中获取相应的值。当返回到调用者函数时，HermeScan合并不同返回地址的临时变量、全局变量、栈、堆和寄存器的定义值，并覆盖调用者函数中的原始值。此外，对于别名问题，如间接调用，HermeScan基于上下文信息从定义-使用图中计算相关寄存器的值，并解析可能的跳转地址进行进一步分析。

（3）按需原则（On-demand principle）。为了提高效率，在跨过程分析期间，静态分析通常采用按需跟踪。现有方法通过判断函数的参数是否被污点化来选择是否进行跟踪。然而，仍然存在两个缺陷：一是依赖符号执行来探索路径的方法受到路径状态存储容量的限制，这使得在嵌套函数中按需跟踪变得困难；二是当涉及到调用外部库函数时，现有方法只总结了常见的C标准库（Libc）函数，并跳过了其他库函数。

HermeScan提出了按需跟踪来解决上述困难。首先，本文的方法通过在分析时更新的数据依赖图来判断一个变量是否被污点化，而不涉及路径存储；其次，本文深入到参数被污点化的库函数中，形成更深入的数据流分析。

2）路径合并策略

HermeScan设计了一种路径合并策略，用以识别和合并在函数调用图中重复经过的路径，从而减少冗余分析工作。

多源污点：对于具有多个源点的函数，现有的固件静态分析方案分别从每个源点作为污点分析起点并独立跟踪其污点传播过程。在RDA的背景下，本文跟踪程序中所有变量的使用-定义分配。因此，HermeScan可以在包含多个输入源点的函数中，用不同的标签污点化每个污点源。这样，HermeScan可以在一次RDA分析中跟踪和区分多个污点值的传播过程，减少了将这些源点作为不同起始位置开始分析的成本。

多汇观察：由于我们使用的是基于路径不敏感的RDA，从调用图开始的任何函数理论上都会被分析。因此，所有包含汇点的函数在一次RDA分析中都被覆盖。HermeScan通过在一次分析中设置多个观察点来避免多次分析这些汇点。

![图4：路径合并策略示意图](/images/iot-linux-firmwork-vuln-analysic/5.png)

图4展示了在不同策略下如何合并路径。启用HermeScan的路径合并策略允许一次RDA传递覆盖函数A中的两个源点和函数A、B和D中的三个汇点，将分析路径的数量从7减少到2。相比之下，SaTC的策略部分合并了重复路径，将分析路径的数量从7减少到3。
