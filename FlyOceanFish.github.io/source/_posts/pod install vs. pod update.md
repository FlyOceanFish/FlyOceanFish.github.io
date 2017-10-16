---
title: pod install vs. pod update # 这是标题
date: 2017-10-14 09:27:00
categories:  # 这里写的分类会自动汇集到 categories 页面上，分类可以多级
- iOS # 一级分类
tags:   # 这里写的标签会自动汇集到 tags 页面上
- iOS
---
# 背景　　
　　作为iOS管理第三方包很好用的一个工具CocoaPods，大部分在使用过程中一开始使用pod install,之后就一直使用pod update。其实这种做法是不对，本篇文章将详细介绍这几个命令的区别，也希望iOS开发者能够合理使用这几个命名
## pod init
　　初始化一个工程，此时会产生Podfile这么一个文件，我们可以添加需要引入的第三方包

## pod install
    pod install --verbose
***用这个命令可以看到所有的执行操作***
　　第一次构建工程时要使用该命令，但是如果我们编辑了Podfile文件，例如添加、删除、更新了一个pod也会使用该命令
* 我们运行这个命令的时候，开始下载安装新的pods，它会将我们每一个第三方包的版本写入一个叫Podfile.lock的文件。这个文件包含了我们安装的所有的第三方包的信息
* 当我们运行这个命令的时候，它只会下载不在Podfile.lock文件里的第三方包
 - 对于Podfile.lock里边有的详细的第三方包（有版本)，它只会下载对应的版本，不管这个第三方包有没有新的版本
 - 如果Podfile.lock里边没有的第三方包，它将下载你Podfile里边写的（例如pod 'MyPod', '~>1.2'），如果不写版本默认下载最新的版本

## pod update
　　如果我们运行pod update PODNAME 将只更新这一个第三方包到最新的版本
如果我们直接运行pod update 没有pod名称，它将会更新我们所有的第三方包
## pod outdated
　　将列出所有的在Podfile.lock文件里的所有第三方包现在对应的最新的版本
## 总结
　　如果我们在Podfile文件增加、减少、或者改变了版本的时候运行pod install；如果我们想更新第三包的时候使用pod update命令

接下来一篇我将详细介绍如果创建自己的spec。
