---
title: CSRF攻击修改用户微信绑定（SRC思路）
date: 2024-07-18 14:29
author: SecureNexusLab
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

在挖洞过程中，web端或app中可能会碰到通过其他方式登录的功能，这些都是需要用户确认授权后，才会登录外部平台的账号，且有外部平台返回的数据。OAuth2.0就是解决客户端和第三方平台互不信任的一种授权协议。

![](/images/csrf-wechat-src/1.png)

而OAuth2授权模式主要是授权码模式，授权码授权流程如下

![](/images/csrf-wechat-src/2.png)

第三方平台会在收到认证请求后会向客户端发送code授权码，客户端再利用code向第三方平台获取授权凭证

但是如果客户端在交换授权凭证时，code没有和客户端的账号做绑定，就可以造成csrf攻击，攻击者可以利用构造好的url让用户绑定自己的第三方平台账号，导致账号接管

在下面的自己的账号个人中心页面发现存在微信绑定功能

![](/images/csrf-wechat-src/3.png)

然后点击绑定并开始burp抓包

![](/images/csrf-wechat-src/4.png)

向微信请求code，通过返回包可以看到code值

![](/images/csrf-wechat-src/5.png)

然后再看客户端通过code向微信换取授权凭证的请求包

![](/images/csrf-wechat-src/6.png)

这里显然是没有token防护，也没有referer检测

那么让受害者访问上面对应的链接，就可以让我的微信绑定受害者的账号

下面是有验证的情况

![](/images/csrf-wechat-src/7.png)

复制url：

```
www.test.com/portalcenter/open/addOpenUserWx/?type=wx&code=091d3Hkl2uCiMd4dSGll21vBuE4d3Hkr&state=test

```

将该url发给受害者，访问后即可触发微信绑定，且绑定的是攻击者的微信

查看受害者的个人中心页面，微信绑定成功

![](/images/csrf-wechat-src/8.png)

返回登录页面，尝试微信登录
