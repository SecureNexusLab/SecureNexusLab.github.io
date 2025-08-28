---
layout: page
title: PolarCTF[RE][easy]全解(下)
author: 白玉京
cover: true
sidebar: []
readmore: true
date: 2024-10-21
tags: 
- 公众号推文
- CTF
categories:
- 公众号推文
---
**引言**

  

本次分享PolarCTF靶场中所有[easy]re题目的WP。

由于文章篇幅较长（一共有22道题目），分为两期发表，本期书接前文，分享下半部分内容，适合刚接触逆向的朋友。如果你也对逆向感兴趣，或者正在准备CTF比赛，希望这些内容能给大家一些帮助。

  


**PolarCTF**[C\^]  


![image.png](/images/PolarCTF-RE-easy-2/1752587208839.png)

64位ELF，直接运行：关键字符串“Please enter flag”  

![image.png](/images/PolarCTF-RE-easy-2/1752587208944.png)

IDA启动！  

![image.png](/images/PolarCTF-RE-easy-2/1752587209157.png)

`  
`

`fun1()`：`a1[i] ^= 1u`

  


![image.png](/images/PolarCTF-RE-easy-2/1752587209304.png)

`  
`

`check()`用于判断`a1`是否和`s`相等：`s = shfiu777`

  


![image.png](/images/PolarCTF-RE-easy-2/1752587209369.png)

所以`flag = shfiu777 ^ 1u`

*   *   *   *   *   *   *   *   *   *   *   *

```c
#include <stdio.h>

int main() {
    char flag[] = "shfiu777";
    
    for (int i = 0; i < 8; i++) {
        flag[i] ^= 0x1u;
    }
    
    printf("%s", flag);
    
    return 0;
}
```



再md5，32位小写加密即可



**PolarCTF**[babyRE]  


![image.png](/images/PolarCTF-RE-easy-2/1752587209430.png)

  

64位exe。运行以后发现，随便输入会输出“Err"  

  

![image.png](/images/PolarCTF-RE-easy-2/1752587209565.png)

  

看眼`endoce()`：对`flag`的每位+2  

  

![image.png](/images/PolarCTF-RE-easy-2/1752587209759.png)

  

那flag是啥嘞：`shift+f12`里有一串诡异字符。可以试试

![image.png](/images/PolarCTF-RE-easy-2/1752587209880.png)

```C
#include <stdio.h>
#include <string.h>  // Fixed from `<string>` to `<string.h>`

int main() {
    char flag[] = "asdfgcvbnmjgtlop";
    
    for (int i = 0; i < strlen(flag); i++) {
        flag[i] += 2;
    }
    
    printf("%s", flag);
    
    return 0;
}
```



**PolarCTF**[easyre1]  




![image.png](/images/PolarCTF-RE-easy-2/1752587210029.png)

  

64位ELF。运行发现：会输出”no no no"

  

![image.png](/images/PolarCTF-RE-easy-2/1752587210136.png)

  

IDA启动！

  

![image.png](/images/PolarCTF-RE-easy-2/1752587210254.png)

  

挨个看吧。

`enkey()`：循环32次，134520896 + 96 = 134520992  

  

![image.png](/images/PolarCTF-RE-easy-2/1752587210329.png)

  

看看这俩内容都是啥：

`134520896: key = 5055045045055045055045055045055`  

  

![image.png](/images/PolarCTF-RE-easy-2/1752587210384.png)

`  
`

`134520992: flag`的位置  

  

![image.png](/images/PolarCTF-RE-easy-2/1752587210467.png)

  

所以`enkey()`就是让`flag`和`key`按位异或。

‍

`reduce()`：循环31次，`flag`每位都-1

![image.png](/images/PolarCTF-RE-easy-2/1752587210529.png)

`  
`

`check()`：对比`flag`和`d^XSAozQPU^WOBU[VQOATZSE@AZZVOF`

  


‍![image.png](/images/PolarCTF-RE-easy-2/1752587210596.png)

  

反过来就是：

  * `d^XSAozQPU^WOBU[VQOATZSE@AZZVOF`每位都+1

  * 然后和`key`按位异或

*   *   *   *   *   *   *   *   *   *   *   *   *   *   *

```c
#include <stdio.h>
#include <string.h>  // Fixed from <string> to <string.h> for strlen()

int main() {
    char flag[] = "d^XSAozQPU^WOBU[VQOATZSE@AZZVOF";
    char key[] = "5055045045055045055045055045055";
    
    for (int i = 0; i < strlen(flag); i++) {
        flag[i]++;
        flag[i] ^= key[i];
    }
    
    printf("%s", flag);
    
    return 0;
}
```

​    

**PolarCTF**[Sign Up]  




![image.png](/images/PolarCTF-RE-easy-2/1752587210715.png)

  

64位exe。直接运行发现关键字符串。

  

![image.png](/images/PolarCTF-RE-easy-2/1752587210830.png)  

IDA启动！  


![image.png](/images/PolarCTF-RE-easy-2/1752587210936.png)

  

![image.png](/images/PolarCTF-RE-easy-2/1752587211026.png)

  

非常eazy啊，`key_num[i]-1 = name; key_password[i]-2 = password`

`key_num = 192168109; key_password = root`

  


![image.png](/images/PolarCTF-RE-easy-2/1752587211206.png)




*   *   *   *   *   *   *   *   *   *   *   *   *   *   *   *   *   *   *   *

```C
#include <stdio.h>
#include <string.h>  // Changed from <string> to <string.h> for strlen()

int main() {
    char key_num[] = "192168109";
    char key_password[] = "root";
    
    // Process key_num (decrement each character by 1)
    for (int i = 0; i < strlen(key_num); i++) {
        key_num[i]--;
    }
    
    // Process key_password (decrement each character by 2)
    for (int i = 0; i < strlen(key_password); i++) {
        key_password[i] -= 2;
    }
    
    printf("name = %s\n", key_num);
    printf("password = %s\n", key_password);
    
    return 0;
}
```



  


检验一下：

  

![image.png](/images/PolarCTF-RE-easy-2/1752587211274.png)

拼接起来md5即可。

等等，不对？

原来是眼睛不好使，没注意人家只替换了7个数

  

![image.png](/images/PolarCTF-RE-easy-2/1752587211501.png)

```C
#include <stdio.h>
#include <string.h>  // Correct header for strlen()

int main() {
    char key_num[] = "192168109";
    char key_password[] = "root";
    
    // Process first 7 characters of key_num (decrement by 1)
    for (int i = 0; i <= 6; i++) {
        key_num[i]--;
    }
    
    // Process all 4 characters of key_password (decrement by 2)
    for (int i = 0; i <= 3; i++) {
        key_password[i] -= 2;
    }
    
    printf("name = %s\n", key_num);
    printf("password = %s\n", key_password);
    
    return 0;
}
```



这下对了！

**PolarCTF**[? 64]  


![image.png](/images/PolarCTF-RE-easy-2/1752587211581.png)

64位可执行。

![image.png](/images/PolarCTF-RE-easy-2/1752587211655.png)

这，我猜是base64，直接在线解密试试：

![image.png](/images/PolarCTF-RE-easy-2/1752587211795.png)

赢！但是还是看看程序  


![image.png](/images/PolarCTF-RE-easy-2/1752587211880.png)

  

![image.png](/images/PolarCTF-RE-easy-2/1752587211975.png)

遗憾！并不是base64的加密程序QAQ。收工

**PolarCTF**[Why32]  


![image.png](/images/PolarCTF-RE-easy-2/1752587212229.png)

64位exe

  


![image.png](/images/PolarCTF-RE-easy-2/1752587212299.png)

  

![image.png](/images/PolarCTF-RE-easy-2/1752587212504.png)

  

![image.png](/images/PolarCTF-RE-easy-2/1752587212742.png)

  

![image.png](/images/PolarCTF-RE-easy-2/1752587212829.png)

  

这几个函数连起来的意思就是：`input.len == 32`

继续看`Do()`函数：`input = cAry[i]-2; cAry = "2gfe8c8c4cde574f7:c6c;:;3;7;2gf:"`

![image.png](/images/PolarCTF-RE-easy-2/1752587212918.png)

```C
#include <stdio.h>
#include <string.h>  // Changed from <string> to <string.h>

int main() {
    char key_num[] = "2gfe8c8c4cde574f7:c6c;:;3;7;2gf:";
    
    // Process each character (subtract 2 from ASCII value)
    for (int i = 0; i <= 31; i++) {
        key_num[i] -= 2;
    }
    
    printf("%s\n", key_num);
    return 0;
}
```



这个half right是什么意思呢？

不管了先直接包裹上flag提交试试，比较这个看起来很像md5加密之后的值。

‍

豪德，不对。

解密试试呢？

![image.png](/images/PolarCTF-RE-easy-2/1752587212997.png)

`F1laig`。这下对了！

‍

**PolarCTF**[康师傅]  




![image.png](/images/PolarCTF-RE-easy-2/1752587213073.png)

  

32位exe。IDA直接跟进`main`函数

  

![image.png](/images/PolarCTF-RE-easy-2/1752587213184.png)



`input[i] ^= 9u == str1[i]`

太简单了哇！直接`str1[i] ^= 9u`就是flag了

*   *   *   *   *   *   *   *   *   *   *   *

```C
#include <stdio.h>
#include <string.h>  // Changed from <string> to <string.h> for strlen()

int main() {
    char flag[] = "oehnr8>?;<?:9k>09;hj00o>:<o?8lh;8h9l;t";
    
    // XOR each character with 9
    for (int i = 0; i < strlen(flag); i++) {
        flag[i] ^= 9u;
    }
    
    printf("%s\n", flag);
    return 0;
}
```



**PolarCTF**[re2]  


![image.png](/images/PolarCTF-RE-easy-2/1752587213265.png)

  

64位ELF，IDA直接启动。

  

![image.png](/images/PolarCTF-RE-easy-2/1752587213349.png)

**PolarCTF**[layout]  


![image.png](/images/PolarCTF-RE-easy-2/1752587213449.png)

  

下载下来发现是apk。安装到雷电模拟器里看看。

  

![image.png](/images/PolarCTF-RE-easy-2/1752587213536.png)

疑似没做竖屏适配。调设置重启一下，发现还是乱码。

OK！上手段——jadx打开，直接搜索`flag{`

![image.png](/images/PolarCTF-RE-easy-2/1752587213665.png)

不对？！

好吧。回到雷电模拟器，用开发者助手提取：

![image.png](/images/PolarCTF-RE-easy-2/1752587213869.png)

这下对了

‍

**PolarCTF**[use\\_jadx\\_open\\_it]  


这个名字——我听话，用 jadx 直接打开，然后搜索字符串：

![image.png](/images/PolarCTF-RE-easy-2/1752587213942.png)

结束

‍

**【未完成】****PolarCTF**[另辟蹊径]  


![image.png](/images/PolarCTF-RE-easy-2/1752587214027.png)

32位exe。但是注意：`Section`是乱码。丢进虚拟机里运行发现果然运行不了。

拽进ida里发现会创建一个新文件，感觉不对。 然后搜了一下writeup，好像这个文件有毒。暂停解题

**PolarCTF**[JunkCode]  


![image.png](/images/PolarCTF-RE-easy-2/1752587214101.png)

32位可执行文件。IDA没看出名堂，进x64dbg试试。

根据题目名，猜测有很多无用代码，所以直接搜索字符串找到关键代码部分

![image.png](/images/PolarCTF-RE-easy-2/1752587214176.png)

简单分析（内容写注释里了）：

![image.png](/images/PolarCTF-RE-easy-2/1752587214303.png)

分析认为`junkcode.1F1258`是判断函数：

  * 调用该函数后，有个jump，一个会输出funny（疑似成功？），一个会输出"no"。

  * 运行测试时，发现输错 flag 就会输出"no"

![image.png](/images/PolarCTF-RE-easy-2/1752587214420.png)

‍

下断点准备跟进去看，结果发现eax里就是flag~

  

**END**  
本文作者：白玉京  

  

  

