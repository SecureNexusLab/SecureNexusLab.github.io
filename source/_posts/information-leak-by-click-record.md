---
layout: page
title: 记一次容易被忽视的功能点造成的大量信息泄露
author: IntAX
cover: true
sidebar: []
readmore: true
date: 2025-06-08
tags: 
- 公众号推文
- Web安全
categories:
- 公众号推文
---



**总体情况**  

  

某拼团小程序存在系统信息泄露漏洞

以下为漏洞详情

**01**  

  

![1.png](/images/记一次容易被忽视的功能点造成的大量信息泄露/1752587233170.png)  
**02**  

**点击拼团详情，抓包**![2.png](/images/记一次容易被忽视的功能点造成的大量信息泄露/1752587233427.png)

数据包：

*   *   *   *   *   *   *   *   *   *   *   *   *   *   *

```
GET /zinfo/tail?initiateId=4&target=2 HTTP/1.1Host: xxxxxAccept: application/json, text/plain, */*Xweb_xhr: 1Authorization: NTc3ODU0MzRFODVGNDRENkY0NzE0MDMwREU1NTc3NERDMkYzNUU0NDNCN0E1NDRENDQ3NzZBQjgxMTA4OUEyNA==User-Agent: Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/126.0.0.0 Safari/537.36 MicroMessenger/7.0.20.1781(0x6700143B) NetType/WIFI MiniProgramEnv/Windows WindowsWechat/WMPF WindowsWechat(0x63090c33)XWEB/13487Content-Type: application/x-www-form-urlencodedSec-Fetch-Site: cross-siteSec-Fetch-Mode: corsSec-Fetch-Dest: emptyReferer: https://servicewechat.com/wx90229e983b201cf1/109/page-frame.htmlAccept-Encoding: gzip, deflate, brAccept-Language: zh-CN,zh;q=0.9Priority: u=1, iConnection: keep-alive
```

  


从手机号尾数可得泄露号码为第一个拼团用户号码

![3.png](/images/记一次容易被忽视的功能点造成的大量信息泄露/1752587233491.png)

  

![4.png](/images/记一次容易被忽视的功能点造成的大量信息泄露/1752587233600.png)  
**03**  


**进行initiateId遍历可获得大量拼团用户手机号码**![5.png](/images/记一次容易被忽视的功能点造成的大量信息泄露/1752587233704.png)

9938个电话号码

![6.png](/images/记一次容易被忽视的功能点造成的大量信息泄露/1752587233838.png)

由于用户均为参加拼团用户，且为未拼团成功的用户可利用这一点进行定点诈骗等添加个人信息微信。

![7.png](/images/记一次容易被忽视的功能点造成的大量信息泄露/1752587234000.png)  

  

最后，欢迎来自不同背景的师傅加入我们，一起打造一个活跃的交流社群，在SNL社群没有背景、能力的差异，我们欢迎大家一起学习进步。

我们是一支多元化的安全技术团队，成员活跃在渗透测试、漏洞挖掘、逆向工程等多个安全领域。日常社群氛围充满技术热情：大家会一起分析最新漏洞案例，组织内部渗透测试实战，协作开发安全研究项目。

团队定期开展技术分享会，从Web安全到二进制漏洞，从CTF解题技巧到企业级安全防护，各种技术话题都能在这里碰撞出火花。如果你也热衷于安全技术的实践与交流，欢迎加入我们一起成长！  

  

[SecNL团队常态化招新提示](https://mp.weixin.qq.com/s?__biz=MzU2MDE2MjU1Mw==&mid=2247485938&idx=1&sn=d28961da6f0319864bb5dfcc4a867da3&scene=21#wechat_redirect)

  

