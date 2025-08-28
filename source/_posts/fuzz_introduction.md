---
layout: page
title: Fuzzing模糊测试——带你初入模糊测试时代
author: Kidder1
cover: true
sidebar: []
readmore: true
date: 2024-03-06 09:18:00
typora-root-url: ./..
tags: 
- 公众号推文
- FUZZ
categories:
- 公众号推文
---

模糊测试，又称为模糊化测试或模糊化，是一种测试软件的方法。其核心思想是将自动生成或半自动生成的随机数据注入到程序中，然后监视程序的异常行为，如崩溃或断言失败，以便发现潜在的程序错误，比如内存泄漏。通常，模糊测试被广泛应用于揭示软件或计算机系统的安全漏洞。

本篇文章会给大家讲解一下基本的模糊测试的原理和介绍。

# 测试分类

模糊测试工具主要分为两大类，即变异测试和生成测试。该方法可以作为白盒、灰盒或黑盒测试的一种手段。尽管文件格式和网络协议是最常见的测试目标，但任何程序输入都可能成为测试对象，包括环境变量、鼠标和键盘事件以及API调用序列。甚至一些通常被忽视的对象，如数据库中的数据或共享内存，也可以用于测试。

有人可能质疑，这不就是随机数自动化测试吗？为何称其为模糊测试？在随机测试中，著名的例子是Android的Monkey Test。其思想是模拟猴子毫无逻辑的操作，以测试手机软件的健壮性。然而，Monkey Test主要关注测试用户界面上的随机操作，而对于一些深层次、底层的问题，比如通讯协议的缺陷，却很难发现。因此，模糊测试更强调通过注入各种随机数据来全面测试程序的鲁棒性和安全性。

目前二进制漏洞挖掘流行的技术就是污点分析、符号执行、模糊测试，三者的结合和近年来AI技术的发展，深度学习的入局对模糊测试产生了深远的影响。

对于这些内容，静态分析还是要系统的学一遍，这里是参考知乎大佬的回答所给的

目前这个课程国内只有北京大学和南京大学有完整的教案。

北京大学：【北京大学公开课】软件分析技术

南京大学软件分析课程录播：南京大学《软件分析》

国外的课件：Static Program Analysis

# 测试对象

- 白盒源代码
- 黑盒仅二进制文件
- 环境变量和参数
- Web应用程序：wfuzz
- 网络协议
- Web浏览器和内存数据
- 物联网固件
- API应用程序的FUZZ

# 黑盒测试模式

 黑盒测试是一种功能导向的测试方法，测试人员不需要知道应用程序的内部工作原理，而是关注于确保软件系统按照规格说明的要求执行。测试人员将输入提供给系统，观察系统的输出，然后比较输出与预期结果，从而评估软件的正确性和功能性。

# 实战举例

简单理解 libfuzzer 就是，如果我们要 fuzz 一个程序，找到一个入口函数，然后利用

```
extern "C" int LLVMFuzzerTestOneInput(const uint8_t *data, size_t size) {    .......    .......}
```

接口（hardness），我们可以拿到 libfuzzer 生成的 测试数据以及测试数据的长度，我们的任务就是把这些生成的测试数据传入到目标程序中让程序来处理测试数据， 同时要尽可能的触发更多的代码逻辑。

这个是官方文档给的定义

![img](/images/fuzz_introduction/640.png)

我们来查看官方的例子

```c++
//fuzz_me.cc
#include <stdint.h>
#include <stddef.h>

bool FuzzMe(const uint8_t *Data, size_t DataSize) {
  return DataSize >= 3 &&
      Data[0] == 'F' &&
      Data[1] == 'U' &&
      Data[2] == 'Z' &&
      Data[3] == 'Z';  // :‑<
}

extern "C" int LLVMFuzzerTestOneInput(const uint8_t *Data, size_t Size) {
  FuzzMe(Data, Size);
  return 0;
}
```

要为此目标构建模糊器二进制文件，使用最新的 Clang 编译器编译源代码带有以下的参数：

-fsanitize=fuzzer（必需）：向 libFuzzer 提供进程内覆盖率信息以及与 libFuzzer 运行时的链接

-fsanitize=address（推荐）：启用 AddressSanitizer

-g（推荐）：启用调试信息，使错误消息更易于阅读

编译选项，并且运行

```
clang++ -g -fsanitize=address,fuzzer fuzzing/tutorial/libFuzzer/fuzz_me.cc
```

说明

可以看出这个模糊器从INFO: Seed: 851904448随机种子开始

默认情况下，libFuzzer 假定所有输入都为 4096 字节或更小

![图片](/images/fuzz_introduction/640-171068608316613.png)

这里进行了报错，在这个代码中的第九行第七列发生的

![图片](/images/fuzz_introduction/640-171068608316614.png)

![图片](/images/fuzz_introduction/640-171068608316715.png)

这里保存了crash的文件

![图片](/images/fuzz_introduction/640-171068608316716.png)

查看crash文件

![图片](/images/fuzz_introduction/640-171068608316717.png)

可以得出，在函数输入为三个字符的时候，数组下标3会发生越界访问