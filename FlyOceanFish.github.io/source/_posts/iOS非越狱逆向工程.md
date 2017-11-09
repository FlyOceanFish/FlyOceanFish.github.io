---
title: iOS非越狱逆向工程 # 这是标题
date: 2017-11-09 13:45:00
categories:  # 这里写的分类会自动汇集到 categories 页面上，分类可以多级
- iOS # 一级分类
- 安全
tags:   # 这里写的标签会自动汇集到 tags 页面上
- iOS
- 安全
- 逆向
---
这几天一直在研究iOS的安全，也动手搞了一下iOS的逆向工程微信。有两个成果吧。一个是拦截了信息；另外一个就是拦截了定位，这样附近的人等需要用到经纬度功能的地方都可以自定义自己想要的经纬度。例如可以定位到日本，去看看日本的美眉。哈哈。。

由于之前的iOSOpenDev已经不维护，所有网上有个大神对此做了一个升级，最主要是可以傻瓜式使用了，dylib注入、打包等等一系列自动化。[iOSOpenDev](https://github.com/AloneMonkey/MonkeyDev)

虽然都是自动化了，不过本人还是尝试了手工dylib注入、打包等功能。首先让我们先看看这个<font color='red' size='4'>非越狱神器</font>的神奇之处，让我先体验一把。（安装的话自己对照官网安装即可，我这里就不啰嗦了！）

## 尝试自动化工具

大家首先按照官网的教程新建一个MonkeyApp工程，我这里新建的是`WeiXin`,接下来我们需要一个已经敲掉壳的ipa文件，如果你自己按照教程一步一步的搞也可以，不过还有更省事的方法，就是通过pp助手在越狱应用中下载一个越狱版微信即可，放到指定路径下。

### 编写代码

写代码的地方有好几个文件其实都是可以写的，只是不同的语法而已。比如.xm后缀的就是支持Logos语法,例如`%hook`、`%orig`等语法。不过本人写在`WeiXinDylib.m`这个文件里，这个文件其实最后会生成一个动态包名`WeiXinDylib`，这个包就是我们需要注入到APP中的。

* 微信消息拦截

在`WeiXinDylib.m`这个文件的最后加上以下代码
```Objective-C
@interface CMessageWrap
@property (nonatomic, strong) NSString* m_nsContent;
@property (nonatomic, assign) NSInteger m_uiMessageType;
@end
CHDeclareClass(CMessageMgr)
CHMethod2(void, CMessageMgr, AsyncOnAddMsg, NSString*, msg, MsgWrap, CMessageWrap*, msgWrap){
    NSString* content = [msgWrap m_nsContent];
    if([msgWrap m_uiMessageType] == 1){
        NSLog(@"收到消息: %@", content);
    }
    CHSuper2(CMessageMgr, AsyncOnAddMsg, msg, MsgWrap, msgWrap);
}
CHConstructor{
    CHLoadLateClass(CMessageMgr);
    CHClassHook2(CMessageMgr, AsyncOnAddMsg, MsgWrap);
}

```
这样的话我们就拦截到了微信收发消息，在Xcode控制台则会打印出我们所收发的消息。

* 微信自定义定位(大家比较期待的功能，想撩哪里的美眉都可以哟！)

同样找到`WeiXinDylib.m`这个文件然后在最后加如下代码

```Objective-C
CHDeclareClass(CLLocationManager)
CHMethod(0,void,CLLocationManager,startUpdatingLocation){
    NSLog(@"定位被拦截");
    CHSuper(0,CLLocationManager, startUpdatingLocation);

    CGFloat lat = 35.707013;
    CGFloat lng = 139.730562;
    CLLocation *tokyoLocation = [[CLLocation alloc] initWithLatitude:lat longitude:lng];
    CLLocation *cantonLocation = [[CLLocation alloc] initWithLatitude:30.231695 longitude:120.283835];
    [NSThread sleepForTimeInterval:4];
#pragma clang diagnostic push
#pragma clang diagnostic ignored "-Wdeprecated-declarations"
    dispatch_after(dispatch_time(DISPATCH_TIME_NOW, (int64_t)(1.0f * NSEC_PER_SEC)), dispatch_get_main_queue(), ^{
        [self.delegate locationManager:self didUpdateToLocation:tokyoLocation fromLocation:cantonLocation];
    });
#pragma clang diagnostic pop
}
CHConstructor{
    CHLoadLateClass(CLLocationManager);
    CHClassHook(0,CLLocationManager, startUpdatingLocation);
}
```

我这里是写死的经纬度，其实大家也可以弹出一个框然后输入也是可以的。代码里拦截到了方法之后我将现场sleep了4秒。主要是如果不sleep我发现这时候回调不起作用的。

总结：其实动态注入的本质就是通过运行时进行method swizzling,进行了方法的替换而已。

## 体验各种手工工具

*  [Hopper Disassembler](https://www.hopperapp.com/)是Mac上的一款二进制反汇编器，跟IDA功能一样
* [Cycript](http://www.cycript.org/) 是一款脚本语言，是混合了objective-c与javascript语法的一个工具，让开发者在命令行下和应用交互，在运行时查看和修改应用。与lldb有点类似
* class-dump(http://stevenygard.com/projects/class-dump/) 能够将工程里边所有的方法提取出来，生成一个.h文件
* [iReSign](https://github.com/maciekish/iReSign) APP重签名，我们也可以使用命令一步一步来签名，这个工具帮我们把命令集成了写了一个界面
* [yololib](https://github.com/KJCracks/yololib) 将dylib动态包注入到APP中
* [ios-deploy](https://github.com/phonegap/ios-deploy.git) 一个非越狱版远程lldb调试工具

这些工具的安装我就不啰嗦了，大家要不就去官网看怎么安装，要不就自行百度了。

我这里可以说一下有的工具下载下来之后，如果使用有时候我们的终端先`cd`到对应的文件下，然后才能使用相应的命令。其实这样也是比较繁琐的。这里我们可以把他们的路径加到.bash_profile文件中，这样在终端则可以直接使用了。

例如我将cycript加到这个文件里，我在终端就可以直接使用了。
![1672149F-06C6-4A6A-82C7-8911DA07DB8D.png](http://upload-images.jianshu.io/upload_images/6644906-a233aea0a7b272f5.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

## 工具使用
使用的话我也只是简单给大家演示一样，起个抛砖引玉的作用，其实每个工具都非常的强大的。

##### class-dump

>class-dump -H 解压完成的APP的文件路径/weixin.app -o /Users/xxxx/Desktop/指定生成文件路径

这样就可以生成很多.h文件了。直接看起来比较费事，大家可以新建一个Xcode工程，然后把文件复制进去，这样看起了就比较舒服了。

通过这个工具我们可以找到我们感兴趣类的方法，这样就可以做对应的拦截修改了。

##### yololib

>yololib 可执行文件 要被注入的.dylib

例如

>./yololib WeiXin.app libWeiXinDylib.dylib

**<font color='red'>注:</font>这样写的话要保证yololib、WeChat、WeChat.app处于同一目录下。**

将我们注入的dylib文件放到WeChat.app目录下。

>cp hook.dylib WeChat.app/

##### iReSign
首先我们要对我们的dylib动态包进行签名

>codesign -f -s "iPhone Developer:xxxx" WeiXin.app/libWeiXinDylib.dylib

然后点击iReSign直接点击运行对ipa包签名
![7E58DAD9-8A11-4461-A9E4-E8C3179380E4.png](http://upload-images.jianshu.io/upload_images/6644906-5433d5d586ec8452.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

选一下对应文件的路径然后点击重新签名即可。那个entitlements.plist文件不选的话也可以成功。不过最好自己新建一个，内容如下：

```
<?xml version="1.0" encoding="UTF-8"?>
 <!DOCTYPE plist PUBLIC "-//Apple//DTD PLIST 1.0//EN" "http://www.apple.com/DTDs/PropertyList-1.0.dtd">
 <plist version="1.0">
 <dict>
     <key>application-identifier</key>
     <string>com.aa.aa.weixin</string>
     <key>get-task-allow</key>
     <false/>
     <key>keychain-access-groups</key>
     <array>
         <string>com.aa.aa.weixin</string>
     </array>
 </dict>
 </plist>

```
大家把`com.aa.aa.weixin`改成自己的bundle id即可。然后复制新建一个文件就可以啦。

##### Cycript

* 找到进程ID
>ps -e | grep WeChat
* 钩住进程
>cycript -p 12435
或
>cycript -r ip:6666

接下来可以干我们想干的事情了，使用任何oc代码。

>cy# alertView = [[UIAlertView alloc] initWithTitle:@"注入" message:@"Cyript" delegate:nil cancelButtonTitle:@"确定" otherButtonTitles:nil]

>cy# [alertView show]

其实我们在逆向工程的时候，大部分是用的内存地址调用
>cy# [#0x108840380 show]

调用方式就是#内存地址,我们可以借助Xcode的Debug View Hierarchy这个工具获取

![1DDDB54A-8837-4BE0-8D2D-C1D91FED3E8F.png](http://upload-images.jianshu.io/upload_images/6644906-e5bd42bf0a096068.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

![效果图.PNG](http://upload-images.jianshu.io/upload_images/6644906-9aa87c880e48b8b7.PNG?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

##### ios-deploy
这个工具与`Cycript`有点类似的。首先就是安装ios-deploy

>ios-deploy --debug --bundle WeiXin.app

运行完之后就可以像Xcode控制器内置的lldb一样，可以在终端执行任意lldb命令啦。比如断点等等

##### Hopper disassembler

这个工具使用起来简单，但是也是非常的强大的。我们打开这个工具，直接将.app拖到这个工具里就可以了。

接下来我们可以搜索全局的方法、类等。甚至可以直接修改指令的，然后再生成对应的APP

## 结论

这一篇主要是理论，让大家有个大体的了解。下一篇来个实战，教大家如果根据内存地址动态打断点，如何获取微信当前的视图控制器的名字，这样我们结合Hopper或者class-dump出来的.h文件进行修改或拦截了。
