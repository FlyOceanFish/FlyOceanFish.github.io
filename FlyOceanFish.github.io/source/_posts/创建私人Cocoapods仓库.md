---
title: 创建Cocoapods私人仓库 # 这是标题
date: 2017-10-25 17:27:00
categories:  # 这里写的分类会自动汇集到 categories 页面上，分类可以多级
- iOS # 一级分类
tags:   # 这里写的标签会自动汇集到 tags 页面上
- iOS
- CocoaPods
---
之前有一篇介绍了怎么创建私人项目[手把手教你创建podspec](http://flyoceanfish.top/2017/10/14/%E6%89%8B%E6%8A%8A%E6%89%8B%E6%95%99%E4%BD%A0%E5%88%9B%E5%BB%BACocoapods%E7%9A%84podspec/)，创建成功之后进行发布。最终发布
到Cocoapods官方仓库中。但是有时候我们创建完自己的pod项目之后，并不想公开发布，只限于自己或者公司使用，这时候就需要我们创建自己的私人仓库，将pod项目发布到自己的私人仓库中。

## 创建一个git私人项目
由于github只能创建公开项目，如果要创建私人项目是要花钱的，所以我这边使用的是[码市](https://coding.net)，免费最多能
创建5个私人项目，挺不错的。**这个私人项目是我们用来存储spec的总仓库**

## 创建podspec

* 创建spec工程

>pod lib create [名称]

这个与`pod spec create`的区别其实就是帮我们拉了一个模板，确实能省很多事情，里边包括了一个Example工程
* 再创建一个git私人项目

这个是用来存放我们Xcode工程和代码的。说白了就是托管我们代码的地方，主要是用来配饰spec中`s.source`用的

* 将我们要公开的代码和Example提交到git仓库

在第一步我们创建的工程中执行

> git add .

> git commit -m 'init'

> git remote add orgin [刚才我们创建仓库的地址]

> git push origin master

到这里为止已经将我们的代码提交到刚才创建的代码仓库中了。(如果没有权限则要配置.ssh中公钥私钥)
由于spec文件的原因我们其实还有一步没有做，其实就是打tag，只有打了tag才行,`:tag => s.version.to_s`就是这句话用到的
接下来我们继续打tag

> git tag 0.1.0

> git push --tags


## 编写xx.podspec文件

````
#
# Be sure to run `pod lib lint Utils.podspec' to ensure this is a
# valid spec before submitting.
#
# Any lines starting with a # are optional, but their use is encouraged
# To learn more about a Podspec see http://guides.cocoapods.org/syntax/podspec.html
#

Pod::Spec.new do |s|
  s.name             = 'Utils'
  s.version          = '0.1.0'
  s.summary          = 'Utils.'

# This description is used to generate tags and improve search results.
#   * Think: What does it do? Why did you write it? What is the focus?
#   * Try to keep it short, snappy and to the point.
#   * Write the description between the DESC delimiters below.
#   * Finally, don't worry about the indent, CocoaPods strips it!

  s.description      = <<-DESC
Utils封装常用组件
                       DESC

  s.homepage         = 'https://coding.net/u/FlyOceanFish/p/Utils'
  # s.screenshots     = 'www.example.com/screenshots_1', 'www.example.com/screenshots_2'
  s.license          = { :type => 'MIT', :file => 'LICENSE' }
  s.author           = { 'FlyOceanFish' => '978456068@126.com' }
  s.source           = { :git => 'https://git.coding.net/FlyOceanFish/Utils.git', :tag => s.version.to_s }
  # s.social_media_url = 'https://twitter.com/<TWITTER_USERNAME>'

  s.ios.deployment_target = '8.0'

  s.source_files = 'Utils/Classes/**/*'

  # s.resource_bundles = {
  #   'Utils' => ['Utils/Assets/*.png']
  # }

  # s.public_header_files = 'Pod/Classes/**/*.h'
    s.frameworks = 'UIKit', 'MapKit'
  # s.dependency 'AFNetworking', '~> 2.3'
end

````
这个是我写的，也可以参考我之前的一篇文章

## 检验spec文件的合法性

>pod spec lint

如果有什么错误，可以参照提示修改就行了

## 发布我们的podspec文件到我们的私人仓库

* 首先添加私人仓库源

这个私人仓库源其实就是我们第一步创建的那个仓库地址

> git repo

这个命令可以看到我们目前的私人仓库源

> git repo add [仓库源名称][仓库地址]

比如:

> git repo add FlyOceanSpecs https://git.coding.net/FlyOceanFish/FlyOceanSpecs.git

仓库源随便起名字，不过这个我们发布我们的spec项目的时候用的到。仓库地址就是我们
第一步创建的那个git仓库地址

* 发布到私人仓库

接上一步我们的FlyOceanSpecs仓库源

> git repo push FlyOceanSpecs Utils.podspec

## 总结
这个过程其实还是挺复杂的，大家如果有什么问题可以给我留言，我会及时回复的
