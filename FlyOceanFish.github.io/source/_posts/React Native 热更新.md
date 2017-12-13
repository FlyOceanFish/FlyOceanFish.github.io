---
title: React Native热更新 # 这是标题
date: 2017-12-08 15:40:00
categories:  # 这里写的分类会自动汇集到 categories 页面上，分类可以多级
- React Native # 一级分类
tags:   # 这里写的标签会自动汇集到 tags 页面上
- React Native # 可配置多个标签，注意格式
- iOS
---
## 介绍

笔者这些天一直在写一个比较复杂的React Native联系项目，也将近接近尾声了。之前也写过一个[FlyOceanMovies](https://github.com/FlyOceanFish/FlyOceanMovies)，但是相对来说比较简单吧。所以这次挑了一个很复杂的项目又写了一个。不过本文的目的不是这个项目的总结，之后会单独写一篇文章来做总结。这篇文章是来介绍React Native热更新的。

大家都知道React Native最大的优点就是能够同时兼容Android、iOS两个平台，不过它还有一个比较明显的有点就是能够热更新。那如何实现RN的热更新呢？

市面上也有比较成熟的热更新服务[MicroSoft CodePush](https://microsoft.github.io/code-push/)和[React Native中文网 pushy](https://github.com/reactnativecn/react-native-pushy),不过很多时候公司可能要搭建自己的热更新服务器，毕竟将代码和控制器放到他人服务器上可控性等方面都不是很好。笔者研究总结了以下个比较成熟的方案。

## 热更新方案

### 全量热更新

这个热更新实现起来比较简单，省时省力。
![流程.png](http://upload-images.jianshu.io/upload_images/6644906-74796b39f03495df.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)


### 增量热更新

实现起来比较复杂，省流量，用户体验会更好

#### 搭建自己的codepush服务器
其实就是在自己服务器上搭建一套codepush系统，这样比较简单也省事。除了服务器地址与微软的不一样，其他的完全一样。

#### 增量热更新(通过算法比较生成补丁)

1、服务端使用bsdiff算法将老RN包和新RN包生成一个补丁patch文件，供客户端下载。

2、客户端下载patch文件，使用将补丁patch文件和老RN包生成一个新RN包。
![增量更新流程.png](http://upload-images.jianshu.io/upload_images/6644906-40709ffb66d14e87.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
* [google-diff-match-patch](https://code.google.com/archive/p/google-diff-match-patch/)

这个算法是通过文本进行比较支持的语言非常多

* [bsdiff、bspatch](http://www.daemonology.net/bsdiff/)

是c语言写的一个工具，比较的是二进制文件

**二者比较图：**

![比较图.png](http://upload-images.jianshu.io/upload_images/6644906-9a9076ef91ed6185.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

## 参考文章
* [React Native热更新方案](https://yq.aliyun.com/articles/74390)
* [自己搭建code-push-server](https://github.com/lisong/code-push-server)
* [自己服务器的code-push 热更新例子](https://github.com/lisong/code-push-demo-app)
* [ios-bsdiff demo](https://github.com/tcguo/ios-bsdiff)
