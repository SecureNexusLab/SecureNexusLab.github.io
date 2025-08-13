---
title: 如何利用EDA的bug隐藏硬件木马？解读Lost in Translation/MiRTL攻击（一）
date: 2025-08-13 20:31:03
cover: true
sidebar: []
readmore: true
tags: 
- 公众号推文
- Web安全
categories:
- 公众号推文
---

# 如何利用EDA的bug隐藏硬件木马？解读Lost in Translation/MiRTL攻击（一）

发布时间: 

> 最近发现了一篇论文解读，写得很好，经过作者知乎@cassuto 同意，分享给大家

MiRTL 攻击是在'Lost in Translation: Enabling Confused Deputy Attacks on EDA
Software with TransFuzz'这篇论文1中首次提出的。论文作者为 ETH Zurich 的 Flavien Solt 和 Kaveh
Razavi，发表在 USENIX
Sec'2025。这篇论文是今年上半年我读过的非常强的工作之一。下面尝试根据自己理解写一篇解读，多有疏漏之处，恳请批评指正。

> 题外话，之前关注 CellIFT2 、Cascade3、μCFI4等工作很久了，后来才注意到这些都是 Flavien Solt
> 他们主导或参与的，真心羡慕，不知道 8 月西雅图开会能不能见到作者本尊｡ﾟヽ(ﾟ´Д`)ﾉﾟ｡

开源代码：HTTPS://GitHub.com/comsec-
group/mirtl，项目主页：HTTPS://comsec.ethz.ch/research/hardware-design-
security/mirtl/

## 0x1 EDA 存在什么问题？

我们知道芯片是 IT 设施**信任链** 的一环。不受信任的厂商可能在电路中植入硬件木马（HT,Hardware
Trojan），从而在硬件层面直接取得系统的较高特权，威胁机密性/完整性/可用性。这大概也是在敏感领域需要坚持走自主可控路线的一条论据。

站在敌手角度看，HT 致命弱点在于白盒 HT 检测，而这个领域已经发展了约 17 年。HT 为了逃过检测，通常将触发条件预设得极为稀疏。相对于整个state
space而言，触发条件只占极小部分，使防御方搜到 HT 的概率极低。然而，白盒的防御方可以通过 SAT 求解器利用安全属性自动找出 HT
的触发条件。又比如，白盒的防御方也可以将系统中高敏感数据标记为污点，通过信息流跟踪（IFT,Information Flow
Track）发现将高敏感数据流连接到低敏感数据流的 HT。

在白盒条件 HT 逃逸如此困难的情况下，Lost in Translation 这篇文章给出一个有趣的发现：**EDA** 某些环节的 bug 有可能被
HT 利用来逃过白盒检测。如下图，传统 IC flow 包含仿真/验证、综合两条分支。假设敌手能够构造一类 HT，利用仿真/验证/综合工具的 bug 将
HT 识别为死代码，但 HT 在正常的综合器中仍然是活的，那么 HT 就会逃过检测（图 a）

![原文Figure3 两种威胁模型](/images/DEA-Bug-MiRTL-Attack/wechat_img_1755088427962.png)原文Figure3 两种威胁模型

图 b 的威胁模型就更有意思了，敌手先植入一个特殊设计的哑的 HT（逻辑上是死代码），于是防御方拿各种工具测个遍也找不到该漏洞。但只要带特定 bug
的综合器就能让 HT 变成活的。更强大的是，敌手可以兼容多种带 bug 的综合器，其中任意一种综合器都能复活 HT（仿真/验证工具同理）。

p.s. 论文作者指出虽然在 post-synthesis 阶段做 LEC 理论上可能阻止这种逃逸攻击，但目前这仍然缺乏 scalability。而且
post-synthesis 阶段的 EDA 工具依旧面临类似 bug 的威胁。

## 0x2 为何是 EDA bug？

这就要先介绍论文的关键 idea：假设敌手能够构造一类特殊的电路（称为 MiRTL gadget），它经过仿真/综合后实际输出值取决于 EDA 是否存在特定
bug。例如，一个存在该 bug 的仿真/验证/综合工具看到该电路的输出稳定到 0，而没有该 bug 的工具看到的输出稳定到 1。

我们不妨想想这样构造 gadget 背后的直觉？

> 论文没有说明，但或许我们可以类比软件木马绕过安全检测的手段。软件木马通常会采用反调试、反沙箱等技术，这些技术依赖于一个函数
> 感知程序是否运行在调试器/沙箱等安全分析工具的上下文中，如果  感知到了分析工具的上下文特征而返回 0，那么木马实际发挥作用的 payload
> 就会变为死代码。例如：


    if (f() && trigger_cond) {  
      payload();  
    }

> 除非实施爆破，否则调试器/沙箱很难触发 payload。电路中也一样，那个 MiRTL gadget 本质上承担了上述
> f()的功能，即电路感知自己是否被带有特定 bug 的 EDA 仿真/综合，以决定是否让 HT 变成死代码/冗余逻辑。

理论上敌手一旦拥有这种 gadget 就具备了逃逸能力。对于 Figure 3a，只要将检测buggy仿真/验证工具的  连到 AND 门阻挡 HT
触发信号的传播，就能让仿真/验证工具看不到活动的木马，一旦 HT 的触发条件变成逻辑上不可满足后，EDA 甚至可能直接就把它优化掉了（基于 CP &
DCE）；同理，对于 Figure 3b，只要将检测buggy综合器的 f 连到 AND 门就能让仿真/验证工具看到是死代码的 HT 在综合器中复活。

我们拿论文中例子说明：

![利用MiRTL gadget隐藏HT的实例](/images/DEA-Bug-MiRTL-Attack/wechat_img_1755088428302.png)利用MiRTL
gadget隐藏HT的实例

a)是受害 cpu 的一个数据通路，s 决定了 cpu 是否运行在特权模式，只有特权模式下秘密数据才能进入 q。

b)被挂了一个权限提升木马，只要它的触发条件 r=1（例如执行某条预设 opcode 的指令），哪怕在非特权模式下秘密数据也会进入 q。当然我们知道这个
HT 很可能被当前白盒检测技术（如 CellIFT）发现。

C)中利用上述 gadget 隐藏 HT。存在 translation bug 的仿真/验证工具中看到  的输出为 0，这使得 r 被 AND 门
gated 了无法传播，于是工具认为那个 HT 永远不会触发。然而，在正常的综合器中， 的输出还是理论值 1，这样 HT 实际上仍然能够被 r
触发。这就达到了 Figure 3a 的效果。同理不难想象如何实现 Figure 3b。

> 我们不妨再验证一下这种攻击是否能逃过 IFT。AND 的污染传播公式为:
>
> 其中 分别是信号的标志。考虑到这是一个提权马，触发信号来自非特权，因此。假设 MiRTL gadget 成功触发 IFT 检验工具的
> bug，那么，于是污染传播结果为，可以只要就断开污染传播路径；换句话说，除非工具能够找到 MiRTL gadget 将 标记为 1，否则 HT 就逃过了
> IFT 检测。

如何让 HT 兼容多种不同含 bug 的 EDA 工具呢？只需为每种 EDA 分别构造一个 gadget，然后将这些 gadget 的结果 AND
起来就可以。这样只要任意一个 gadget 输出为 0 就可以让 HT 变成死代码。这与软件木马中组合多种反调试手段是类似的。

> 后记：个人觉得软件类比进一步揭示了如下 insight，也回答小标题的问题：  
> （1） EDA 的 bug 只是用来判断电路在仿真/验证工具还是在综合工具中的行为特征之一。实际上能判断 EDA 的特征可能不止 bug，也可能包含
> feature 差异等；  
> （2） 现在 HT 检测工具中没有类似检测软件木马那样的爆破反调试/反沙箱机制。对于依赖 bug 的 gadget，实现这种爆破等同于修复
> bug。对于依赖 feature 差异实现的 gadget（如果真的存在），那可能需要准确定位 gadget，将其输出强制置为 1 来实现爆破。

## 0x3 利用哪些 EDA bug？

我们看到作者观察了 322 个开源 EDA 中已报告的 bug

![报告的开源EDA中bug数量统计](/images/DEA-Bug-MiRTL-Attack/wechat_img_1755088428597.png)报告的开源EDA中bug数量统计

这些 bug 中有一类 translation bug 非常有趣，它意味着用户写的 HDL
代码有可能跟仿真/验证/综合算法所使用的中间表示（IR,Intermediate Representation）不一致，造成所见非所得。直觉上这跟隐藏 HT
很有关联。

> 其它 bug 大多只能让 EDA 报错或者 DoS，似乎跟隐藏 HT 关联不明显，也侧面印证了这点

举例来说，在验证工具内部，用户给定的 RTL 设计首先经过 translation passes 翻译为 IR，这个过程包含了各种优化（例如
CP、DCE、CSE 等等），然后在 IR 上运行验证算法，如下图：

![验证工具内部的前端和后端](/images/DEA-Bug-MiRTL-Attack/wechat_img_1755088428829.png)验证工具内部的前端和后端

> 注：对于综合器，translation 一直延伸到最终产生网表（也可以看成一种 IR），这个过程包含了各种 PPA 优化算法

Translation bug 意味着上述 translation passes 生成的 IR 与 RTL
设计不一致，直接导致仿真/验证/综合结果不符合用户期望和规范。我们可以从 translation bug 案例中建立更好的直觉：

![](/images/DEA-Bug-MiRTL-Attack/wechat_img_1755088429087.png)

部分 translation bug 可以用来构造 MiRTL gadget，从而按 0x2 章介绍的那样实现逃逸攻击。拿作者发现的一个新
bug（V6，CVE-2024-25493）举例，这个 bug 是这样的：Wrong not when checking (n)eq under
and/or tree，而触发这个 bug 的电路长这样：

![利用V6的MIRTL gadget](/images/DEA-Bug-MiRTL-Attack/wechat_img_1755088429593.png)利用V6的MIRTL gadget

> 为啥这个电路长这么奇怪？因为这是基于 fuzzer 随机生成的结果，后面会详细介绍

理论上这个电路的输出肯定恒为 1。但是 Verilator 存在 bug，使得这个电路仿真结果竟然是取反的，也就是输出为 0。

> 这有些奇怪，所以我特意去 verilator 的 GitHub 仓库看了下，确实看到了作者提的
> issue：HTTPS://GitHub.com/verilator/verilator/issues/4857，还在 id 为 4a0e5a 的
> commit 修复了这个 bug，作者还提供了复现 bug 的
> PoC：HTTPS://GitHub.com/flaviens/verilator-b14/tree/master，赞

这种 gadget，作者共举例了 6 个（6/20 个新发现的 translation bug），并且按照 0x2 那样演示了如何在 CVA6 RISC-V
处理器中植入提权马。

> 最后，我还是好奇作者是怎么想到 MiRTL 这个idea的。follow 该作者之前工作或许能发现蛛丝马迹。他们 2024 年的另一篇非常强的工作
> Cascade 发现并报告了 yosys 存在一个 bug，但可惜的是那篇文章并没有深入讨论这个 bug 的影响，而这篇文章填补了这个 gap

## 0x4 怎么找 EDA bug？

我们通过案例了解利用 translation bug 实现 HT 检测逃逸的原理。但一个问题摆在眼前：如何高效找到 EDA 软件中的 bug？

作者的回答是利用差分 fuzzing 自动挖掘 EDA 软件的 bug。Fuzzing 是一种非常传统的技术了，它可以通过大量随机生成的 HDL
和激励来测试仿真器/综合器。

问题是，既然听起来比较直观，那**实现这种 fuzzing 有什么挑战呢** ？作者主要给出 3 点挑战和解决办法：

**挑战 1** ：作者认为需要找到一种合适的抽象层次来描述硬件设计。其实在此之前已经有直接生成 RTL HDL 的 fuzzer
了（Verismith），然而作者认为现有直接生成 HDL 的 fuzzer 偏向于寻找仿真器前端的 bug（关于词法语法什么的？），而不擅长找后端的
bug。于是作者继承了先前工作 CellIFT 中 macro cell 的概念，生成一种类似网表的 RTL 代码，它更注重电路结构而不是 HDL
写法。不过作者也承认 macro cell 这种类网表的代码的缺点是缺少行为级的描述，从而不能找到语义相关的 bug。

这里还隐含一些技术问题：如何确保随机生成的 RTL 是 sound 的，规避掉一些工程上有不确定性的情况，例如组合逻辑环、多重驱动、pin 悬空、不定态 X
传播、异步电路等。实现上还是基于各种启发式规则，例如把电路图限制为 DAG。

为了生成 verilog 测试用例，作者提出了一个非常聪明的办法，将 Python 生产的网表传递给 yosys 中一个自定义的 pass，由这个 pass
解析出 IR，传递给已有的 verilog 后端生成.v 文件。

**挑战 2** ：检测 bug 的发生存在困难。我认为这个比较关键，传统差分测试比较 DUT 和 GRM（Golden Reference
Model）之间结果的差异容易发现 bug，但是首先得有个 GRM。综合器/仿真器很难搞到靠谱的 GRM。于是作者提出差分
fuzzing，它融合了两种思路，一是观察不同仿真器/综合器的结果，多数表决投票；另一种是关闭 EDA 的所有优化，然后看结果怎么演变。下图 a、b
是面向仿真器的差分 fuzzing，图 C 是面向综合器的差分 fuzzing

![](/images/DEA-Bug-MiRTL-Attack/wechat_img_1755088429813.png)

> 注意到在测试综合器时，最后还是基于仿真结果来判断逻辑等价性（测试驱动）。此处不用 LEC 也许是出于 scalability 的考虑

这里还隐含一些 technical challenge：比较仿真结果就意味着需要拿到电路中的电平取值，然而监视每个时钟周期所有信号的取值非常昂贵，这与
fuzzing 需要尽快测试大量测试用例矛盾。作者其实只监视了电路的输出，并且将它们每个时钟周期的取值累计哈希形成签名，最后只需比较一次签名即可。

> 严格来说这样可能造成 false negatives，不过比较开销确实明显减少了，也许那些漏掉的 bug 能够通过相同时间生成的更多测试用例补回来。

挑战 3：如何利用发现的 bug。fuzzing 发现的触发 bug 的 verilog 可能包含冗余的部分，需要剪枝找出最小的能触发 bug 的电路，构造
MiRTL gadget，最后在实际场景中演示可行性。

面向上述challenge，作者提出了**TRANSFUZZ** ：

![](/images/DEA-Bug-MiRTL-Attack/wechat_img_1755088430149.png)

首先看 **Test case generation** 阶段，这是一个 fuzzer 比较核心的部分。我们知道 fuzzer
覆盖率主要取决于生成的测试用例质量。TRANSFUZZ 利用了已有 bug 的先验知识来设计测试用例的生成算法。

作者观察了 Verilator, Icarus 和 Yosys 的 100 个 bug 以及 CXXRTL 三年内的所有 bug，总结出一些先验：

**观察 1** ：触发 translation bug 需要多种算子类型（移位器、多路选择器、比较器、与或非等等）——需要在较宽范围内随机选择 macro
cell 的算子类型。

**观察 2** ：触发 translation bug 需要 non-trivial 的互联（多种算子同时组成不同的连接方式），——需要随机选择 cell
的输入形成互联。为了保护互联关系的多样性，TRANSFUZZ 支持把线拼接起来构成更宽的向量，这样一个 cell 的输入可能是之前生成的若干 cell
输出的拼接（cell 的 pin 可以是多位的向量），同时也支持从位向量中 slice 出部分 bits。此外，算法还偏向于尚未连接过的
cell，从而减少悬空的输出。

**观察 3** ：发现 22 个 translation bug 中 21 个的操作数宽度大多位于 2~4bit 内。——要求 cell
的操作数宽度偏向于少量 bit。TRANSFUZZ 从概率分布：

来采样操作数位宽，使得算法偏向选择小于 10bit 的操作数宽度，同时为 32bit 以上也保留了 2% 的概率。

基于上述规则，当一个 macro cell
的算子类型、操作数宽度、输入输出等都通过随机采样确定时，就最终实例化为网表的一部分。反复重复上述实例化过程就得到了类似下图的全随机的 subnet：

![subnet](/images/DEA-Bug-MiRTL-Attack/wechat_img_1755088430487.png)subnet

多个 subnet 随机互联起来，形成整个电路图。在这个过程中，TRANSFUZZ
为电路添加异步信号（时钟/复位等位于敏感列表中的信号），并且随机化测试激励。这里激励是由元组(subnet_id,
input_values)构成的时间序列，subnet_id 指示电路中的线、input_values 指出要赋的值。

在 **HDL generation** 阶段，上述电路和测试激励传递到改装的 yosys 前端中，由自定义的 pass 转换到 IR，最后借助 yosys
的后端转换到 verilog 测试用例。

在 **Execution** 阶段，每一份随机生成的 verilog 测试用例交由多个综合/仿真工具处理，最后检查输出签名的一致性以判断是否出现 bug。

最后在 **Characterization & reduction** 阶段，所有触发 bug 的 verilog 测试用例会被进一步简化，形成最终的
MiRTL gadget。

## 0x5 TRANSFUZZ 挖掘 bug 的成绩

TRANSFUZZ 利用 128 进程在约 24h 内共发现 31 个新的 EDA bug。如下图，可以看到在起初 2 小时内它发现了大部分
bug，而随着时间推移 bug 发现逐渐稀疏，最后约 12h 内没有发现任何 bug。这跟一般 fuzzer 的规律一致。

![发现bug的累积时间](/images/DEA-Bug-MiRTL-Attack/wechat_img_1755088430689.png)发现bug的累积时间

下面是发现的 31 个新 bug，其中 20 个是 translation bug（下划线），而论文演示了分别利用其中 6 个 translation
bug 构造的 MiRTL gadget（黄色高亮）。我们看到 TRANSFUZZ 也能找到其它类型的 bug，它们落在了
exception、slow/hang 等分类上，并没有用于构造 MiRTL gadget。

![](/images/DEA-Bug-MiRTL-Attack/wechat_img_1755088430942.png)

## 0x6 实用性如何？

论文演示了利用上述 MiRTL gadget 隐藏 CVA6 RISC-V CPU 中的木马。

我们知道在 Sv39 虚拟内存系统中，PTE 包含一个标志位 U 代表用户可访问。启用虚拟内存后，用户态程序尝试访问 U=0 的页面会报异常从而被 OS
拦下来。要演示的木马希望用户态程序也能访问 U=0 的页面，从而实现权限提升，同时利用 MiRTL 加固该木马以绕过检测。

为此，他们修改了 FMV.X.D 指令实现，让它在操作数为 0x12345678badcable 时触发木马，允许从 U=0 页面访问数据。具体是改了
i_cva6_mmu 里面权限检查的代码：

![待修改的代码](/images/DEA-Bug-MiRTL-Attack/wechat_img_1755088431446.png)待修改的代码

敌手把上面代码改了一下，用!dtlb_pte_q.u && (~MiRTL gadget || ~trigger_cond)信号 AND
了一下，替换掉原来的!dtlb_pte_q.u，其中 trigger_cond 指示上一条指令是否为带操作数为 0x12345678badcable 的
FMV.X.D 指令。这样就完成了 HT 植入，HT将在MiRTL gadget输出1时活动。

然后他们把 Figure3 中两种场景都评估了一遍。一种场景是敌手直接破坏了 RTL，但是依赖为仿真器设计的 gadget
来隐藏木马；另一种是敌手提交一个不包含漏洞的设计，但是依赖综合器 gadget 引入漏洞。实验结果表明两种场景都可行，对 PPA 的影响较小：

![](/images/DEA-Bug-MiRTL-Attack/wechat_img_1755088431775.png)

## 0x7 未完待续

由于时间仓促，本篇只能大概介绍 MiRTL 攻击的总体思路和结果。

原始论文中还有大量实现细节和 tricks、以及 TRANSFUZZ 性能评估对比的内容没来得及详细介绍，希望以后有时间能补上。

## 免责声明

本文内容均来自互联网公开内容，仅限学术探讨。读者需知晓相关规定和风险，引发后果与本文作者无关。

参考

    1. 1\. F. Solt and K. Razavi, ‘Lost in Translation: Enabling Confused Deputy Attacks on EDA Software with TransFuzz’, USENIX Security. Paper= https://comsec. ethz. ch/wp-content/files/mirtl_sec25. pdfURL= https://comsec. ethz. ch/mirtl, 2025.
    2. 2\. F. Solt, B. Gras, and K. Razavi, ‘CellIFT: Leveraging Cells for Scalable and Precise Dynamic Information Flow Tracking in RTL’, in 31st USENIX Security Symposium (USENIX Security 22), Boston, MA: USENIX Association, Aug. 2022, pp. 2549–2566. [Online]. Available: https://www.usenix.org/conference/usenixsecurity22/presentation/solt
    3. 3\. Solt, K. Ceesay-Seitz, and K. Razavi, ‘Cascade: CPU Fuzzing via Intricate Program Generation’, in 33rd USENIX Security Symposium (USENIX Security 24), Philadelphia, PA: USENIX Association, Aug. 2024, pp. 5341–5358. [Online]. Available: https://www.usenix.org/conference/usenixsecurity24/presentation/solt  
       ^K. Ceesay-Seitz, F. Solt, and K. Razavi, ‘µCFI: Formal Verification of
       Microarchitectural Control-flow Integrity’, 2024.

> 原文地址：https://zhuanlan.zhihu.com/p/1927138475397874487

  

