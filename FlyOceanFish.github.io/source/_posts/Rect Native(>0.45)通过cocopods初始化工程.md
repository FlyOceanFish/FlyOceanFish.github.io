---
title: Rect Native(>0.45)通过cocopods初始化工程 # 这是标题
categories:  # 这里写的分类会自动汇集到 categories 页面上，分类可以多级
- React Native # 一级分类
tags:   # 这里写的标签会自动汇集到 tags 页面上
- React Native # 可配置多个标签，注意格式
- iOS
---
由于React Native 0.45以上版本增加了4个依赖包，并且被墙了，所以大家在初始化工程之后运行会报错，就是因为缺少了文件。其实就是Bridge文件或者它所依赖的文件。0.45以后的bidge开始使用了RCTCxxBridge,之前都是RCTBatchedBridge，根据Facebook的官方资料RCTBatchedBridge慢慢会被废弃掉的，建议使用RCTCxxBridge。因为使用RCTCxxBridge我们就必须导入被墙的4个依赖包，就会点繁琐。使用RCTBatchedBridge比较简单不用翻墙的。一下是两个Bridge对应的不同Podfile。

### RCTBatchedBridge对应的Podfile文件内容
````
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
    'BatchedBridge'//这个是重点，在ReactNative提供的官方例子中没导入，所以按照官方文档去做初始化的工程会报错
    # 在这里继续添加你所需要的模块
  ]
  # 如果你的RN版本 >= 0.42.0，请加入下面这行
  pod "Yoga", :path => "../node_modules/react-native/ReactCommon/yoga"
end

````
### RCTCxxBridge对应的Podfile文件内容
````
# Uncomment the next line to define a global platform for your project
# platform :ios, '9.0'
react_native_path = '../node_modules/react-native'

target 'StudyReactNative' do
  # Uncomment the next line if you're using Swift or would like to use dynamic frameworks
  use_frameworks!

  # Native Navigation!
  #pod 'native-navigation', :path => '../node_modules/native-navigation'

  # To use CocoaPods with React Native, you need to add this specific Yoga spec as well
  pod 'Yoga', :path => react_native_path + '/ReactCommon/yoga'

  # Third party deps
  pod 'DoubleConversion', :podspec => react_native_path + '/third-party-podspecs/DoubleConversion.podspec'
  pod 'GLog', :podspec => react_native_path + '/third-party-podspecs/GLog.podspec'
  pod 'Folly', :podspec => react_native_path + '/third-party-podspecs/Folly.podspec'

  # You don't necessarily need all of these subspecs, but this would be a typical setup.
  pod 'React', :path => react_native_path, :subspecs => [
    'Core',
    'CxxBridge',
    'RCTText',
    'RCTNetwork',
    'RCTWebSocket', # needed for debugging
    'RCTImage',
    'RCTNetwork',
    # Add any other subspecs you want to use in your project
    'DevSupport'
  ]
end

````
这里因为导入了四个包，被墙了所以要翻墙。

四个包的地址
````
fetch_and_unpack glog-0.3.4.tar.gz https://github.com/google/glog/archive/v0.3.4.tar.gz "\"$SCRIPTDIR/ios-configure-glog.sh\""
fetch_and_unpack double-conversion-1.1.5.tar.gz https://github.com/google/double-conversion/archive/v1.1.5.tar.gz
fetch_and_unpack boost_1_63_0.tar.gz https://github.com/react-native-community/boost-for-react-native/releases/download/v1.63.0-0/boost_1_63_0.tar.gz
fetch_and_unpack folly-2016.09.26.00.tar.gz https://github.com/facebook/folly/archive/v2016.09.26.00.tar.gz
````
其实可以通过上边地址自己下载下来放到
XXReactNative工程/node_modules/react-native/third-party文件夹下
如果没有third-party这个文件夹就新建一个。

也可以通过百度云盘[third-party](https://pan.baidu.com/s/1kVDUAZ9#list/path=%2F)下载下来解压缩之后放到XXReactNative工程/node_modules/react-native这个路径下就可以了
之后运行pod install即可
![7DC30CAE-BDB1-439A-810E-1BE35FCC8EA4.png](http://upload-images.jianshu.io/upload_images/6644906-01d13bd37248e3e4.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

##更新以前的工程
我通过以上方法来更新我以前的工程，同时替换package.json为最新的，发现报错。此时需要我们yarn  install或npm install，然后在ios工程的根目录下执行pod install

##‘boost/iterator/iterator_adaptor.hpp’ file not found

* /Users/Vanessa/.rncache 中 boost_1_63_0.tar.gz， double-conversion-1.1.5.tar.gz， folly-2016.09.26.00.tar.gz， glog-0.3.4.tar.gz 文件下载不完整

* node_modules/react-native/third-party 文件不完整

解决方案：

删除 .rncache 后重新下载，或手动下载后放入 .rncache 中

把以上文件解压后放入 node_modules/react-native/third-party 下

Clean & Build
