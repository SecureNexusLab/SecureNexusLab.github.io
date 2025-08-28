---
title: kali Linux配置那啥上网（懂得都懂）
date: 2024-04-22 16:46
author: 【白】
tags: 公众号推文
categories: 公众号推文
cover: true
sidebar: []
readmore: true 
---
# 前言

![](/images/kalilinux-configure-vpn/1.png)



首先来看看废话，本次实验是在kali中安装，当然我们在Windows也可以使用v2



# 安装

添加公钥

```
wget -qO - https://apt.v2raya.org/key/public-key.asc | sudo tee /etc/apt/keyrings/v2raya.asc
```

第二步：
```
echo "deb [signed-by=/etc/apt/keyrings/v2raya.asc] https://apt.v2raya.org/ v2raya main" | sudo tee /etc/apt/sources.list.d/v2raya.list
```

第三步：
```
sudo apt update
```

第四步：
```
sudo apt install v2raya v2ray
```

![](/images/kalilinux-configure-vpn/2.png)

到这里就完成了。

以上四步大家按照顺序依次执行命令。

# 使用

v2属于一个网页端，我们首先需要启动
```
sudo systemctl start v2raya.service
```
启动之后我们可以直接使用网页端，访问：
```
127.0.0.1:2017
```

我们访问之后，会有一个需要创建管理员

# 创建账号

![](/images/kalilinux-configure-vpn/3.png)

这里我们自己去创建一个就可以了，然后回提示我们导入，这里大家就自行购买

![](/images/kalilinux-configure-vpn/4.png)

我们可以购买订阅，也可以单独导入一个jd，我这里是购买了订阅地址，
```
我们可以购买订阅，也可以单独导入一个jd，我这里是购买了订阅地址，
```
![](/images/kalilinux-configure-vpn/5.png)

这里可以更新订阅，

![](/images/kalilinux-configure-vpn/6.png)

这里可以选择我们一个，然后到右上角就绪点击一下

![](/images/kalilinux-configure-vpn/7.png)

这里会变成正在运行，然后我们就可以快乐的看了。

![](/images/kalilinux-configure-vpn/8.png)

最后经常用的记得开一下开机自启

```
systemctl enable v2raya.service
```