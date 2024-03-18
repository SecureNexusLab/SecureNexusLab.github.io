---
layout: page
title: 内网过waf检测扫描工具
author: 佛系白猫
cover: true
sidebar: []
readmore: true
date: 2024-03-11 17:35:00
typora-root-url: ./..
tags: 
- 公众号推文
- Bypass
categories:
- 公众号推文
---

MASSCAN是TCP端口扫描程序，它异步传输SYN数据包，产生的结果与最著名的端口扫描程序nmap相似。在内部，它更像scanrand、unicornscan和ZMap，使用异步传输。它是一个灵活的实用程序，允许任意地址和端口范围。
正常情况下，扫描器进行扫描的时候，会有个特征，如果目标端口为80，masscan发出去的探测包会带着"masscan"字眼，这里通过反编译，把这种特征去掉，最后达到waf检测不到的效果。

获取工具关注公众号**SecNL安全团队**，回复关键词

```
masscan
```

**注：**

**本工具由本团队佛系白猫提供**