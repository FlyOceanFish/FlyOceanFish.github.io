---
title: 在mac上快速搭建Android、iOS自动打包环境(一) # 这是标题
categories:  # 这里写的分类会自动汇集到 categories 页面上，分类可以多级
- iOS # 一级分类
tags:   # 这里写的标签会自动汇集到 tags 页面上
- iOS
---
本文实现了jenkins+fastlane自动打包Android、iOS，然后上传到蒲公英上，最后通过邮件通知相关人员
# 工具
* [jenkins](https://jenkins.io)
* [fastlane](https://github.com/fastlane/fastlane)
* Homebrew

# 步骤
1、 安装brew，如果电脑已经安装，可以忽略
> curl -LsSf http://github.com/mxcl/homebrew/tarball/master | sudo tar xvz -C/usr/local --strip 1

2、 安装java环境
> brew cask install java

3、安装jenkins
> brew install jenkins

4、设置端口
> java -jar /usr/local/Cellar/jenkins/2.67/libexec/jenkins.war --httpPort=8080

5、启动jenkins
> jenkins

到这步jenkins已经安装完毕，在浏览器中输入http://localhost:8080 即可出现以下界面:

![4046DB34-2FB9-481F-A81C-2B3924F33C72.png](http://upload-images.jianshu.io/upload_images/6644906-e7bd74cf75b6b438.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
6、在终端命令行输入以下命令获取密码，填入输入框之后点击continue
> cat /Users/wrf/.jenkins/secrets/initialAdminPassword
7、之后出现以下界面，点击install suggested plugin。这步操作安装的插件基本包括了常规使用的全部

![7FCE5BB6-C6D6-4DD6-B60C-510523C6BA33.png](http://upload-images.jianshu.io/upload_images/6644906-4bb3b4f4d3d86e8c.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

8、之后会看见如下界面，如果出现红叉的代表安装失败，只要点击retry即可，基本都会安装成功。之后创建账号就行了。整个jenkins已经安装成功

![E1BFCF45-4DCC-44DD-B30E-18674C91098A.png](http://upload-images.jianshu.io/upload_images/6644906-673e05f3f6de2730.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
9、fastlane安装
> brew cask install fastlane

　　至此整个环境已经完全搭好，第二篇文章将详细介绍android、iOS的打包流程。

#*遇到的坑*

　　在这个安装过程中遇到一个坑，研究了好久好不容易解决了。如果通过jenkins官方下载的.dmg文件安装，发现打包都是超时，打包不成功。通过命令行安装的jenkins能正确打包。

       ERROR: Timeout after 10 minutes
       ERROR: Error fetching remote repo 'origin'
　　研究之后主要原因其实是这两种方式安装成功之后的安装路径不一样，一个是在shared路径下，一个是在/User/用户名这个路径下
