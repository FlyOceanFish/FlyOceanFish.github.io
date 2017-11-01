---
title: 通过Fastlane自动化发布Spec # 这是标题
date: 2017-10-31 09:40:00
categories:  # 这里写的分类会自动汇集到 categories 页面上，分类可以多级
- iOS # 一级分类
tags:   # 这里写的标签会自动汇集到 tags 页面上
- iOS
- Fastlane
---
在程序的世界里，很多程序的产生其实就是由于作者本身'懒'的原因，
所以就做了一个程序，方便了自己也方便了大家。在做组件化的时候，
大家肯定遇到过繁琐的发布spec。既要提交git又要发布到自己的私
有仓库，通过命令来实现也有很多条，那有没有发布只要执行一条语句
，就能帮我们自动执行我们想要的所有语句呢？答案肯定是有的！这就
需要fastlane来帮忙了。一个自动化工具，可以帮忙打包Android或
iOS的不同渠道包等。接下来就让我们看看通过fastlane怎么实现自
动发布我们的lib到自己的私有仓库。
## fastlane安装
>brew cask install fastlane

安装很简单只要通过这一条命令即可，如果没有安装brew的话，可以
自行百度安装即可

>fastlane init
通过执行这个命令可以在我们根目录中创建一个fastlane文件夹，里边有
AppFile、Fastfile等文件，这个是打包ipa包用的。所以这个命令在
这里我们不需要执行。我们只要了解一下它的目录结构

## Fastfile文件准备
如果我们没有执行`fastlane init`命令，此时需要我们在根目录中创建
一个文件夹fastlane，然后在文件夹下创建一个`Fastfile`文件

如下图的目录结构:

![5BC4EE94-95EA-4E97-9151-721FB2D19C5C.png](http://upload-images.jianshu.io/upload_images/6644906-dd339fee102b460d.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

## Fastfile文件的编写

这个文件其实就是将Fastlane官方已经做好的有关Action放在一起，即可实现我们想要的
功能。[Actions](https://docs.fastlane.tools/actions/)

````
desc "发布Spec自动化通过YTOSpec命令"
lane :FOFSpecRelease do |options|
  tag_version = options[:tag]
  target = options[:target]
  repoName = options[:repo]
  #pod install
  cocoapods(
    clean: true,
    podfile: "./Example/#{target}"
  )
  #pod lib lint
  pod_lib_lint(verbose: true,allow_warnings: true)
  version_bump_podspec(path: "#{target}.podspec", version_number: "#{tag_version}")
  #1、git add .
  git_add
  #2、git commit
  git_commit(path: ".", message: "升级")
  #3、git push

  #4、git tag
  add_git_tag(
    tag: "#{tag_version}"
)
  push_to_git_remote(
  remote: "origin",         # optional, default: "origin"
  local_branch: "master",  # optional, aliased by "branch", default: "master"
  remote_branch: "master", # optional, default is set to local_branch
  force: false,    # optional, default: false
  tags: true     # optional, default: true
  )
  pod_push(path: "#{target}.podspec", repo: "#{repoName}",allow_warnings: true)
end
````
上边这些命令在官网都有相应的解释，而且官网还有其他众多强大的Action
现在实现的其实还是有点有问题的。就是我们在打tag的时候如果tag已经存在
我们则要手工删除tag才能继续进行。判断tag是否存在已经有对应的Action
但是木有删除tag的Action，所以此时我们需要自定义一个Action才行
