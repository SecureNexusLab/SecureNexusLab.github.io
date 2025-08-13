---
title: 从flash提取固件及oob数据去除
date: 2024-09-04 12:26
cover: true
sidebar: []
readmore: true
tags: 
- 公众号推文
- 二进制安全
categories:
- 公众号推文
---



**前言**



​    本文介绍了拆解设备flash芯片的过程，并使用编程器来读取Flash芯片内容，然后对提取的固件进行分析，寻找到OOB数据的位置，去除固件中存在的oob数据，最终从文件系统中提取出文件信息。



**固件背景**

​	本次分析的设备是某款摄像头，由于该摄像头提供的固件都经过加密处理，且不同型号所使用的加密密钥也不同，所以本次通过设备的Flash芯片提取固件。本文主要介绍了如何拆解设备的flash芯片，然后通过编程器来读取Flash芯片内容，最后对提取的固件进行分析，通过芯片对应的手册信息，去除固件中存在的oob数据，最终正常的获取文件系统的文件信息。

![](/images/iot-flash-firmware-data-remove/1.png)

**固件提取**

​	拆解芯片所使用的硬件工具：热风焊台，恒温内热烙铁（尖头，刀头），吸锡带，吸锡器，助焊剂，松香，镊子，锡丝，隔热胶带等，在拆解芯片时记得提前在芯片引脚涂上助焊剂。

将热风枪的温度设置在350摄氏度左右，距离芯片1-2厘米，持续加热直到芯片引脚锡点融化，使用镊子轻轻触碰，能够感觉到明显的松动，就可以取下芯片。

将烙铁加热到400摄氏度左右，融化一截锡丝至芯片的引脚处，记得提前涂上助焊剂，然后将融化的锡球来回在芯片引脚拖动，来回拖动几次就能够轻松使用镊子取下芯片。

W29N0DN60012芯片拥有48引脚，相比于常见的flash芯片(8pin或16pin)在拆解难度上更大。

![](/images/iot-flash-firmware-data-remove/2.jpg)

​	将拆好的芯片表面的焊油擦拭干净，仔细观察芯片的引脚有没有被锡点粘连在一起的现象，使用锡球拖动时温度控制不好，这种芯片的引脚很容易发生粘连，如果发现粘连，在引脚涂上助焊剂使用烙铁仔细清理即可。

将拆解好的芯片放入48脚的底座中，然后使用RT809H编程卡住底座。使用的编程器依然是Ifix爱修的RT809H，当然也有其他可选的编程器，例如T48。

![](/images/iot-flash-firmware-data-remove/3.png)

然后将编程器和电脑正确连接，然后打开编程器软件。编程器软件下载链接国内版 - iFix爱修网，点击编程器软件中的智能识别，然后根据芯片上的信息进行校对，点击读取即可。

![](/images/iot-flash-firmware-data-remove/4.png)

使用RT809H编程器读取芯片中的内容，保存至本地的文件中。

![](/images/iot-flash-firmware-data-remove/5.png)

将提取到的固件直接使用binwalk 工具进行分析，提示偏移量0x1C5COOO处为UBI文件系统相关的内容。

![](/images/iot-flash-firmware-data-remove/6.png)

​	Binwalk会将这部分内容提取出来，并使用偏移量来命令。通过file命令查看到文件为ubi镜像，直接使用ubireader_extract_images命令从UBI 文件系统镜像中提取数据，但是系统提示错误信息：Less than 2 layout blocks found说明镜像并不完整

![](/images/iot-flash-firmware-data-remove/7.png)

​	直接将芯片中提取到的固件进行分析，并没能获得任何有价值的信息。

​	经过查阅资料可以知道，从NAND FLASH芯片中提取的文件中包含OOB数据，OOB (Out-Of-Band) 区域是一种嵌入式存储器系统中的特殊区域，通常用于存储与数据关联的元数据或控制信息，如 ECC (Error Correction Code) 纠错码、坏块标记、页编号等，而不是用于存储实际数据。

通过对应芯片手册也能看见相关信息，每页拥有2112字节，其中2048字节存储数据，64字节作为备用空间。W29N01HV芯片手册链接地址https://pdf1.alldatasheetcn.com/datasheetpdf/view/1353893/WINBOND/W29N01HV.html

![](/images/iot-flash-firmware-data-remove/8.png)

通过芯片的数据存储结构也能清楚看见

![](/images/iot-flash-firmware-data-remove/9.png)

​	现在知道了数据中包含OOB数据，所以需要恢复文件就需要将OOB数据清除，如何在文件中查找到对应的OOB数据很关键。

这里以binwalk处理后的文件1C5C000.ubi文件为例，使用文本编辑器010Editor打开文件。

![](/images/iot-flash-firmware-data-remove/10.png)

​	可以看见文件存在UBI等字符，现在需要找到文件中OOB数据的位置，前面通过芯片手册知道每一页大小2112字节（0x840）,数据区域2048字节（0x800）和备份区域64字节（0x40）。

​	所以在010Editor编辑器中挑取合适的一页进行分析，应该能识别到OOB数据。因为这个文件比较特殊，首部存在大量的00，所以就选择第一页作为分析的起点。可以查看到在第一页的中部0x410处存在部分非0数据。

![](/images/iot-flash-firmware-data-remove/11.png)

​	然后就是第一页的页尾，存在大量非0数据。

![](/images/iot-flash-firmware-data-remove/12.png)

​	通常会将数据区域和备份区域前后排列，例如一页的前0x800字节作为数据区域，0x40字节作为备份区域，那么只需要去除每一页的后0x40字节的数据就可以了，但事实并非如此，通过去除后0x40字节的数据，数据并没有正常的恢复。所以OOB数据并非简单的排列。

​	通过多个页进行比对，发现在每一页的中部都会存在14字节的数据，在页的尾部都存在50字节的数据，刚好凑成64字节。

![](/images/iot-flash-firmware-data-remove/13.png)

![](/images/iot-flash-firmware-data-remove/14.png)

事实也确实是这样，因为ubi文件系统的原因，存储到OOB数据进行了移动翻转，并未连在一起形成一个完整的块。

既然已经知道了OOB数据存在的位置，那么可以写一个脚本来清除掉OOB数据。

```
#!/usr/bin/python3

# 打开名为 "W29N01H.BIN" 的二进制文件以读取数据
data = open("W29N01H.BIN", "rb").read()

# 创建一个名为 "W29N01H_fix.bin" 的二进制文件以写入数据
f = open("W29N01H_fix.bin", "wb")

# 设置页面大小（page size）为 0x840（十六进制），即 2112 字节
p = 0x840

# 遍历输入数据并重新排列数据，以修复数据
for i in range(len(data)//p):
    # 将每页的数据重新排列并写入输出文件
    f.write(data[i*p:i*p+0x410] + data[i*p+0x41E:i*p+0x800] + data[i*p+0x802:i*p+0x810])

# 关闭输出文件
f.close()
```

将提取的固件修复之后，再使用ubireader_extract_images命令提取数据。可以看到提取的文件中有两组文件大小相同，命名也相似。

![](/images/iot-flash-firmware-data-remove/15.png)

这里对提取到的ubifs文件系统的命名进行解释，

- l "img-527035962"：这部分可能是该镜像文件的标识符或版本号。
- l "vol-app"：这部分表示 UBI 文件系统中的一个卷（Volume）的名称，通常包含应用程序、数据或其他文件。
- l "pri"：这部分可能表示卷的优先级或类型。通常，UBI 卷可以分为主卷（primary）和备份卷（secondary）。主卷包含实际文件系统数据，而备份卷用于冗余备份。这里的 "pri" 可能表示主卷。
- l ".ubifs"：这是文件扩展名，指示这是一个 UBIFS（UBI File System）文件系统的镜像文件。

所以提取到的文件主要有两部分，app_pri和cfg_pri。还有的文件是这两个文件的备份文件(文件内容和主卷不完全相同)，和一个空文件。

然后对ubifs文件，可以使用ubireader_extract_files命令从 UBI文件系统中提取文件。

![](/images/iot-flash-firmware-data-remove/16.png)

现在能够从ubifs文件系统提取出文件信息，说明我们成功的从Flash芯片中获取到了我们想要的内容，然后就可以继续分析固件中的其他文件。



**总结**

本次分析主要是从设备falsh芯片提取固件，然后去除OOB数据，其中重要的一部分内容是如何寻找到OOB数据的布局，不同的芯片中OOB数据分布是不一样的，有的是直接拼接在有效数据后面，还有的就像本次分析的目标一样，分散到有效数据的内部。当然能够顺利找到OOB数据也需要一定的技巧和经验，例如找到全为0xFF或者0x00的数据段，然后对比观察。另外这种大容量的NAND Flash芯片，ubifs格式的文件系统更适合，所以还可以定位一些该文件系统的关键字例如UBI#和UBI!等信息。

**参考链接**

W29N01HV芯片手册链接地址：

https://pdf1.alldatasheetcn.com/datasheet-pdf/view/1353893/WINBOND/W29N01HV.html

RT809编程器软件下载地址：

http://doc.ifix.net.cn/@rt809/CHN.html
