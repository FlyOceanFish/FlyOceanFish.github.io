---
title: 将RN工程嵌入到现有原生iOS应用 # 这是标题
categories:  # 这里写的分类会自动汇集到 categories 页面上，分类可以多级
- React Native # 一级分类
tags:   # 这里写的标签会自动汇集到 tags 页面上
- React Native # 可配置多个标签，注意格式
- iOS
---
今天心血来潮，就想尝试一下将RN工程单独嵌入到原生工程中，所以就做了尝试，本文是通过cocopods集成RN到现有工程的，但是其中也遇到一个问题，怎么编译都不过。
#依赖包
React Native的植入过程同时需要React和React Native两个node依赖包，所以需要我们创建package.json文件和正确的RN文件目录结构
> 对于一个典型的React Native项目来说，一般package.json和index.ios.js等文件会放在项目的根目录下。而iOS相关的原生代码会放在一个名为ios/的子目录中,这里也同时放着你的Xcode项目文件（.xcodeproj）。

![88161340-E7F4-46B0-86A9-5A13DCCDA819.png](http://upload-images.jianshu.io/upload_images/6644906-30b409760b9414ae.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

按照上图的目录结构创建
###package.json
````json
{
	"name": "TestReactNative",
	"version": "0.0.1",
	"private": true,
	"scripts": {
		"start": "node node_modules/react-native/local-cli/cli.js start",
		"test": "jest"
	},
	"dependencies": {
		"react": "16.0.0-alpha.6",
		"react-native": "0.44.3"
	},
	"devDependencies": {
		"babel-jest": "20.0.3",
		"babel-preset-react-native": "2.1.0",
		"jest": "20.0.4",
		"react-test-renderer": "16.0.0-alpha.6"
	},
	"jest": {
		"preset": "react-native"
	}
}
````
#安装依赖包
使用npm（node包管理器，Node package manager）来安装React和React Native模块。这些模块会被安装到项目根目录下的node_modules/目录中。 在包含有package.json文件的目录（一般也就是项目根目录）中运行下列命令来安装：
>npm install

#新建index.ios.js
这个是随便一个index.ios.js即可,我是从一个RN的初始化工程复制了一个出来
````javascrip
/**
 * Sample React Native App
 * https://github.com/facebook/react-native
 * @flow
 */

import React, { Component } from 'react';
import {
  AppRegistry,
  StyleSheet,
  Text,
  View
} from 'react-native';

export default class DemoApp extends Component {
  render() {
    return (
      <View style={styles.container}>
        <Text style={styles.welcome}>
          Welcome to React Native!
        </Text>
        <Text style={styles.instructions}>
          To get started, edit index.ios.js
        </Text>
        <Text style={styles.instructions}>
          Press Cmd+R to reload,{'\n'}
          Cmd+D or shake for dev menu
        </Text>
      </View>
    );
  }
}

const styles = StyleSheet.create({
  container: {
    flex: 1,
    justifyContent: 'center',
    alignItems: 'center',
    backgroundColor: '#F5FCFF',
  },
  welcome: {
    fontSize: 20,
    textAlign: 'center',
    margin: 10,
  },
  instructions: {
    textAlign: 'center',
    color: '#333333',
    marginBottom: 5,
  },
});

AppRegistry.registerComponent('TestReactNative', () => DemoApp);
````
至此RN的相关依赖已经配置好，接下来就该配置cocopods了
#cocopods配置
1、***创建Podfile，如果已经有了可以忽略此步骤***
```` jsp
##在iOS原生代码所在的目录中（也就是`.xcodeproj`文件所在的目录）执行：
$ pod init
````
Podfile文件内容如下:
````jsp
# Uncomment the next line to define a global platform for your project
# platform :ios, '9.0'

target 'TestReactNative' do
  # Uncomment the next line if you're using Swift or would like to use dynamic frameworks
  # use_frameworks!

  # Pods for TestReactNative
  pod 'React', :path => '../node_modules/react-native', :subspecs => [
    'Core',
    'DevSupport', # 如果RN版本 >= 0.43，则需要加入此行才能开启开发者菜单
    'RCTText',
    'RCTNetwork',
    'RCTWebSocket', # 这个模块是用于调试功能的
    # 在这里继续添加你所需要的模块
  ]
  # 如果你的RN版本 >= 0.42.0，请加入下面这行
  pod "Yoga", :path => "../node_modules/react-native/ReactCommon/yoga"
end
````
subspecs就是你依赖的包。可用的subspec都列在[node_modules/react-native/React.podspec
](https://github.com/facebook/react-native/blob/master/React.podspec)中，基本都是按其功能命名的。一般来说你首先需要添加Core，这一subspec包含了必须的AppRegistry、StyleSheet、View以及其他的一些React Native核心库。如果你想使用React Native的Text库（即<Text>组件），那就需要添加RCTText的subspec。同理，Image需加入RCTImage，等等。
2、***安装依赖包***
执行pod install
![C07B6B3E-1642-4EAC-9814-BE6A55C4527C.png](http://upload-images.jianshu.io/upload_images/6644906-d2f8ac5ea7e08d21.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
会看到以上过程则表示安装依赖已经成功
3、***调用RN工程***
````objective-c
#import "ViewController.h"
#import <React/RCTRootView.h>

@interface ViewController ()

@end

@implementation ViewController

- (void)viewDidLoad {
    [super viewDidLoad];
    // Do any additional setup after loading the view, typically from a nib.
}

- (IBAction)actionClick:(id)sender {
    NSLog(@"High Score Button Pressed");
    NSURL *jsCodeLocation = [NSURL
                             URLWithString:@"http://localhost:8081/index.ios.bundle?platform=ios"];
    RCTRootView *rootView =
    [[RCTRootView alloc] initWithBundleURL : jsCodeLocation
                         moduleName        : @"TestReactNative"
                         initialProperties :nil
                          launchOptions    : nil];
    UIViewController *vc = [[UIViewController alloc] init];
    vc.view = rootView;
    [self presentViewController:vc animated:YES completion:nil];
}

- (void)didReceiveMemoryWarning {
    [super didReceiveMemoryWarning];
    // Dispose of any resources that can be recreated.
}
@end
````
至此整集成已经结束，可以直接运行xcode过程，或者通过命令运行工程
````jsp
# From the root of your project, where the `node_modules` directory is located.
$ npm start
````
````jsp
# 在项目的根目录中执行：
$ react-native run-ios
````
#坑
不知道大家有没有注意我们package.json中react-native中的版本是"0.44.3"，而为什么不是最新版46的呢？？这就是一个很大的坑！从45版本开始RN有4个文件被墙了，所以如果大于44版本的react-native会一直编译不过
***有以下3个解决方案：***
1、不导入最新的包，导入44版本
2、翻墙呗。。。。
3、[下载被墙的4个依赖包，手工导入](http://blog.csdn.net/u013751625/article/details/75046147)
###优化加载速度
在原生调用RN的时候有点慢。大家在集成的时候会发现加载的过程挺慢的，可以压缩加载本地文件，这样加载就快了。通常我们可以将文件放到服务器上，然后通过IP地址来加载，这种方式其实也是比较慢的，最好的方式就是将文件提前下载到本地，通过一定的更新机制确保本地文件时最新的，通过使用这种方式加载就很快了。
- 到当前项目的根目录中执行:
````
react-native bundle --platform ios --dev false --entry-file index.ios.js --bundle-output ios/main.jsbundle
````
- 将main.jsbundle加载到Xcode工程中，然后通过以下方式获取URL地址来加载
````
NSString *path = [[NSBundle mainBundle] pathForResource:@"main" ofType:@"jsbundle"];
NSURL *jsCodeLocation = [NSURL URLWithString:path];
````

如果大家在这个过程还有不明白的可以留言给我，我将尽最大努力给大家解决的哦
