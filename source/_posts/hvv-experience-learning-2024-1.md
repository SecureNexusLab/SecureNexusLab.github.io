---
layout: page
title: 2024护网经验学习
author: SecureNexusLab
date: 2024-05-04 20:19
cover: true
sidebar: []
readmore: true
tags: 
- 公众号推文
- Web安全
- 护网
categories:
- 公众号推文
---

### **01 暴力破解：基本特征就是短时间内大量的登录请求**

包括`SMB邮件` `ftp` `sslvpn` `rdp` `ssh` `telnet` `mysql`协议等

![](/images/hvv-experience-learning-2024-1/1.png)

误报排除方法：查看时间，是否为连续时间有大量的访问日志或者暴力破解安全日志，并确认是否每次暴力破解安全日志之间的间隔相似

![](/images/hvv-experience-learning-2024-1/2.png)

短时间内发起了大量的请求，一般不会是误报，需要去相应终端尽行排查

![](/images/hvv-experience-learning-2024-1/3.png)

到终端上查看登录日志，是否存在大量的登录失败日志 

Windows: `rdp/smb`都可以在事件查看器中找4625的登录信息事件 

Linux: 除root之外，是否还有其它特权用户(uid 为0) `awk -F: ‘$3==0{print $1}’ /etc/passwd`

可以远程登录的帐号信息 `awk ‘/$1|$6/{print $1}’ /etc/shadow`

统计登录失败日志 `grep -o “Failed password” /var/log/secure|uniq -c`


### **02 Web路径探测：疑IP或者内网主句在较短时间内请求了目的系统较多的web目录**

![](/images/hvv-experience-learning-2024-1/4.png)

误报排除方法：如果短时间内扫描大量不存在的web页面（人工达不到的速度，比如上面一分多钟达到208次）那就很有可能是在探 测web目录是否存在一些中间件

查看扫描的部分路径是否一些常见中间件的url，比如上面一直扫描web是否存在mysql的管理工具，如果扫描到 mysql管理工具可能利用工具存在漏洞进行渗透，那就不是误报，或者扫描一些webshell路径是否存在的时候会扫描 一些奇奇怪怪的不存在的页面，比如`xx.php/db.php/shell.php`等。扫描路径一般带有一些规律的，

可以分为以下几类：数据库管理页面（phpmyadmin，adminer等）、后台管理页面（manage.php , admin.php 等）、文本编辑器特定路径（ewebeditor,ueditor）、部分webshell地址(shell.php)

如果路径里面都是一些普通的html页面、客户自己的业务页面则为误报，比如扫描很多index.php index.html default.php等,这种场景一般是客户自己一些正常业务发出的大量请求

### **03 Webshell上传：检测webshell上传，在提交的请求中有webshell特征的文件或者一句话木马字段。可以网上检索到webshell**

![](/images/hvv-experience-learning-2024-1/5.png)

误报排除方法：若查看到日志，为乱码（实际为二进制无法读取）可判断为误报若SIP提示的链接下载的webshell文件为正常文件也可判断为误报

![](/images/hvv-experience-learning-2024-1/6.png)

如果自己拿不准的话，将样本的md5或者样本上传到virustotal或者微步云沙箱进行检查webshell一般不会重复上传，查看到多条日志，提交的数据包都是相同的webshell文件，可以判断是否为客户的业务行为，为误报。如下图，所有的日志都是相同的，判断为客户业务行为

![](/images/hvv-experience-learning-2024-1/7.png)

若为复杂的代码格式, 含有eval等危险函数, 各种编码转换和加密字符串的，基本为正报

![](/images/hvv-experience-learning-2024-1/8.png)

对于编码的webshell，需要进行解码查看,Accept-Charset字段值进行base64解码，解码结果包含syetem()、echo()、certutil.exe等敏感关键字，且不涉及业务内容，判断为真实漏洞探测行为

### **04 Sql注入攻击：Sql注入通过判断web请求提交的字段中是否有数据库查询语句或者关键词进行判断**

![](/images/hvv-experience-learning-2024-1/9.png)

误报排除方法：查看日志，是否存在数据库查询的关键字，如select，where，union，等，并判断提交的语句是否为客户业务请求不规范所导致

![](/images/hvv-experience-learning-2024-1/10.png)

通过返回包判断是否攻击成功，若判断为攻击行为，但是无返回包，判断为攻击不成功

![](/images/hvv-experience-learning-2024-1/11.png)

### **05 XSS攻击：看日志中是否存在或类似的变形的字符串，这类是属于探测类攻击**

看日志中是否存在外部url地址，这些地址可能是用于接收管理员信息的第三方平台

误报排除方法：如果出现大量xss攻击日志，但日志内容格式都是一样的，没有明显变化，则可能是误判

如果攻击日志为json类数据，需和客户确认是否是正常业务，如果是正常业务，则为误判