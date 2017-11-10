---
title: iOS逆向之工具使用 # 这是标题
date: 2017-11-10 14:00:00
categories:  # 这里写的分类会自动汇集到 categories 页面上，分类可以多级
- iOS # 一级分类
- 安全
tags:   # 这里写的标签会自动汇集到 tags 页面上
- iOS
- 安全
- 逆向
---
上一篇已经大体给大家介绍了几款工具，相信大家都有所了解了。如果不了解可以再去看看。哈哈。言归正传，接下来就是通过一个案例展示一下Hopper、ios-deploy等这些工具的结合使用；另一方面就是让大家体验下lldb中break等调试语句的使用

这次给大家演示的是通过这些工具获取当前应用对应的视图控制器(接下来整个流程看着很复杂，其实这个过程只是展示工具的使用，如果真想实现这个功能有个更简单的方法，大家有可能想到了，没想到也没关系，我在文章的最后边给大家揭露)

## 获取当前可见视图对应的控制器

* 真机连接，开始lldb远程调试

注意执行下列语句要保证终端已经cd到WeiXin.app目录下

>ios-deploy --debug --bundle WeiXin.app

执行完这个语句要等一会APP安装到手机并且启动之后，终端出现如下以`(lldb)`开头了代表启动成功，可以远程调试了。

![2ADCF3E1-88BC-4F98-9F72-693A56F113CB.png](http://upload-images.jianshu.io/upload_images/6644906-b722309c13a8e411.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

* 打全局断点

>br set -n viewWillAppear:

这个时候只要进入任何界面都会停住同时终端会如下显示(关于lldb中break等语句的使用推荐大家看这篇文章[lldb使用技巧](http://blog.csdn.net/quanqinyang/article/details/51321338))

![0653D621-991E-4DF4-B916-7952B0C343EE.png](http://upload-images.jianshu.io/upload_images/6644906-ce25bacf06f61740.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

图中我标出的1代表断点的名称，删除等操作用的到

操作APP，这时候会停住，终端如下图：

![4DE7AB92-DD65-440B-A8EB-B40E0C21A634.png](http://upload-images.jianshu.io/upload_images/6644906-2bf096c3dafbba94.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

这个时候我们就拿到了当前视图对应的控制器的名字了。然后输入`br dis`回车，再输入`c`就可以让代码继续了。


* 获取当前视图名称

>(lldb) po $x0
<MMUINavigationController: 0x11c1a3c00>

>(lldb) po [(MMUINavigationController*)0x11c1a3c00 viewControllers]

大家看有的教程是`po $r0` 这是命令是对应32位机器操作的；`po $x0`是对应64位的命令

![1DB7FB69-8138-4231-BB7F-5461BCC2E14F.png](http://upload-images.jianshu.io/upload_images/6644906-9290ec48aea4acd1.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

**上边是通过断点的方式获取到了当前视图控制器的名字，接下来给大家展示另一种方式**

第二种：
````
(lldb) e UIApplication *$app = [UIApplication sharedApplication]
(lldb) e UIWindow *$keyWindow = $app.keyWindow

(lldb) po $keyWindow.rootViewController
<MMTabBarController: 0x138947000>

(lldb) e MMTabBarController *$tab = $keyWindow.rootViewController

(lldb) po $tab.viewControllers
<__NSArrayM 0x139775c50>(
<MMUINavigationController: 0x1380a2a00>,
<MMUINavigationController: 0x138938a00>,
<MMUINavigationController: 0x13893d600>,
<MMUINavigationController: 0x138943600>
)

(lldb)e MMUINavigationController *$navi2 = $tab.viewControllers[2]

(lldb) po $navi2.visibleViewController

(SeePeopleNearbyViewController *) $8 = 0x00000001398f55c0
````
这种方式其实就是通过lldb远程调试中的`e`语法能够执行任何oc语句的特性与Cycript类似

第三种：

直接打开Xcode的View Debug Hierarchy即可。😁😁

## 通过内存地址打断点调试

首先要给大家普及几个知识，接下来做起来可能就得心应手了。

>模块偏移后的基地址 = ASLR偏移量+ 模块偏移前基地址

* 模块偏移后的基地址:这个地址才是对象真正的地址，我们调试过程中使用的地址都是此地址。

* 模块偏移前基地址：这个其实就是Hopper解析我们应用中的所显示的地址

* [ASLR(Address Space Layout Randomization,地址空间格局的随机化)](http://blog.csdn.net/better0332/article/details/5262990)偏移量：这是ASLR所随机产生的一个内存地址.其实就是一个安全措施，`模块偏移前基地址`通过任何汇编工具都很容易能够拿到，不过要想拿到`ASLR`偏移量就没那么简单。

### ASLR偏移量

>image list -o -f "WeChat"

![A612F9FE-E91D-479F-A6F1-E48CD50E7E0F.png](http://upload-images.jianshu.io/upload_images/6644906-5a36325fcfffb06b.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

其中`0x0000000000300000`就是**ASLR偏移量偏移量**

### 获取模块偏移前基地址

将.app文件拖到Hopper应用中，注意选择64位的要不计算的地址不对，要与你手机的位数匹配起来。过程可能比较忙耐心等待一下

我们拿拦截消息为例：在Hopper中搜索函数的名称
![A97E9160-D079-4330-8E8F-14710F764DC3.png](http://upload-images.jianshu.io/upload_images/6644906-3fc1812e14302597.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

`0x00000001029c2964`就是我们要获取的**模块偏移前基地址**

大家找一个计算器这是16进制的加起来即可。

0x102cc2964

打断点

>br a -s 0x102cc2964

接下来各位可以找一个美女发个消息看看啦。哈哈

![5DF1BE04-2972-450B-BA6B-BF58DA855679.png](http://upload-images.jianshu.io/upload_images/6644906-8fcf2c9181285933.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

>ni

>(lldb) po $x0
<CMessageMgr: 0x1c41b8aa0>
