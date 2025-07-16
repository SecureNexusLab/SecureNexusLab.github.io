---
layout: page
title: simpread-OpenVPN小白篇 | 私人知识库
author: 胖大海
cover: true
sidebar: []
readmore: true
date: 2024-11-20
tags: 
- 公众号推文
categories:
- 公众号推文
---

发布时间: 

¶ Docker部署OpenVPN实现访问内网

> 官方制作的Docker镜像支持X86和arm64带有UI操作但是只有英文界面  
> 本教程不提供Docker安装过程

## ¶ 1\. Docker加速

建docker文件夹如果有则跳过。


​    
​    sudo mkdir -p /etc/docker

创建daemon.json文件并写入配置（使用nano或者vim都可以）。  


​    
​    sudo nano /etc/docker/daemon.json

输入以下内容，里面的加速地址可以按照格式修改，网上有很多,修改完成后按Ctrl+O，选择Y保存。  


​    
​    {  
​        "registry-mirrors": [  
​            "https://dockerpull.com",  
​            "https://docker.udayun.com",  
​            "https://dockerproxy.cn",  
​            "https://docker.rainbond.cc",  
​            "https://docker.211678.top"  
​        ]  
​    }

执行命令让配置生效。


​    
​    sudo systemctl daemon-reload  
​    sudo systemctl restart docker

## ¶ 2\. 创建容器并启动


​    
​    #拉取镜像  
​    docker pull openvpn/openvpn-as  
​    #完整端口映射  
​    docker run -d --name=openvpn-as --restart always --cap-add=NET_ADMIN -p 943:943 -p 443:443 -p 1194:1194/udp -v path:/openvpn openvpn/openvpn-as  
​    #其实只需要映射一个就可以443走tcp或者1194走UDP，考虑稳定选择443，自行选择。  
​    docker run -d --name=openvpn-as --restart always --cap-add=NET_ADMIN -p 443:443 -v path:/openvpn openvpn/openvpn-as

参数解释。


​    
​    -d 后台运行  
​    --name=openvpn-as  将容器命名为“openvpn-as”可自行修改  
​    --restart always 让容器跟随docker进程自启动  
​    --cap-add=NET_ADMIN  让容器拥有网络管理权限  
​    -p 将容器的端口映射到宿主机的端口，冒号左边为宿主机端口可根据情况修改，冒号右边为容器内部端口不能更改  
​    -v 映射容器中的目录到宿主机目录，冒号左边为宿主机目录可根据情况修改，冒号右边为容器目录不能更改openvpn/openvpn-as 结束的内容为你要指定哪一个镜像来启动容器

## ¶ 3\. 配置服务端

等待两分钟在日志中找初始密码。


​    
​    docker logs openvpn-as | grep "pass"

![](/images/simpread-OpenVPN小白篇  私人知识库/1752587166002.png)  
访问https://宿主机IP+你将容器内部的443端口映射到宿主机的端口，因为我这里映射的是“443:443”所有访问https://宿主机IP即可，如果你映射的不是宿主机的443请在ip后加上“:你的端口”访问。一定要用https访问。输入用户名为openvpn密码则为我们从日志中获取到的初始密码。  
![](/images/simpread-OpenVPN小白篇  私人知识库/1752587166132.png)


​    
​    登录后修改密码。

![](/images/simpread-OpenVPN小白篇  私人知识库/1752587166215.png)  
![](/images/simpread-OpenVPN小白篇  私人知识库/1752587166386.png)  
在地址后面加admin路径并访问，输入用户名openvpn密码为我们修改之后的密码。  
由于是在docker里面运行我们需要他能访问宿主机网段的设备就需要配置一个路由，登录后找到vpn设置，如下图1标处,按照个人网段书写，不要删除原来的配置还要记得换行。  
通常我们不需要客户端的互联网流量走vpn因为我们只需要能访问局域网设备即可，所以关闭下图中2标的选项（默认是开启的）能有效增加流畅度，根据个人所需修改。  
![](/images/simpread-OpenVPN小白篇  私人知识库/1752587166453.png)  
往下翻点击“Save Settings”按钮保存设置，然后顶部会出现“Update Running Server”按钮，点击更新重启服务生效。  
到这里服务端就已经配置完成了现在我们理一下思路，我们将容器中的443端口映射到了宿主机的443端口，那么需要把宿主机的443端口走tcp协议映射到公网就行。至于用什么方案穿透大家各自发挥我用的是cpolar，因为我只是在外面写写文档没有大量数据传输需求。

## ¶ 4\. 客户端配置

这个服务端已经提供了各个平台的安装包不需要再去找，访问https://宿主机IP，有端口加端口，登录后就能看到下载。  
![](/images/simpread-OpenVPN小白篇  私人知识库/1752587166582.png)  
但是这里下载的软件安装后默认配置还是连接的docker容器的地址，所以我们需要新创建一个配置，以windows为例。  
首先下载一个配置文件，notpad++打开配置文件使用替换功能将原有的IP地址全部替换为公网IP或者域名，同样的方式将所有443端口替换为公网的端口并保存。  
![](/images/simpread-OpenVPN小白篇  私人知识库/1752587166639.png)  
![](/images/simpread-OpenVPN小白篇  私人知识库/1752587166865.png)  
打开客户端新增配置选择文件导入。  
![](/images/simpread-OpenVPN小白篇  私人知识库/1752587166960.png)![](/images/simpread-
OpenVPN小白篇  私人知识库/1752587167173.png)  
输入用户名，勾选保存密码输入服务端的密码。  
![](/images/simpread-OpenVPN小白篇  私人知识库/1752587167229.png)  
点击CONNECT保存，会有一个弹窗点OK就可以了，然后就能看到我们的新配置，点击按钮启用连接就可以了。  
![](/images/simpread-OpenVPN小白篇  私人知识库/1752587167347.png)  
现在你可以尝试ping你的内网设备能ping通就没有问题啦。

  

  

