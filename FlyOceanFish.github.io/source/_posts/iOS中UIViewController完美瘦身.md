---
title: iOS中UIViewController完美瘦身 # 这是标题
date: 2019-05-29 10:43:00
categories:  # 这里写的分类会自动汇集到 categories 页面上，分类可以多级
- iOS # 一级分类
tags:   # 这里写的标签会自动汇集到 tags 页面上
- iOS
---
对于`UIViewController`瘦身是一个老生常谈的问题，现在也有比较多的架构来实现此效果，比如MVVM等等。不过这次我们是对于传统的MVC架构设计实现完美的瘦身。此方法也完全不妨碍在此基础上使用MVVM等其他方法瘦身。

# 情景
首先让我们看一下我们要做的界面效果
![](https://upload-images.jianshu.io/upload_images/6644906-a62f73002e0751fb.jpeg?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

看着界面不是很难，最上边是二维码扫描；中间输入一些信息，数字1、2、3是按钮点击网络请求然后弹出选择界面供用户选择；最下边历史数据。

其实写下来交互非常的多，接口非常多，业务逻辑也非常的复杂，
传统MVC下来整个.h文件估计至少1500行以上。

# 瘦身思路

通过上图中的两根红色实现，我们将此界面划分成三部分。则声明四个`UIViewController`:
分别为：
- `ScanViewController` 扫描功能
- `InfoInputViewController`输入信息
- `HistoryViewController`历史数据
- `MyViewController`组装功能
在`MyViewController`中我们进行组装，并处理一些非业务事件

````c
self.myScanVC = [[ScanViewController alloc] init];
[self addChildViewController:self.myScanVC];
[self.view addSubview:self.myScanVC.view];
[self.myScanVC didMoveToParentViewController:self];

self.myInforInputVC = [[InfoInputViewController alloc] init];
[self addChildViewController:self.myInforInputVC];
[self.view addSubview:self.myInforInputVC.view];
[self.myInforInputVC didMoveToParentViewController:self];

self.myHistoryVC = [[HistoryViewController alloc] init];
[self addChildViewController:self.myHistoryVC];
[self.view addSubview:self.myHistoryVC.view];
[self.myHistoryVC didMoveToParentViewController:self];
````
通过此方法，我们第一步已经成功降低了耦合度，而且在项目中还有更大的一个优点就是***`ScanViewController`和`HistoryViewController`两个类可以达到复用的效果，其他功能也需要这两个功能***，所有的业务逻辑全部在`InfoInputViewController`中

---

由于`InfoInputViewController`中也有非常多的业务逻辑而且以按钮点击之后开始响应为主。
现在看一下我们的点击事件有多简洁:
```c
- (IBAction)actionClick:(UIButton *)sender {
    [self.view endEditing:YES];
    [self.actionsContext doActionWithLevel:sender.tag];
}
```
是的没有任何判断，而且我们把所有的业务逻辑均封装在了一个一个的Action中。
下午是我们类结构的设计
![](https://upload-images.jianshu.io/upload_images/6644906-bc298bc89f5a7431.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

以上设计思路其实我有单独写过[《设计模式之感悟和实践(二)》](http://flyoceanfish.top/2019/01/08/设计模式实践2/)
具体代码文章中也有。

通过此方法既消除了if...else的判断，同时每个点击时间的业务逻辑也就行了单独的封装。

---
通过以上两个方法的结合成功了将“UIViewController”瘦的简直不能再瘦了！！
