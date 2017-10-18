---
title: iOS中App Search # 这是标题
date: 2017-10-18 09:40
categories:  # 这里写的分类会自动汇集到 categories 页面上，分类可以多级
- iOS # 一级分类
tags:   # 这里写的标签会自动汇集到 tags 页面上
- iOS
---
自从iOS9.0，苹果提供了多种方式让用户获取APP中的信息，甚至你即使没有安装应用。用户可以通过Spotlight、HandOff等方式获取你APP深层次的一些信息。

## 实现方式

### NSUserActivity

`NSUserActivity`类提供了一些方法，可以让您捕获用户之前访问过的特定应用程序状态和导航点，然后使用Handoff(在您的应用程序中学习更多关于启用切换的知识)来恢复它们。在iOS 8和后来的应用程序中，用户希望Handoff帮助他们在一个设备上启动一个活动，并在另一个设备上继续运行。

````Objective-C
self.activity = [[NSUserActivity alloc] initWithActivityType:@"com.rw.activity"]; 
self.activity.title = @"大头鱼";
self.activity.keywords=[NSSet setWithArray:@[@"tiger",@"horse"]]; 
self.activity.eligibleForSearch = YES;   
self.activity.eligibleForPublicIndexing = YES; 
[self.activity becomeCurrent];
````
![76416.png](http://upload-images.jianshu.io/upload_images/6644906-4a22d5a45c22b6bd.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

> 注意：通过NSUserActivity方式的实现变量要声明成全局变量，否则会被释放掉而没有效果

### Core Spotlight framework


`Core Spotlight`框架提供了在应用程序中索引内容和管理私有设备索引的方法。`Core Spotlight`最适合于索引用户特定的内容，比如朋友、被标记为最喜欢的项目、购买的项目等等。当你使用`Core Spotlight`api使某些内容可以搜索时，可以让用户很容易地在`Spotlight`搜索结果中找到他们的内容。

* 文本
````Objective-C
CSSearchableItemAttributeSet *attributeSet = [[CSSearchableItemAttributeSet alloc] initWithItemContentType:@"my"];
attributeSet.title = @"first";
attributeSet.contentDescription = @"my app is running!";
NSArray *secondArrayKey = [NSArray arrayWithObjects:@"second",@"测试",@"secondeVIew", nil];
attributeSet.contactKeywords = secondArrayKey;
attributeSet.keywords = @[@"5月17日"];
attributeSet.addedDate = [NSDate date];
CSSearchableItem *searchItem = [[CSSearchableItem alloc] initWithUniqueIdentifier:@"cat" domainIdentifier:@"com.rw" attributeSet:attributeSet];
[[CSSearchableIndex defaultSearchableIndex] indexSearchableItems:@[searchItem] completionHandler:^(NSError * _Nullable error) {
        NSLog(@"%@",error.description);
    }];
````
![890293.png](http://upload-images.jianshu.io/upload_images/6644906-b7eae5ac9f2e3757.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

* 电话

````Objective-C
attributeSet.supportsPhoneCall = @(YES);
attributeSet.phoneNumbers = @[@"15601856811"];
      attributeSet.accountHandles = @[@"15601856811”];
````
![119931.png](http://upload-images.jianshu.io/upload_images/6644906-84558af39aa45677.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
>注：supportsPhoneCall和phoneNumbers两个属性设置之后会显示一个电话图标，但是点击没有反应。只有设置accountHandles的时候，点击之后会拨打电话

* 更新

更新与增加是同样的方法，只是将Item的CSSearchableItemAttributeSet重新实例就好啦

* 删除
- deleteAllSearchableItemsWithCompletionHandler://删除所有的
- deleteSearchableItemsWithDomainIdentifiers:completionHandler://根据Domain删除
- deleteSearchableItemsWithIdentifiers:completionHandler://根据Identifiers删除

### Web Markup

如果你的应用程序的部分或全部内容也可以在你的网站上使用，你可以使用web标记来让用户访问你在搜索结果中的内容。使用web标记可以让Applebot web爬虫索引你在苹果服务器端索引中的内容，这使得它可以在Spotlight和Safari搜索结果中对所有iOS用户开放。

* [Smart App Banners](https://developer.apple.com/library/content/documentation/AppleApplications/Reference/SafariWebContent/PromotingAppswithAppBanners/PromotingAppswithAppBanners.html#//apple_ref/doc/uid/TP40002051-CH6)

在你的网站上，一个Smart App Banners邀请那些没有安装你的应用程序的用户从App Store下载它，它给已经安装了你的应用程序的用户安装了一个简单的方法，可以在它里面打开一个页面。如下图:

![image.png](http://upload-images.jianshu.io/upload_images/6644906-bb50d8ad936e580a.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

* [Support Universal Links](https://developer.apple.com/library/content/documentation/General/Conceptual/AppSearch/FindingImages.html#//apple_ref/doc/uid/TP40016308-CH14-SW1)

当你支持通用链接时，iOS用户可以点击链接到你的网站，并无缝地重定向到你安装的应用程序，而无需经过Safari。如果你的应用没有安装，点击一个链接到你的网站打开你的网站在Safari。

## 参考文章

[App Search Programming Guide](https://developer.apple.com/library/content/documentation/General/Conceptual/AppSearch/UniversalLinks.html#//apple_ref/doc/uid/TP40016308-CH12-SW1)
