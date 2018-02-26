---
title: iOS11文件存储最佳实践 # 这是标题
date: 2018-2-26 14:24:00
categories:  # 这里写的分类会自动汇集到 categories 页面上，分类可以多级
- iOS # 一级分类
tags:   # 这里写的标签会自动汇集到 tags 页面上
- iOS
---
众所周知，iOS11之前的版本我们无法看到应用里边的任何文件，这主要是由于iOS的沙盒机制导致的。在我们应用的沙盒里边，有三个文件夹供我们使用来存储文件。分别是`Documents`、`Library`、`tmp`。
* Documents
您应该将所有的应用程序数据文件写入到这个目录下。这个目录用于存储用户数据。该路径可通过配置实现iTunes共享文件。可被iTunes备份。
* Library
Preferences 目录：包含应用程序的偏好设置文件。您不应该直接创建偏好设置文件，而是应该使用NSUserDefaults类来取得和设置应用程序的偏好.
Caches 目录：用于存放应用程序专用的支持文件，保存应用程序再次启动过程中需要的信息。
可创建子文件夹。可以用来放置您希望被备份但不希望被用户看到的数据。该路径下的文件夹，除Caches以外，都会被iTunes备份。
* tmp
这个目录用于存放临时文件，保存应用程序再次启动过程中不需要的信息。该路径下的文件不会被iTunes备份。

不过由于iOS11系统中增加了一个`Files`的应用，导致`Documents`和`Library`有些细微的变化。
# Files 是什么
Files 可以集中管理iOS上应用内创建的文件，以及各个云盘服务中保存的文件。
# Files使用
Files会自动整合应用内公开的所有文件，其实就是`Documents`文件夹中所有的文件。但是有时候我们存储的用户的数据不想让用户看到，这时应该怎么办呢？这时候我们可以将数据存储到`ApplicationSupport`文件夹。这个文件夹与`Documents`基本一样，除了一个能被用户看到，一个是隐藏的。
## 实践
我们新建一个工程，然后在Info.plist中增加Application supports iTunes file sharing和Supports opening documents in place这两个选项，并且设置为YES
![1519629565974.jpg](http://upload-images.jianshu.io/upload_images/6644906-82d02b690d367496.jpg?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
>Application supports iTunes file sharing 可以通过iTunes操作手机上APP的`Documents`中的内容，实现了文件的共享

加上了这两个值之后，此时应用的`Documents`文件夹中的内容就可以通过Files应用看到了。
![WechatIMG31.jpeg](http://upload-images.jianshu.io/upload_images/6644906-acc90036799b6de0.jpeg?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
![WechatIMG32.jpeg](http://upload-images.jianshu.io/upload_images/6644906-23b89a672d9d7523.jpeg?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
## 设置Documents或Library文件夹下的文件不被备份至iCloud
```Objective-C
- (BOOL)addSkipBackupAttributeToItemAtURL:(NSURL *)URL
{
    assert([[NSFileManager defaultManager] fileExistsAtPath: [URL path]]);

    NSError *error = nil;
    BOOL success = [URL setResourceValue:[NSNumber numberWithBool: YES]
                                  forKey: NSURLIsExcludedFromBackupKey error: &error];
    if(!success){
        NSLog(@"Error excluding %@ from backup %@", [URL lastPathComponent], error);
    }

    return success;
}

```
## 检查可用空间大小
主要用于当我们下载某个文件到手机的时候，如果此时手机的存储空间不够用，可以先提示用户。否则不会下载，增加用户的体验
```Objective-C
NSURL *fileURL = [[NSURL alloc] initFileURLWithPath:NSTemporaryDirectory()];
NSError *error = nil;
NSDictionary *results = [fileURL resourceValuesForKeys:@[NSURLVolumeAvailableCapacityForImportantUsageKey] error:&error];
if (!results) {
    NSLog(@"Error retrieving resource keys: %@\n%@", [error localizedDescription], [error userInfo]);
    abort();
}
NSLog(@"Available capacity for important usage: %@", results);

```
* NSURLVolumeAvailableCapacityKey 获取到整个手机剩余的可用空间
* NSURLVolumeTotalCapacityKey 获取到整个手机的存储空间，比如32G的手机获取的数据时32G
* NSURLVolumeAvailableCapacityForImportantUsageKey和NSURLVolumeAvailableCapacityForOpportunisticUsageKey 第一个获取的空间比第二个获取到的大。具体怎么计算的不知道，NSURLVolumeAvailableCapacityForImportantUsageKey与NSURLVolumeAvailableCapacityKey获取的值比较接近；NSURLVolumeAvailableCapacityForOpportunisticUsageKey获取到的值比NSURLVolumeAvailableCapacityKey小一些。作者猜想可能这几个枚举与沙河里边的文件夹相对应吧。（如果有知道具体怎么回事的可以留言哦）
