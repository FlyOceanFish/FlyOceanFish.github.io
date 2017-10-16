---
title: SDWebImage源码解读与学习（一） # 这是标题
date: 2017-10-14 10:27:00
categories:  # 这里写的分类会自动汇集到 categories 页面上，分类可以多级
- iOS # 一级分类
tags:   # 这里写的标签会自动汇集到 tags 页面上
- iOS
---
SDWebImage可以说是一个家喻户晓的一个优秀的图片加载框架，极大的方便了我们加载网络图片。好的代码和框架都是我们学习的资源，所以我将一点一点的，从全局到局部的思路对SDWebImage的源码进行解读。

首先让我们了解一下这么优秀的框架都干了啥，有哪些优点！
## 优点
-  通过UIImageView、 UIButton的category的方式实现，可以让我们方便的引入和使用
- 异步下载图片
- 在内存和硬盘上异步缓存图片，有自动过期处理机制
- 在后台实现了图片解压
- 保证了同样的图片地址不会下载多次
- 保证了主线程永远不会被阻塞
- 使用GCD和ARC
- 支持UIImage (JPEG, PNG, ...)
- ***支持WebP***

## SDWebImage框架类图
通过此图我们可以了解到SDWebImage整个框架的设计思路，可以看到SDWebImage有哪些核心类和方法(如果不清晰，建议下载下来看)

![SDWebImage框架类图.png](http://upload-images.jianshu.io/upload_images/6644906-69c145b9f4f11725.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
## 核心类的作用
***SDWebImageManager***
通过调用各个不同的类实现了缓存查询和图片的下载
***SDWebImageDownloader***
在内部通过调用下载类去下载我们需要的图片，这个类是通过设计模式的外观模式实现的。即使以后通过其他方式的下载，只要重写一个下载类更改此类的逻辑就好，根本影响不到我们本身代码的调用
***SDWebImageDownloaderOperation***
一个线程，继承于NSOperation，内部通过NSURLRequest和NSURLSession的方式实现图片的下载
***SDWebImageDecoder***
实现了图片的解码
***NSData+ImageContentType***
判断出图片的格式。jpeg、png、gif、tiff、webp
***SDImageCache***
在磁盘或内存图片的缓存和读取。当达到内存警告的时候，清除内存的缓存;当程序UIApplicationWillTerminateNotification时，清除磁盘缓存
***SDImageCacheConfig***
主要配合SDImageCache使用，帮我们配置是否需要解压缩图片、是否关闭iCloud备份、是否缓存到内存、缓存时间、缓存大小
***SDWebImageCompat***
根据屏幕的Scale生成不同Scale的图片
***UIImage+MultiFormat***
根据图片的类型生成不同的图片，包括gif、webp、和其他三种大的类型图片的生成方式
***UIView+WebCache***
将所有的UIView的category的图片下载操作提取出来，做成公用的下载方法
## 使用
````Objective-C
#import <SDWebImage/UIImageView+WebCache.h>
...
[imageView sd_setImageWithURL:[NSURL URLWithString:@"http://www.domain.com/path/to/image.jpg"]
             placeholderImage:[UIImage imageNamed:@"placeholder.png"]];
````
可以看到如果我们UIButton、UIImageView如果想使用网络图片，直接调用一个方法即可实现我们想要的，是不是很简单啊？？
