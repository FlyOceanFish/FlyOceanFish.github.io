---
title: iOS文件擦除 # 这是标题
date: 2017-11-02 09:00:00
categories:  # 这里写的分类会自动汇集到 categories 页面上，分类可以多级
- iOS # 一级分类
tags:   # 这里写的标签会自动汇集到 tags 页面上
- iOS
- 安全
---
## 文件擦除

为了完全擦除文件，必须将文件打开，并且每个字节都被覆盖。

* C语言编写的删除 安全等级很高。
* Objective-C编写的 可能被攻击者劫持方法的实现

## SQL记录擦除

在iOS手机上只要SQLite数据库文件本身没有被删除，数据库中被删除的数据记录将会被保留在未被分配的存储空间里，直到新记录覆盖它们。

auto_vacuum命令删除的数据还有可能被恢复，通过VACUUM命令重建整个数据库，可以彻底删除。但是执行时间较长，会造成额外的性能损耗。

一个更好的方式是通过SQL的UPDATE函数替代DELEATE，对要删除的数据用其他数据覆盖。（用来覆盖的数据的长度一定要大于被覆盖的数据）

## 键盘缓存

* 密码框中的数据不会缓存
* 纯数字的字符串不会缓存。像信用卡之类的是安全的
* 禁用文本框的自动更正功能可以防止缓存
* 太短的一两个字符的数据不会被缓存
* 文本的复制会被缓存，所以可以通过禁用文本框的复制/粘贴功能可以防止缓存

```Objective-C
textField.autocorrectionType = UITextAutocorrectionTypeNo;//禁用自动纠正功能
textField.secureTextEntry = YES;//密码
```
## 应用程序快照
当一个应用在后台挂起时，会有一个当前内容的屏幕快照。这样做是为了让用户感觉到程序从后台唤起无缝恢复

```Objective-C
- (void)applicationWillResignActive:(UIApplication *)application {
    [UIApplication sharedApplication].keyWindow.hidden = YES;
}

- (void)applicationDidEnterBackground:(UIApplication *)application {
    [UIApplication sharedApplication].keyWindow.hidden = YES;
}
```
这样就可以做到屏幕快照的时候是一个空白的页面
