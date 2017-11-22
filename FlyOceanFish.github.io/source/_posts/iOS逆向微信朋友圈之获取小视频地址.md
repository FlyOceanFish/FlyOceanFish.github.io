---
title: iOS逆向微信朋友圈之获取小视频地址 # 这是标题
date: 2017-11-22 11:14:00
categories:  # 这里写的分类会自动汇集到 categories 页面上，分类可以多级
- iOS # 一级分类
- 安全
tags:   # 这里写的标签会自动汇集到 tags 页面上
- iOS
- 安全
---
## 简介
本文是针对于阅读过相关逆向朋友圈小视频的人，如果没有看过的话，阅读本文应该会一脸懵逼，所以建议大家可以搜一篇研究一下。

在逆向的过程中，大家拿微信练手占绝大一部分，一般实现的功能有将朋友圈小视频保存到本地(现在的微信原生版已经有了这个功能)或者转发朋友圈等功能。一步一步的怎么逆向我也不啰嗦了，一搜的话肯定也一大堆资料。那本文到底说的是什么呢？

网上千篇一律其实本质都教你怎么一步一步的拿到`WCDataItem`,进而拿到小视频的地址。这里说的就是以另外一种非常简单的方法去获取。

## 获取小视频视图名字
我这里首先是通过Xcode的Debug View Hierarchy得到到小视频视图名字`WCTLContentItemTemplateVideo`,(有的教程是`WCContentItemViewTemplateNewSigh`,我这里的微信版本是6.5.21比较新，所以旧的版本有可能是这个名字)

## 定位小视频长按事件
`WCTLContentItemTemplateVideo`通过class-dump处理的.h文件中可以找到几个长按方法，一个一个断点测试可以知道`- (void)onLongTouch`就是长按事件。

打断点验证:
```
br s -a 0x000000000259c000+0x000000010261092c
```
`0x000000000259c000`通过`image list -o -f WeChat`获取
`0x000000010261092c`通过Hopper获取到
![1511327895597.jpg](http://upload-images.jianshu.io/upload_images/6644906-61c08c5f5b47a55b.jpg?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

---
做到这里与网上的基本类似。只是试图不一样而已。接下来就是重点了。

## 找到`WCDataItem`
既然`WCTLContentItemTemplateVideo`这个视图就是显示小视频的，所以肯定会有数据源的。查看这个视图看了一下没有什么有价值的类。不过再去观看的时候会发现它是继承于`WCContentItemBaseView`，到这个类观察一下。
![1511328262098.jpg](http://upload-images.jianshu.io/upload_images/6644906-cb0d85c950682826.jpg?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
看到这个大家则会豁然开朗了。

```
po [[[[[$x0 oDataItem]contentObj]mediaList]lastObject]dataUrl
```
![1511328559662.jpg](http://upload-images.jianshu.io/upload_images/6644906-f382cd43eb411aeb.jpg?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

通过如上命令可以验证出这个就是我们要找的小视频的地址

接来怎么做相信大家肯定知道了吧。哈哈哈
