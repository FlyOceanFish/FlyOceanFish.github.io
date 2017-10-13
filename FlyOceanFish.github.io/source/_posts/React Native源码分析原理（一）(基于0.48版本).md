---
title: React Native源码分析原理（一）(基于0.48版本) # 这是标题
categories:  # 这里写的分类会自动汇集到 categories 页面上，分类可以多级
- React Native # 一级分类
tags:   # 这里写的标签会自动汇集到 tags 页面上
- React Native # 可配置多个标签，注意格式
- iOS
---
* [React Native源码分析原理（一）(基于0.48版本)](http://www.jianshu.com/p/8324b379c020)
* [React Native源码分析原理（二）(基于0.48版本)](http://www.jianshu.com/p/0688b24950f4)
### React和React Native初步认识
说起Facebook的React Native就不得不说一下React，React是个什么东东呢？
>React 是一个用于构建用户界面的 JAVASCRIPT 库。
React主要用于构建UI，很多人认为 React 是 MVC 中的 V（视图）。
React 起源于 Facebook 的内部项目，用来架设 Instagram 的网站，并于 2013 年 5 月开源。
React 拥有较高的性能，代码逻辑非常简单，越来越多的人已开始关注和使用它。

***React***

>1.声明式设计 −React采用声明范式，可以轻松描述应用。
2.高效 −React通过对DOM的模拟，最大限度地减少与DOM的交互。
3.灵活 −React可以与已知的库或框架很好地配合。
4.JSX − JSX 是 JavaScript 语法的扩展。React 开发不一定使用 JSX ，但我们建议使用它。
5.组件 − 通过 React 构建组件，使得代码更加容易得到复用，能够很好的应用在大项目的开发中。
6.单向响应的数据流 − React 实现了单向响应的数据流，从而减少了重复代码，这也是它为什么比传统数据绑定更简单。

根据以上内容相信大家对React有了一个初步认识。其实高效、简单使用是React最大的特点了。

***React Native***

React Native说白了就是让React拥有了与原生APP交互，能够写接近原生APP的功能，就像它的两个单词一样，一个React，一个Native。有了它就能让js和oc或Android能够方便的相互调用，实现了能够通过js开发接近原生的APP的功能。是的，非常接近原生APP的体检，尤其绘制出的视图全部都是原生的！但是它与混合开发是不同的，一般我们说的混合开发是原生APP和html混合开发，但是体验上是比较差的。React Native也秉承了React的理念 `Learn Once, Write Anywhere`,而不是混合开发所追崇的`Write Once,Run Anywhere`。现在像携程、腾讯等这样的公司已经都使用了React Native写项目了，所以是时候大家也跟进一波，不管用不用，先学习攒着嘛！哈哈。毕竟技多不压身的嘛。

### 通讯框架图和几个核心类的作用

首先让我们看看React Native的js与oc交互框架（个人整理的，非官方的）。

![通讯图.001.jpeg](http://upload-images.jianshu.io/upload_images/6644906-cb4d495b7b74ffd4.jpeg?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
* `RCTCxxxBridge.h` oc与js之间的调用都是通过这个桥梁(0.44以前是RCTBatchedBridge.h这个类)。这个桥梁持有各个具体功能类和一个jsThread,它的作用其实就是一个调度的作用。注意所有js代码的执行都是在此jsThread中执行，这个Thread是放在runloop中的，会保持一直运行的。此线程也是大家常听说的js线程。一个项目中最好只有一个Bridge，要不是比较消耗性能的。
    * `JSCExecutor.h` 所有的js与oc的通信都是通过此类来实现的
    * `BatchedBridge.js`这个js文件仅仅是一个导入文件。导入了`MessageQueue.js`这个文件。
        * `MessageQueue.js`这个文件是js端与oc通信的一个桥梁文件，里边有个计时器将要调用oc方法的事件放入一个queue中，当oc没有及时的在消息队列中调js的时候, 大于5ms后js就会调用这个回调, 主动调用oc
    * `NativeModule.js` 实现了js调用oc代码。大家注意这个调用其实都是oc先调用js文件，执行js，在这时候传入适当的参数，然后实现了js调用oc方法哦。

* `RCTBridgeModule.h` 这个类非常重要，只要想与js实现通信，这个协议必须实现。如果不实现此协议我们js在调用原生方法就会报错，RN在内部会将第一检验的就是类有没有实现这个协议。除了一个标识的作用，还给我们提供了几个宏供我们使用。这几个宏也是js调用原生的具体实现
* `RCTUIManager.h`    `UIManage.js`这两个类一个是在oc端，一个是在js端，通过这两个类RN将我们的component转换成了我们原生的视图类。该类实现了视图的创建、查找、删除等功能。创建View的时候，每个View都有一个
