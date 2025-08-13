---
title: 记一次帮朋友扬眉吐气的逻辑漏洞挖掘
author: SecureNexusLab
date: 2024-09-09 17:16
cover: true
sidebar: []
readmore: true
tags: 
- 公众号推文
- Web安全
- SRC
categories:
- 公众号推文
---

0x00前言

出差刚回酒店，朋友发来消息，我以为是有什么好事情，点开一看，竟然是这么个事情。



当时很累回了一句没时间，出差回家想着不如看一下，就当学习，啪的一下很快啊，地址就发过来点进去一看。



0x01漏洞挖掘



打开站点，抓了个登录包，心凉了半截，全加密又有签名，我这两年半的功力也不够啊。

每次登录加密的密钥都是不一样的。

![](/images/logical-vuln-mining-record/1.png)

JS逆向只会一些简单，点开JS看下吧

![](/images/logical-vuln-mining-record/2.png)

![](/images/logical-vuln-mining-record/3.png)

大致看了一下，大概是这么个意思

- key:`'_dc'`: 定义了一个名为`_dc`的方法。

- value: 这个方法接受两个参数e和a，e是要解密的加密文本，a是可选的密钥，默认为E()函数返回的密钥。

- t = a || E(): 如果没有提供密钥，则使用默认密钥。

- m.default.AES.decrypt(e, t, {mode: m.default.mode.ECB, padding: m.default.pad.Pkcs7}): 使用AES算法解密加密文本e，使用密钥t。

- `m.default.enc.Utf8.stringify(r)`: 将解密后的字节序列转换回UTF-8编码的字符串。

- `.toString()`: 返回最终的解密字符串。

 不会写对应的解密脚本，捣鼓了半天还是放弃了，扫了波目录发现个好东西，

![](/images/logical-vuln-mining-record/4.png)

马上上工具梭哈，什么的没有，估计是换了或者关了AK登录

![](/images/logical-vuln-mining-record/5.png)

![](/images/logical-vuln-mining-record/6.png)

看了一下前台有一个忘记密码的模块，顺便输了一个账号抓包看了一下

![](/images/logical-vuln-mining-record/7.png)

输入存在的账号会回显userid，存在一个枚举，bp跑了一波枚举了大量用户。

![](/images/logical-vuln-mining-record/8.png)

![](/images/logical-vuln-mining-record/9.png)

想着通过修改接受验证码的手机号和修改回显进行绕过重置密码，最后都是失败告终。

![](/images/logical-vuln-mining-record/10.png)

后面手工根据枚举的用户名进行了一波弱口令，成功拿下了一个弱口令但是。

![](/images/logical-vuln-mining-record/11.png)

![](/images/logical-vuln-mining-record/6.png)

有账号比没账号好，可以测试的功能模块更多了，登录进行随便抓个包，JS没逆向出来我这没得搞出来啊

![](/images/logical-vuln-mining-record/12.png)

![](/images/logical-vuln-mining-record/13.png)

我不信全是这样的，全部的模块都点了一遍，翻了一下bp的数据包，发现了一个这样的数据包

![](/images/logical-vuln-mining-record/14.png)

熟悉的师傅应该都知道这是个用户信息啥的数据包，users后面的值就是userid，还是GET请求，这种大部分都是没有鉴权，

马上结合枚举替换成管理员的userid，成功获取管理员的信息

![](/images/logical-vuln-mining-record/15.png)

修改性别试一下，竟然成功了

![](/images/logical-vuln-mining-record/16.png)

前面我们看到忘记密码那一块可以进行验证码校验修改验证码，为自己手机号（因为为测试环境且授权了不会影响业务，师傅们在挖洞时候要记得修改的操作要谨慎）

![](/images/logical-vuln-mining-record/17.png)

![](/images/logical-vuln-mining-record/18.png)

![](/images/logical-vuln-mining-record/19.png)

![](/images/logical-vuln-mining-record/20.png)

最后也是成功修改密码登录上了管理员账户，也是让我朋友扬眉吐气了注：作者：IntAx

![](/images/logical-vuln-mining-record/21.png)
