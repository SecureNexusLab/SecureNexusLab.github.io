---
title: 针对API漏洞挖掘技巧学习
date: 2024-07-09 09:03
cover: true
sidebar: []
readmore: true
tags: 
- 公众号推文
- Web安全
categories:
- 公众号推文
---

## 前言

首先我们需要了解API基本的一些知识，我们首先来看几个GET方式的API
```
GET /api/books HTTP/1.1
Host: example.com
```
首先上面这种，是api的端点，也就是请求点，通过交互获得图书馆的图书列表，例如另一个端点可能是`/api/books/mystery`，那么这个可能可以检索书籍列表。

一旦确认了端点，就需要确定怎么样可以和他们产生交互，也就是触发检索。

## API文档

api一半都会有相对应的文档，以便开发人员去使用，通常是公开可用的，但是也有不公开的，我们也可以使用api的程序来访问。例如：
```
- /api
- /swagger/index.html
- /openapi.json
```
如果标识了资源终结点，需要调查一下基本路劲，比如：表示了终结点/api/swagger/v1/users/123，就需要调查以下路径

```
- /api/swagger/v1
- /api/swagger
- /api
```

## 靶场一、使用文档利用API端点

我们想要完成这个靶场，需要知道几点：

什么是API文档，API文档对攻击者如何使用，如何发现API文档

这里我是用插件`findsomething`找到页面中的api

![](/images/vuln-mining-skills-for-api/1.png)

访问之后提示我们缺少参数，我们往上一级目录，也就是api目录访问看看
![](/images/vuln-mining-skills-for-api/2.png)

发现了说明文档

我们直接点delete就可以直接对指定用户进行删除，但是这里回显是401权限不足，根据靶场提供的信息我们wiener用户，再次访问即可删除。当然，我们也可以通过抓包，去查看这个api的使用参数，仿照发送请求，达到任意控制效果

![](/images/vuln-mining-skills-for-api/3.png)

## 常见支持HTTP的方法

- GET - Retrieves data from a resource.
  GET -从资源中检索数据。
- PATCH - Applies partial changes to a resource.
  PATCH -删除对资源的部分更改。
- OPTIONS - Retrieves information on the types of request methods that can be used on a resource.
  OPTIONS -检索有关可在资源上使用的请求方法类型的信息。

研究api端点时，测试方法很重要，比如我们知道端点/api/tasks，我们可以尝试以下方法

- GET /api/tasks - Retrieves a list of tasks.
  GET /api/tasks -检索任务列表。
- POST /api/tasks - Creates a new task.
  POST /api/tasks -创建一个新任务。
- DELETE /api/tasks/1 - Deletes a task.
  DELETE /api/tasks/1 -完成任务。

## 靶场二、查找利用未使用的API端点

该靶场，首先我们需要找到未被使用的api端点，上面一个靶场我们是找不到的，这里根据靶场提示，我们挨个点击靶场中购买，走一遍购买流程，我们可以在数据记录中，找到一个隐藏的api
![](/images/vuln-mining-skills-for-api/4.png)

这个是我们在提交购买的时候产生的，我们将这个发送到重放器中 

尝试使用不同的方式进行排查，比如我们可以尝试使用/api/products/1，或者/api/products、/api来排查所有的内容，但是这里均无法响应

那么下一步我们可以尝试不同的方式

比如我这里使用post
![](/images/vuln-mining-skills-for-api/5.png)

这里提示不支持该方法，并且告诉了我可用的方法，这里我们试试
![](/images/vuln-mining-skills-for-api/6.png)

提示内部服务错误，我们在下面加上括号
![](/images/vuln-mining-skills-for-api/7.png)

提示我们缺少price参数

加上参数

![](/images/vuln-mining-skills-for-api/8.png)

提示必须是非负整数，我们去掉引号试试

![](/images/vuln-mining-skills-for-api/9.png)

提示我们需要`Content-Type: application/json`

我们复制放到下面

![](/images/vuln-mining-skills-for-api/10.png)

这里成功修改了价格，我们将价格修改为0元，购买即可通关。

![](/images/vuln-mining-skills-for-api/11.png)



本关卡以api传递参数的方式，让我们成功修改了参数。

## 识别隐藏参数

我们通常可以看到，一个api请求，他会允许我们修改某些东西

PATCH /api/users/请求它允许用户更新他们的用户名和电子邮件，并包含以下JSON：

```
{"username": "wiener", "email": "wiener@example.com", }
```
返回的信息是以下JSON
```
{"id": 123, "name": "John Doe", "email": "john@example.com", "isAdmin": "false" }
```

这表示隐藏的id和参数，可能可以进行改变使用

我们想要测试上面的isadmin参数，可以将上面的参数修改后发送到PATCH请求

```
{"username": "wiener", "email": "wiener@example.com", "isAdmin": false, }
```

如果我们将false修改为true，那么在没有充分验证的情况下，有可能会错误绑定对象，获取权限。

## 靶场三、利用批量分配漏洞
该靶场，我们需要分析一下，流量包中的api，根据提示，我们在购买的过程中，找到两个api的包
![](/images/vuln-mining-skills-for-api/12.png)

一个get一个post两个数据包，这里我们可以在post数据包中看到一个数据结构

![](/images/vuln-mining-skills-for-api/13.png)

在GET数据包中，可以在返回包中发现一些隐藏的数据传递方式

![](/images/vuln-mining-skills-for-api/14.png)

我们可以尝试拼接到post数据包中进行尝试
![](/images/vuln-mining-skills-for-api/15.png)

这里报错提示我们资金不足， 我们尝试改变内容
![](/images/vuln-mining-skills-for-api/16.png)

尝试数据改成100，直接通关了
![](/images/vuln-mining-skills-for-api/17.png)

返回true，完成了关卡

![](/images/vuln-mining-skills-for-api/18.png)

该靶场的问题，在于，我们可以从GET中获取到一些隐藏的参数，在得到隐藏参数之后，我们可以通过post或者其他的方法进行发送，尝试执行。