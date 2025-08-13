---
layout: page
title: PolarCTF[RE][easy]全解(上)
author: 白玉京
cover: true
sidebar: []
readmore: true
date: 2025-03-03
tags: 
- 公众号推文
- CTF
categories:
- 公众号推文
---


**引言**

  

本次分享PolarCTF靶场中所有[easy]re题目的WP。

由于文章篇幅较长（一共有22道题目），分为两期发表,本期先分享上半部分内容，适合刚接触逆向的朋友。如果你也对逆向感兴趣，或者正在准备CTF比赛，希望这些内容能给大家一些帮助。

  

  

**PolarCTF**[ezpack]  

  

  

![image.png](/images/PolarCTF-RE-easy-1/1752587198831.png)


发现是Aspack壳，用工具脱壳：

![image.png](/images/PolarCTF-RE-easy-1/1752587199122.png)

32位：用IDA打开。直接shift+f12发现关键字符“Enter password:”  

![image.png](/images/PolarCTF-RE-easy-1/1752587199208.png)

直接跟进函数看看：  

![image.png](/images/PolarCTF-RE-easy-1/1752587199633.png)

  * 关键函数：`sub_401738`。继续跟进去看看：

    * 大概逻辑就是：`input`逐个和`0xC`进行异或，然后和`Str2`进行比较

    * 所以`input = Str2 ^ 0xC`

  

  * ![image.png](/images/PolarCTF-RE-easy-1/1752587199686.png)

查看一下`Str2 = >4i44oo4?i=n>:m;8m4=oo4i;>?4>h9m``  
`  
![](/images/PolarCTF-RE-easy-1/1752587199797.png)

  

![](/images/PolarCTF-RE-easy-1/1752587199881.png)

  

编写解密脚本：

```C
#include <stdio.h>

int main(){
    char str[] = ">4i44oo4?i=n>:m;8m4=oo4i;>?4>h9m";
    for(int i = 0; i < 32; i++) {
        str[i] ^= 0xC;
    }
    printf("%s", str);	
    return 0;
}// 28e88cc83e1b26a74a81cc8e72382d5a
```


**PolarCTF**[L00k_at_h3r3]  


![image.png](/images/PolarCTF-RE-easy-1/1752587199945.png)


查壳发现是NsPack，用工具脱壳：  

![image.png](/images/PolarCTF-RE-easy-1/1752587200147.png)

32位，用IDA打开。直接搜索函数，找到`main`函数：

![image.png](/images/PolarCTF-RE-easy-1/1752587200236.png)

很长，但实际上很简单，步骤大概是：

  * 和几个数组逐个进行比较，且异或不同的数

  * 例如第一个`for`循环，`input[i]`和 `aNqt[i] ^ 0xBu`的结果进行比较

  * 第二个`for`循环，`input[i+len(aNqt)]`和 `aKixs[i] ^ 0xCu`的结果进行比较

  * ...

  

![image.png](/images/PolarCTF-RE-easy-1/1752587200297.png)

  * aNqt = nqT

  * aKixs = kixS

  * aKa9jr = ka9jR

  * aHCq = h|>cQ

  * aG = g<}<

编写解密脚本：

```C
#include<stdio.h>  
int main(){
    char str1[] = "nqT";	
    char str2[] = "kixS";
    char str3[] = "ka9jR";
    char str4[] = "h|>cQ";
    char str5[] = "g<}<";  
    for(int i = 0; i < sizeof(str1)-1; i++) {
        str1[i] ^= 0xBu;	
    }	
    printf("%s", str1);	
    for(int i = 0; 
        i < sizeof(str2) - 1; i++){
            str2[i] ^= 0xCu;	
        }	
        printf("%s", str2);	
        for(int i = 0; i < sizeof(str3) - 1; i++) {		str3[i] ^= 0xDu;	
        }	
        printf("%s", str3);	for (int i = 0; i < sizeof(str4) - 1; i++){
            str4[i] ^= 0xEu;	
        }	
        printf("%s", str4);	for(int i = 0; i < sizeof(str5) - 1; i++){
            str5[i] ^= 0xFu;	
        }
        printf("%s", str5);  
        return 0;
    }//ez_get_fl4g_fr0m_h3r3// 需要md5 32位小写加密后再提交

```
**PolarCTF**[shell]  


查壳发现是upx

![image.png](/images/PolarCTF-RE-easy-1/1752587200401.png)

工具脱壳：`upx -d [FilePath]`  


![image.png](/images/PolarCTF-RE-easy-1/1752587200494.png)

![image.png](/images/PolarCTF-RE-easy-1/1752587200595.png)

32位，IDA打开，`shitf+F12`发现关键字符串。跟进去看看  

![image.png](/images/PolarCTF-RE-easy-1/1752587200776.png)

F5反编译直接看到！  

![image.png](/images/PolarCTF-RE-easy-1/1752587200877.png)

**PolarCTF**[PE结构]  


![image.png](/images/PolarCTF-RE-easy-1/1752587201171.png)

不是PE文件？根据题目，大概可以猜到，是想让我们修复PE结构。winhex打开看看  

![image.png](/images/PolarCTF-RE-easy-1/1752587201375.png)

上来就发现不是MZ！notepad++修改一下（别问为什么不是winhex，notepad用起来顺手一点QAQ）

![image.png](/images/PolarCTF-RE-easy-1/1752587201682.png)  

再打开，发现OK了。  

![image.png](/images/PolarCTF-RE-easy-1/1752587201989.png)

直接运行试试：  

![image.png](/images/PolarCTF-RE-easy-1/1752587202073.png)

**PolarCTF**[拼接]  


![image.png](/images/PolarCTF-RE-easy-1/1752587202147.png)

  

32位，直接用IDA打开，发现脸上就是main函数，手直接就挪F5上了啊！  

  

![image.png](/images/PolarCTF-RE-easy-1/1752587202222.png)

嗯······这拼接在，嗯。哈哈。

‍

**PolarCTF**[加加减减]  


![image.png](/images/PolarCTF-RE-easy-1/1752587202285.png)

32位，IDA打开。main函数又在脸上，直接F5：  

![image.png](/images/PolarCTF-RE-easy-1/1752587202373.png)

看眼逻辑：`input`每位都`--`，然后与`str2`进行比较。

所以`str2`每位都`++`，就是`flag`

```C
int main() {
    char str2[] = "ek`fz5123086/ce7ac7/`4a81`6/87b`b28a5|";
    
    for (int i = 0; i < strlen(str2); i++) {
        str2[i]++;
    }
    
    printf("%s", str2);
    return 0;
}
```

![image.png](/images/PolarCTF-RE-easy-1/1752587202469.png)

  

**PolarCTF**[HowTo\\_LogIn]  

  

  

![image.png](/images/PolarCTF-RE-easy-1/1752587202527.png)

upx壳，工具脱掉：  

![image.png](/images/PolarCTF-RE-easy-1/1752587202630.png)

32位，因为是注册机，所以先运行看看：

![image.png](/images/PolarCTF-RE-easy-1/1752587202733.png)  
![image.png](/images/PolarCTF-RE-easy-1/1752587202842.png)


OK，上IDA，搜索字符串看看：  

![image.png](/images/PolarCTF-RE-easy-1/1752587203013.png)

  

跟进去发现关键：

  


![image.png](/images/PolarCTF-RE-easy-1/1752587203073.png)

pawd会被打印，所以可以x64嗯调到打印的地方？ 哦不行，试了一下发现，只会打印输入的东西，还是得自己对照QAQ

复制下来试试：

```C
#include <stdio.h>
#include <string.h>

int main() {
    char v11[20];
    
    v11[0]  = 'C';
    v11[1]  = 'Z';
    v11[2]  = '9';
    v11[3]  = 'd';
    v11[4]  = 'm';
    v11[5]  = 'q';
    v11[6]  = '4';
    v11[7]  = 'c';
    v11[8]  = '8';
    v11[9]  = 'g';
    v11[10] = '9';
    v11[11] = 'G';
    v11[12] = '7';
    v11[13] = 'b';
    v11[14] = 'A';
    v11[15] = 'X';
    v11[16] = '\0';
    
    if (strlen(v11) == 0x10) {
        printf("%s", v11);
    }
    
    return 0;
}
```
![image.png](/images/PolarCTF-RE-easy-1/1752587203207.png)


运行检验一下：  

![image.png](/images/PolarCTF-RE-easy-1/1752587203294.png)

哦对，邮箱要有`@`，且`@`后面有字符，有`.`，且`.`后面要有字符。反正写规范一点，直接123会被拦截

tip：最终密码要进行32位md5加密哦

flag{c3ec13a01b07ad218dbd5f4bbab592b9}

**PolarCTF**[box]  

![image.png](/images/PolarCTF-RE-easy-1/1752587203381.png)

64位ELF。进kali看看：

![image.png](/images/PolarCTF-RE-easy-1/1752587203485.png)

程序入口是key2。IDA，启动！

![image.png](/images/PolarCTF-RE-easy-1/1752587203581.png)

跟进去发现，`key2=str1`，str1 = that_ok = key2

![image.png](/images/PolarCTF-RE-easy-1/1752587203663.png)

![image.png](/images/PolarCTF-RE-easy-1/1752587203795.png)

接着找谁调用了`key2()`，找到`main`函数  

![image.png](/images/PolarCTF-RE-easy-1/1752587203890.png)

  

![image.png](/images/PolarCTF-RE-easy-1/1752587204345.png)

  

![image.png](/images/PolarCTF-RE-easy-1/1752587204546.png)

直接复制下来，运行一下发现：`key1 = 11694441`，`key3 = NNSXS===`  

![image.png](/images/PolarCTF-RE-easy-1/1752587204715.png)

拼起来：`flag{11694441that_okNNSXS===}`

啊？不对。

草，key3需要base32解密，结果为：key 所以：`flag{11694441that_okkey}`，再进行md5，32位小写加密即可

‍

**PolarCTF**[crc]  


![image.png](/images/PolarCTF-RE-easy-1/1752587204816.png)

64位ELF，进kali看看：发现没有任何输出，直接等你输入。

好吧，那IDA启动！  

![image.png](/images/PolarCTF-RE-easy-1/1752587204913.png)

简单分析一下`strmncpy`函数：  

![image.png](/images/PolarCTF-RE-easy-1/1752587205043.png)

简单分析一下`magic`函数：  

![image.png](/images/PolarCTF-RE-easy-1/1752587205212.png)

查找发现，python的binascii库和zlib库有crc32

  

因为flag的前4位包是flag，所以可以试试：`flag`加密后是否和`0xd1f4eb9a`相等，看看python库里的crc32的实现是否和这题的实现一致。  

![image.png](/images/PolarCTF-RE-easy-1/1752587205435.png)

OK！可以开始爆破了

```python
import binascii

def find_matches():
    # Single-character checks
    for i in range(128):
        s = chr(i)
        crc = binascii.crc32(s.encode())
        if crc == 0x15d54739:
            print(f"str2 = {s}")
        if crc == 0xfcb6e20c:
            print(f"str6 = {s}")

    # Two-character checks
    for i in range(128):
        for j in range(128):
            s = chr(i) + chr(j)
            crc = binascii.crc32(s.encode())
            if crc == 0x3fcbd242:
                print(f"str4 = {s}")

    # Four-character checks
    for i in range(128):
        for j in range(128):
            for k in range(128):
                for l in range(128):
                    s = chr(i) + chr(j) + chr(k) + chr(l)
                    crc = binascii.crc32(s.encode())
                    if crc == 0xd1f4eb9a:
                        print(f"str1 = {s}")
                    if crc == 0x540bbb08:
                        print(f"str3 = {s}")
                    if crc == 0x2479c623:
                        print(f"str5 = {s}")

    print("end")

find_matches()
```
![image.png](/images/PolarCTF-RE-easy-1/1752587205592.png)

拼起来：flag{ezrebyzhsh}

‍

**PolarCTF**[EasyCPP2]  


![image.png](/images/PolarCTF-RE-easy-1/1752587205647.png)

64位ELF。直接运行发现：没有任何输出，直接等你输入。

IDA启动：  

![image.png](/images/PolarCTF-RE-easy-1/1752587205818.png)

看了眼`flag = qisngksofhuivvmg`

再看眼`encode()`：`+=3`再`^6=1u`

![image.png](/images/PolarCTF-RE-easy-1/1752587205906.png)

所以对`flag = qisngksofhuivvmg`进行`encode()`就是我们的`input`

```C
#include <stdio.h>
#include <string.h>

int main() {
    char flag[] = "qisngksofhuivvmg";
    
    for (int i = 0; i <= 15; i++) {
        flag[i] += 3;
        flag[i] ^= 0x1u;
    }
    
    printf("%s", flag);
    
    return 0;
}
```
**PolarCTF**[一个flag劈三瓣儿]  




![image.png](/images/PolarCTF-RE-easy-1/1752587205989.png)

64位ELF，直接运行：flag{HaiZI233N145wuD!le112@666}  

![image.png](/images/PolarCTF-RE-easy-1/1752587206053.png)

啊？真这么简单wow

  

