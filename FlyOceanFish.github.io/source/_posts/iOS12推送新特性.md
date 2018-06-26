---
title: iOS12中推送通知新特性 # 这是标题
date: 2018-06-26 13:55:00
categories:  # 这里写的分类会自动汇集到 categories 页面上，分类可以多级
- iOS # 一级分类
tags:   # 这里写的标签会自动汇集到 tags 页面上
- iOS
- WWDC18
---
# 序言
众所周知，iOS中消息推送扮演了不可或缺的位置。不管是本地通知还是远程通知无时不刻的在影响着我们的用户体验，以致于在iOS10的时候苹果对推送大规模重构，独立了已`UserNotifications`和`UserNotificationsUI`两个单独的framework，可见重要性一斑。针对于WWDC18苹果又给我们带来了什么惊喜呢？
# 新特性
* Grouped notifications
推送分组
* Notification content extensions
推送内容扩展中的可交互和动态更改Action
* Notification management
推送消息的管理
* Provisional authorization
临时授权
* Critical alerts
警告性质的推送
## 推送分组

随着手机上应用的增多，尤其QQ和微信这两大聊天工具，当手机锁屏的时候，伴随着就是好满屏的推送消息。这一现象不知大家有没有觉着不高效和体验性比较差呢？苹果针对锁屏情况下，对消息进行了分组，从而有效的提高了用户的交互体验，分组形式如下：

![1529981248132.jpg](https://upload-images.jianshu.io/upload_images/6644906-44a8e42bb655ad02.jpg?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

### 分组形式：

* 苹果会自动帮我们以APP的为分类依据进行消息的分组；
* 如果我们设置了`threadIdentifier`属性则以此属性为依据，进行分组。

![1529981611452.jpg](https://upload-images.jianshu.io/upload_images/6644906-4236ec1b397b424c.jpg?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
代码如下：
```Objective-C
let content = UNMutableNotificationContent()
content.title = "Notifications Team"
content.body = "WWDC session after party"
content.threadIdentifier = "notifications-team-chat"//通过这个属性设置分组，如果此属性没有设置则以APP为分组依据
```
### 摘要(Summary)格式定制
当苹果自动将推送消息的归拢到一起的时候，最下边会有一个消息摘要。默认格式是:`n more notifications from xxx`。不过此格式我们是可以定制的。
* 第一种
```Objective-C
let summaryFormat = "%u 更多消息啦啦"
return UNNotificationCategory(identifier: "category-identifier",
actions: [],
intentIdentifiers: [],
hiddenPreviewsBodyPlaceholder: nil,
categorySummaryFormat: summaryFormat,
options: [])
```
![1529979505103.jpg](https://upload-images.jianshu.io/upload_images/6644906-0a93a4998c90342e.jpg?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
* 第二种
let summaryFormat = "%u 更多消息啦啦！来自OceanFish"
```Objective-C
let content = UNMutableNotificationContent()
content.body = "..."
content.summaryArgument = "OceanFish"
```
![1529979632784.jpg](https://upload-images.jianshu.io/upload_images/6644906-bcf29b863af11e0b.jpg?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

>同一个category的不同格式，苹果会将其合并在一起；并且不同的`summaryArgument`苹果也会将其默认合并到一起进行显示

![1529979325697.jpg](https://upload-images.jianshu.io/upload_images/6644906-fd30d599f801a0e8.jpg?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
> 也可以通过`let summaryFormat = NSString.localizedUserNotificationString(forKey: "NOTIFICATION_SUMMARY",
arguments: nil)`来进行本地化服务

### 数字定制
有时会出现另一个场景：比如发送了2条推送消息，一条是“你有3个邀请函”，另一条是“你有5个邀请函”。那摘要则会显示你有2更多消息。这显然不是我们想要的！我们最好的期望肯定是"你有8个邀请函"。那这种效果怎么显示呢？

苹果给我们提供了另外一个属性，结合上边的[摘要(Summary)格式定制](#)我们可以实现以上效果。

```Objective-C
let content = UNMutableNotificationContent()
content.body = "..."
content.threadIdentifier = "..."
content.summaryArgument = "Song by Song"
content.summaryArgumentCount = 3
```
当多个消息归拢到一起的时候，苹果会将`summaryArgumentCount`值加在一起，然后进行显示
## 推送内容扩展中的可交互和动态更改Action
之前消息是不支持交互的和动态更改Action的，比如界面有个空心喜欢按钮，用户点击则变成了实心喜欢按钮；有个Acction显示“喜欢”,用户点击之后变成"不喜欢"

#### 推送界面可交互

![1529990894961.jpg](https://upload-images.jianshu.io/upload_images/6644906-f817e45d3e077143.jpg?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
如上图推送界面有个空心喜欢按钮

* 首先配置Notification Content Extention的`UUNNotificationExtensionUserInteractionEnabled`为`YES`

![1529990622777.jpg](https://upload-images.jianshu.io/upload_images/6644906-141ec5cefa721ef2.jpg?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
* 然后代码实现

```Objective-C
import UserNotificationsUI
class NotificationViewController: UIViewController, UNNotificationContentExtension {

    @IBOutlet var likeButton: UIButton?

    likeButton?.addTarget(self, action: #selector(likeButtonTapped), for: .touchUpInside)

    @objc func likeButtonTapped() {
        likeButton?.setTitle("♥", for: .normal)
        likedPhoto()
    }
}

```
#### Action动态化

![1529990465078.jpg](https://upload-images.jianshu.io/upload_images/6644906-e75934704fdd6b4d.jpg?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

![1529990483022.jpg](https://upload-images.jianshu.io/upload_images/6644906-6d7dd0b8ae6172d1.jpg?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)


```Objective-C
// Notification Content Extensions
class NotificationViewController: UIViewController, UNNotificationContentExtension {

    func didReceive(_ response: UNNotificationResponse, completionHandler completion:
        (UNNotificationContentExtensionResponseOption) -> Void) {
        if response.actionIdentifier == "like-action" {
            // Update state...
            let unlikeAction = UNNotificationAction(identifier: "unlike-action",
                                                    title: "Unlike", options: [])
            let currentActions = extensionContext?.notificationActions
            let commentAction = currentActions![1]
            let newActions = [ unlikeAction, commentAction ]
            extensionContext?.notificationActions = newActions
        }
    }
}
```
>`performNotificationDefaultAction()`用于点击推送的时候启动应用；`dismissNotificationContentExtension()`用于关闭锁屏页面的推送具体一条消息
## 推送消息的管理
这个主要是苹果针对消息增加了一个“管理”的按钮，消息左滑即可出现。
![1529991382740.jpg](https://upload-images.jianshu.io/upload_images/6644906-1472847689393329.jpg?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
帮助我们快速的针对消息进行设置。
* Deliver Quietly 则会不会播放声音。
* turn off 则会关闭推送
* Setttings 我们可以自己定制

```Objective-C
import UIKit
import UserNotifications
class AppDelegate: UIApplicationDelegate, UNUserNotificationCenterDelegate {
    func userNotificationCenter(_ center: UNUserNotificationCenter,
                                openSettingsFor notification: UNNotification? ) {

    }
}
```
## 临时授权
临时授权主要体现就是推送消息过来会有两个按钮，会主动让用户自己选择
![1529991740590.jpg](https://upload-images.jianshu.io/upload_images/6644906-2174115f81cf5aa7.jpg?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

```Objective-C
let notificationCenter = UNUserNotificationCenter.current()

noficationCenter.requestAuthorization(options: [.badge,.alert,.sound,.provisional]) { (tag, error) in

}
```
在申请权限的时候，加上`provisional`即可。
## 警告消息
比如家庭安全、健康、公共安全等因素的时候。此消息需要用户必须采取行动。最简单的一个场景是家里安装了一个摄像头，我们去上班了，此时如果家中有人，则摄像头会推送消息给我们。
![1529991999274.jpg](https://upload-images.jianshu.io/upload_images/6644906-51cb5eb4f5b31db1.jpg?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
* 证书申请
https://developer.apple.com/contact/request/notifications-critical-alerts-entitlement/

* 本地权限申请
```Objective-C
let notificationCenter = UNUserNotificationCenter.current()
noficationCenter.requestAuthorization(options: [.badge,.alert,.sound,.criticalAlert]) { (tag, error) in

}
```
在申请权限的时候，加上`criticalAlert`。
* 播放声音
```Objective-C
let content = UNMutableNotificationContent()
content.title = "WARNING: LOW BLOOD SUGAR"
content.body = "Glucose level at 57."
content.categoryIdentifier = "low-glucose—alert"
content.sound = UNNotificationSound.criticalSoundNamed(@"warning-sound" withAudioVolume: 1.00)
```

```
// Critical alert push payload

{
  // Critical alert push payload

  {
      "aps" : {
          "sound" : {
              "critical": 1,
          }
      }
      "name": "warning-sound.aiff",
      "volume": 1.0
  }
}
```
# 总结
至此WWDC中关于推送都已经整理完毕。大家有不懂的欢迎留言相互交流
# 引用

* [源码Using, Managing, and Customizing Notifications](https://developer.apple.com/sample-code/wwdc/2017/Customizing-UserNotifications.zip)
* [What’s New in User Notifications](https://developer.apple.com/videos/play/wwdc2018/710/)
* [Using Grouped Notifications](https://developer.apple.com/videos/play/wwdc2018/711/)
