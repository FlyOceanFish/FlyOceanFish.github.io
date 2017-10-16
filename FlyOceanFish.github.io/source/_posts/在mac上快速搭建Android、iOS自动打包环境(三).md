---
title: 在mac上快速搭建Android、iOS自动打包环境(三) # 这是标题
date: 2017-10-14 14:27:03
categories:  # 这里写的分类会自动汇集到 categories 页面上，分类可以多级
- iOS # 一级分类
tags:   # 这里写的标签会自动汇集到 tags 页面上
- iOS
---
接上一篇iOS自动打包文章，本文将介绍Android的自动打包。
# 准备工作
　　Android的SDK已经安装好（这个网上教程一大堆，如果不会的可以自行搜索，实在搞不定的可以留言）

# Android自动打包
　　其实Android的自动打包与iOS的自动打包基本一样，主要由于开发工具的不同，所以不同点都是由于开发工具不同造成。
***本文将写出与iOS的不同的地方，前边几步相同的就省略了***
1、 ***配置gradle***
* 找到“构建”->“增加构建步骤”->“Excute Shell”。这一步由于服务器上的gradlew一直获取不到权限
![4ADDE352-E751-4379-8BA2-2402F36ECB5C.png](http://upload-images.jianshu.io/upload_images/6644906-39e67a7b9138b591.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
* 找到“构建”->“增加构建步骤”->“Invoke Gradle Script”，选择"Use Gradle Wrapper"。其中Build File 代表build.gradle的路径
![83CAA3F9-9675-46DE-9332-A8B3C4D579C0.png](http://upload-images.jianshu.io/upload_images/6644906-c37c70459c0bee8d.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

2、配置上传apk脚本
      curl -F "file=@app/build/outputs/apk/app-debug.apk" -F "uKey=4acbf7559e513f1f6081294723ea4d0f" -F "_api_key=d9e87e9febbf20dd4f384876bdf72a22" https://qiniu-storage.pgyer.com/apiv1/app/upload

3、如果构建的时候报找不到“ ANDROID_HOME”，则就在“系统设置”->“全局属性”中增加，然后“系统管理”->“读取设置”重新启动Jenkins。
![A658B2DA-D10F-484F-8782-0F77210CC615.png](http://upload-images.jianshu.io/upload_images/6644906-b7b42037ef20691c.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

　　很庆幸自己坚持了这么久了，已经把《在mac上快速搭建Android、iOS自动打包环境》这一个系列完成。感性简友的支持。如果自己在搭建过程中有任何问题可以联系我。
