---
title: 手把手教你创建Cocoapods的podspec # 这是标题
date: 2017-10-14 09:27:00
categories:  # 这里写的分类会自动汇集到 categories 页面上，分类可以多级
- iOS # 一级分类
tags:   # 这里写的标签会自动汇集到 tags 页面上
- iOS
---
现在大部分iOS项目都是在用Cocoapods来管理项目了，但是当我们自己写的一个公有或者私有项目的时候，想要支持Cocoapods，那应该怎么做呢？本教程手把手的教大家如何创建podsepc
## 步骤
1. github上创建自己的共有项目
![2220EA18-9E12-4891-B409-EB99776CBA78.png](http://upload-images.jianshu.io/upload_images/6644906-af82c503463a2495.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
2.  创建自己的Xcode工程，并提交到该repository
![0EF5DF0F-67C1-41D4-B3BA-5F8A8BB0BE78.png](http://upload-images.jianshu.io/upload_images/6644906-b677f02da468d7cb.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
3. 创建podspec文件
我们cd当我们项目的根目录下。本项目中就是上图的文件夹下。执行如下命令：

       pod spec create OceanFish git@github.com:FlyOceanFish/CreateOwnSpec.git
4. 编辑我们的podspec文件

````
#
#  Be sure to run `pod spec lint OceanFish.podspec' to ensure this is a
#  valid spec and to remove all comments including this before submitting the spec.
#
#  To learn more about Podspec attributes see http://docs.cocoapods.org/specification.html
#  To see working Podspecs in the CocoaPods repo see https://github.com/CocoaPods/Specs/
#

Pod::Spec.new do |s|

  # ―――  Spec Metadata  ―――――――――――――――――――――――――――――――――――――――――――――――――――――――――― #
  #
  #  These will help people to find your library, and whilst it
  #  can feel like a chore to fill in it's definitely to your advantage. The
  #  summary should be tweet-length, and the description more in depth.
  #

  s.name         = "OceanFish"
  s.version      = "0.0.1"
  s.summary      = "make a Spec"

  # This description is used to generate tags and improve search results.
  #   * Think: What does it do? Why did you write it? What is the focus?
  #   * Try to keep it short, snappy and to the point.
  #   * Write the description between the DESC delimiters below.
  #   * Finally, don't worry about the indent, CocoaPods strips it!
  s.description  = <<-DESC
                  手把手叫你如何创建自己的Spec
                   DESC

  s.homepage     = "https://github.com/FlyOceanFish"
  # s.screenshots  = "www.example.com/screenshots_1.gif", "www.example.com/screenshots_2.gif"


  # ―――  Spec License  ――――――――――――――――――――――――――――――――――――――――――――――――――――――――――― #
  #
  #  Licensing your code is important. See http://choosealicense.com for more info.
  #  CocoaPods will detect a license file if there is a named LICENSE*
  #  Popular ones are 'MIT', 'BSD' and 'Apache License, Version 2.0'.
  #

  s.license      = { :type => "MIT", :file => "LICENSE" }


  # ――― Author Metadata  ――――――――――――――――――――――――――――――――――――――――――――――――――――――――― #
  #
  #  Specify the authors of the library, with email addresses. Email addresses
  #  of the authors are extracted from the SCM log. E.g. $ git log. CocoaPods also
  #  accepts just a name if you'd rather not provide an email address.
  #
  #  Specify a social_media_url where others can refer to, for example a twitter
  #  profile URL.
  #

  s.author             = { "OceanFish" => "978456068@qq.com" }
  # Or just: s.author    = "frances"
  # s.authors            = { "frances" => "978456068@qq.com" }
  # s.social_media_url   = "http://twitter.com/frances"

  # ――― Platform Specifics ――――――――――――――――――――――――――――――――――――――――――――――――――――――― #
  #
  #  If this Pod runs only on iOS or OS X, then specify the platform and
  #  the deployment target. You can optionally include the target after the platform.
  #

  # s.platform     = :ios
   s.platform     = :ios, "10.3"

  #  When using multiple platforms
  # s.ios.deployment_target = "5.0"
  # s.osx.deployment_target = "10.7"
  # s.watchos.deployment_target = "2.0"
  # s.tvos.deployment_target = "9.0"


  # ――― Source Location ―――――――――――――――――――――――――――――――――――――――――――――――――――――――――― #
  #
  #  Specify the location from where the source should be retrieved.
  #  Supports git, hg, bzr, svn and HTTP.
  #

  s.source       = { :git => "https://github.com/FlyOceanFish/CreateOwnSpec.git", :tag => "#{s.version}" }


  # ――― Source Code ―――――――――――――――――――――――――――――――――――――――――――――――――――――――――――――― #
  #
  #  CocoaPods is smart about how it includes source code. For source files
  #  giving a folder will include any swift, h, m, mm, c & cpp files.
  #  For header files it will include any header in the folder.
  #  Not including the public_header_files will make all headers public.
  #

  s.source_files  = "FlyOceanFish", "FlyOceanFish/**/*.{h,m}"
  #s.exclude_files = "Classes/Exclude"

  # s.public_header_files = "Classes/**/*.h"


  # ――― Resources ―――――――――――――――――――――――――――――――――――――――――――――――――――――――――――――――― #
  #
  #  A list of resources included with the Pod. These are copied into the
  #  target bundle with a build phase script. Anything else will be cleaned.
  #  You can preserve files from being cleaned, please don't preserve
  #  non-essential files like tests, examples and documentation.
  #

  # s.resource  = "icon.png"
  # s.resources = "Resources/*.png"

  # s.preserve_paths = "FilesToSave", "MoreFilesToSave"


  # ――― Project Linking ―――――――――――――――――――――――――――――――――――――――――――――――――――――――――― #
  #
  #  Link your library with frameworks, or libraries. Libraries do not include
  #  the lib prefix of their name.
  #

  # s.framework  = "SomeFramework"
  # s.frameworks = "SomeFramework", "AnotherFramework"

  # s.library   = "iconv"
  # s.libraries = "iconv", "xml2"


  # ――― Project Settings ――――――――――――――――――――――――――――――――――――――――――――――――――――――――― #
  #
  #  If your library depends on compiler flags you can set them in the xcconfig hash
  #  where they will only apply to your library. If you depend on other Podspecs
  #  you can include multiple dependencies to ensure it works.

  s.requires_arc = true

  # s.xcconfig = { "HEADER_SEARCH_PATHS" => "$(SDKROOT)/usr/include/libxml2" }
  # s.dependency "JSONKit", "~> 1.4"

end

````
5.  执行如下命令进行校验文件的正确性

        pod lib lint

![AECADD88-FAD7-4B9C-BD73-701B7AC40282.png](http://upload-images.jianshu.io/upload_images/6644906-8ee99362e9e96299.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
如果见到如上图，则代表验证通过；如果没有通过的话按照提示进行修改即可。
备注：这个命令是本地校验的，还要pod spec lint 这个是本地和远程校验。如果此时我们用pod spec lint会报错，因为我们还有发布一个版本，就是第6步还没做呢。
6. 在github上创建一个发布版本
![3EF1084F-7DF7-403E-8CE7-519AB42194E0.png](http://upload-images.jianshu.io/upload_images/6644906-d8d193f003b2720b.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
此时我们运行pod spec lint则会见到如下图
![EDBCCFB9-6125-4816-8CEE-DD7178C06CB0.png](http://upload-images.jianshu.io/upload_images/6644906-e7ff39a2608266f9.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
到这步整个制作过程已经完成接下来让我们发布自己spec
## 发布到CocoaPods
CocoaPods 0.33中加入了 Trunk 服务，使用 Trunk 服务可以方便的发布自己的Pod。要想使用 Trunk 服务，首先需要使用如下命令注册自己的电脑。这很简单，只要你指明你的邮箱地址（spec文件中的）和名称即可。CocoaPods 会给你填写的邮箱发送验证邮件，点击邮件中的链接就可通过验证。
1. 注册
    pod trunk register 978456068@qq.com "FlyOceanFish"

2、验证是否注册成功

    pod trunk me

![7117B254-7688-427F-A205-69968ABF3AE3.png](http://upload-images.jianshu.io/upload_images/6644906-1c0e9356bacb2005.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
3、发布我们的podspec

    pod trunk push
![7FA508FC-B6CB-4ED1-B31E-1799F83BCE6F.png](http://upload-images.jianshu.io/upload_images/6644906-870e654c65f46478.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
此图代表我们已经发布成功

不过此时我们pod search还搜不到我们的podspec。

可以通过一下两步之后再尝试搜索:
* 运行pod setup更新本地的spec，再搜索一下试试看。
* 删除~/Library/Caches/CocoaPods目录下的search_index.json文件
>pod setup成功后会生成~/Library/Caches/CocoaPods/search_index.json文件。
终端输入rm ~/Library/Caches/CocoaPods/search_index.json
删除成功后再执行pod search

## 参考链接
[CocoaPods](https://guides.cocoapods.org/using/index.html)
[between 'pod spec lint' and 'pod lib lint'?](https://stackoverflow.com/questions/32304421/whats-the-difference-between-pod-spec-lint-and-pod-lib-lint)
