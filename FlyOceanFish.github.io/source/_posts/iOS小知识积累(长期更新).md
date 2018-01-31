---
title: iOS小知识积累(长期更新) # 这是标题
date: 2018-01-29 09:35:00
updated: 2017-01-29 09:06:00
categories:  # 这里写的分类会自动汇集到 categories 页面上，分类可以多级
- iOS # 一级分类
tags:   # 这里写的标签会自动汇集到 tags 页面上
- iOS
---
***以前工作中有很多小的知识点，但是有时候只是用了，没有真正积累下来，有时候也会忘记。所以写这篇文章就是慢慢的将以前小的知识点或者之后用到的用文字的形式记录下来。***
1、_Nullable和_Nonnull(Xcode 6.3 )
_Nullable表示可能是NULL或nil；_Nonnull表示不应该是NULL或nil。如果不遵循会有警告
如果需要每个属性或每个方法都去指定nonnull和nullable，是一件非常繁琐的事。苹果为了减轻我们的工作量，专门提供了两个宏：NS_ASSUME_NONNULL_BEGIN和NS_ASSUME_NONNULL_END。
2、__kindof(Xcode 6.3 )
表示我们不用强转类型了。比如：

    - (__kindof UITableViewCell *)dequeueReusableCellWithIdentifier:(NSString *)identifier;
我们我可以直接用某个cell接收，不用再写（SomeCell*）这种形式强转了。

    @property (nonatomic, readonly, copy) NSArray<__kindof UIView *> *subviews;
这样，写下面的代码时就没有任何警告了：

    UIButton *button = view.subviews.lastObject;
3、instancetype、id、NSObject
instancetype 与 id 不一样, instancetype 只能在方法声明中作为返回类型使用。我们写方法返回的时候用instancetype而不用id了。instancetype代表当前类的类型。
* id关键字在编译时不被检查，而NSObject在编译时会被检查是否被调用一些错误方法。
* id可以是任何对象，包括非NSObject对象
* 定义id的时候不使用*，是一个指针，NSObject却需要。

4、Storyboard References (iOS 9)
可以帮我们简化Storyboard,平时我们的做法是将一个大的Storyboard人为的去分成几个小的，然后代码组装起来。Storyboard References可以帮我们简化这一步。如果本身有一个Storyboard很负责，可以选择其中联系很大的一部分，然后点击 Xcode 的菜单栏，选择"Editor->Refactor to Storyboard"。这时候就有一个独立的Storyboard了。
5、打印当前ViewController 的名字
这个对于我们拿到一个陌生的代码，调试的时候非常方便。比如我们运行一个APP，看到一个界面，但是想知道对应哪个ViewController的时候。

* 添加Symbolic Exception
![A7AB9D87-CF83-4F63-8D66-9C5A113B76ED.png](http://upload-images.jianshu.io/upload_images/6644906-649d65ae020b72cb.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

* 添加Symob和Action
![73D79C18-5E28-4A87-B039-4DDA566FD5F7.png](http://upload-images.jianshu.io/upload_images/6644906-6aca286b93c6de2b.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

6、Autolayout中Intrinstic Size、Compression Resistance Priority、Compression Resistance Priority
* Intrinstic Size就是不用设置frame，视图通过内容、字体大小等属性就能获得的大小。我们设置的Content Hugging Priority和Compression Resistance Priority都是针对于这个的。
* Content Hugging Priority 内容拥抱，就是想变小的优先级
* Compression Resistance Priority抗压缩，就是想变大的优先级


![A90EC537-5342-47B0-8207-9F3DBC0AB8A3.png](http://upload-images.jianshu.io/upload_images/6644906-625f6cbcf66800cf.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
如上图视图内容比文字内容宽，这个时候我们想要视图变小，与文字大小一样（此时intrinsic的大小就是文字区域部分的大小）。所以只要改变Content Hugging Priority的Horizontal的优先级变大就好了
7、topLayoutGuide&& bottomLayoutGuide
这两个属性可以理解为性表示不希望被透明的状态栏或导航栏（UITabBar、UIToolBar、NavigationBar等）遮挡的内容范围的最高或最低位置。这个属性的值是它的length属性的值。如下：

    [view mas_makeConstraints:^(MASConstraintMaker *make) {
        make.top.mas_equalTo(self.mas_topLayoutGuideBottom);
        make.leading.mas_equalTo(self.view.mas_leading);
        make.trailing.mas_equalTo(self.view.mas_trailing);
        make.bottom.mas_equalTo(self.mas_bottomLayoutGuideTop);
    }];
8、Autolayout中约束LayoutConstraint的有效无效（8.0）
有时候我们经常遇到一个view在不同的场景下约束不同，通常做法就是在不同场景下先删除再增加这样操作，这样效率反而不高。我们可以通过active这个属性设置生效不生效。

    self.mLayoutConstraintWidth.active = YES;
9、隐藏导航栏返回按钮旁边的文字

    [[UIBarButtonItem appearance] setBackButtonTitlePositionAdjustment:UIOffsetMake(0, -60) forBarMetrics:UIBarMetricsDefault];
10、替换返回按钮图片
````
UIImage *bgImage = [[UIImage imageNamed:@"return"] imageWithRenderingMode:UIImageRenderingModeAlwaysOriginal];
[[UINavigationBar appearance] setBackIndicatorImage:bgImage];
[[UINavigationBar appearance] setBackIndicatorTransitionMaskImage:bgImage];
````
11、头文件导入
@class 在.h文件时引入其他.h文件的时候，效率更高
#include "" 自己写的文件
#include <> 引入系统的
#import <> 用于包含系统文件
#import ""自己写的文件，搜索路径更广
import比include更好，能够避免文件的重复导入，效率更高
不过还有一个更好的的方式[@import](https://stackoverflow.com/questions/18947516/import-vs-import-ios-7/18947634#)，这种方式让我们不必在设置里边添加包了，它会帮我们自动关联。
12、内存管理语义
* ***assign*** "设置方法"只会执行针对"纯量类型"(例如CGFloat或NSInteger等)的简单赋值操作
* ***strong*** 此特质表明该属性定义了一种"拥有关系"。为这种属性设置新值时，设置方法会先保留新值，并释放旧值，然后再将新值设置上去。
* ***weak*** （5.0以上，可以说是对unsafe_unretained的一个升级,更安全，但是由于要遍历是否已经销毁，并且设置成nil，所以是消耗性能）此特质表明该属性定义了一种"非拥有关系"。为这种属性设置新值时，设置方法既不保留新值，也不释放旧值。此特质同assign类似，然而在属性所指的对象遭到摧毁时，属性值也会清空(nil)。
* ***unsafe_unretained*** (5.0以下必须使用而不能使用weak，优点比weak快，缺点容易造成野指针)此特质的语义和assign相同，但是它使用于"对象类型"，该特质表达一种"非拥有关系"，当对象遭到摧毁时，属性值不会自动清空（"不安全",unsafe）这一点与weak有区别。
* ***copy*** 此特质所表达的所属关系与strong类似。然而设置方法并不保留新值，而是将其"拷贝"。警惕NSMutableArray为copy的时候奔溃问题。copy之后是NSArray添加对象会crash。
13、查找或判断xib是否存在
````
NSString *path = [[NSBundle mainBundle] pathForResource:NSStringFromClass(className) ofType:@"nib"];
Boolean exist = [[NSFileManager defaultManager] fileExistsAtPath:path];
````
备注：其实关键就是Typ是"nib"而不是"xib"。因为我们打出的包，系统将我们的xib文件编译成了以nib为结尾的文件。
14、UI_APPEARANCE_SELECTOR
当我们自定义控件的时候可以在属性后边增加这个声明。代表支持appearance方法。如果你自己调用了setXX方法，则appearance方法失效。
````
@property (nonatomic, assign) CGFloat selectionIndicatorHeight UI_APPEARANCE_SELECTOR;
````
````
[XXXClass appearance].selectionIndicatorHeight = xxxx
````
15、旋转动画方向问题
````
button.transform = CGAffineTransformRotate(button.transform,M_PI);
````
我们先旋转180顺时针
````
button.transform = CGAffineTransformIdentity;
````
当我们调用上句话的时候，是顺时针旋转180返回原位置，但是很多时候我们想原路返回，其实就是想逆时针返回。
````
button.transform = CGAffineTransformRotate(button.transform,M_PI-0.0001);
````
只要这么设置就会逆时针返回了。旋转动画选择了最短路径返回

16、纯代码布局，添加约束
如果我们使用纯代码布局，则需要以下代码约束才能起作用；默认左右是有16像素间距的，设置左右各-16才能靠边(这个如果有其他好办法可以留言给我)

    self.view.translatesAutoresizingMaskIntoConstraints = NO;
17、NSDate和NSDateFormatter
NSDate：是获取的时间没有时区概念，所以获取的是0时区的时间。

    [NSDate date]//获取的时间与当前时间差8个小时

NSDateFormatter：将日期格式成我们想要的。同时会根据时区、地域给我们做相应的计算。(看注释)

````
NSDateFormatter *formatter = [[NSDateFormatter alloc] init];
formatter.dateFormat = @"yyyy-MM-dd HH:mm:ss";
NSTimeZone *timeZone = [NSTimeZone localTimeZone];
//  formatter.locale = [NSLocale currentLocale];
 formatter.timeZone = timeZone;
NSDate *currentDate = [NSDate date];
NSString *currentString = [formatter stringFromDate:currentDate];//会将currentDate增加8个小时，获取我们当前时区的时间
NSDate *date = [formatter dateFromString:@"2017-8-31 9:51:00"];//获取的date是针对0时区的时间，所以会减去8个小时

````
以上代码可以看到我们可以设置timeZone和locale。如果我们只设置local照样能够输出我们想要的结果。但是二者有什么区别呢？
  - local是设置区域的。配合setTimeStyle:、setDateStyle:这些方法可以控制日期输出不同的格式。因为每个地方的格式不同。  [Formatting Data Using the Locale Settings](https://developer.apple.com/library/content/documentation/MacOSX/Conceptual/BPInternational/InternationalizingYourCode/InternationalizingYourCode.html)


![7FAEF832-CBCD-41A2-A4EE-4056FBF1B931.png](http://upload-images.jianshu.io/upload_images/6644906-320495bdfa8f9c47.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

 - timeZone设置时区的。主要用来将时间转换成不同时区。

所以在代码格式化的时候，建议设置timezone来格式化我们的时期

18、Xcode关闭控制台打印不用信息

 Xcode->Edit Scheme -> Run -> Arguments, 在Environment Variables里边添加 OS_ACTIVITY_MODE ＝ disable

19、NSObject中isKindOfClass、isMemberOfClass、isSubclassOfClass区别

isSubclassOfClass和isKindOfClass的作用基本上是一致的，只不过一个是类方法，一个是对象方法。
isKindOfClass来确定一个对象是否是一个类的成员，或者是派生自该类的成员,
isMemberOfClass只能确定一个对象是否是当前类的成员.
isMemberOfClass 筛选条件更为苛刻，只有当类型完全匹配的时候才会返回YES。

20、判断当前queue是否是main queue
````
  static void *mainQueueKey = &mainQueueKey;
  static dispatch_once_t onceToken;
  dispatch_once(&onceToken, ^{
    dispatch_queue_set_specific(dispatch_get_main_queue(),
                                mainQueueKey, mainQueueKey, NULL);
  });
  return dispatch_get_specific(mainQueueKey) == mainQueueKey;
````
21、为什么有的属性要用copy，而不用strong

像NSString和NSArray这两类，在给变量赋值的时候，如果来源是可变字符串或可变数组的时候，使用copy能够避免原来变量对现在变量的影响。如果用strong的话，原来变量改变时同时会影响我们现在的变量。我们大多数期望的结果是两个变量相互独立，互补影响的，所以使用copy能够避免这类问题
22、SEL、IMP

SEL 函数的ID，相当于将函数的名字用一个唯一识别符与之相对应。
IMP 本质是函数指针。其实就是函数的实现，我们在调用方法的时候是一层一层的查找，最终找到了IMP，如果直接调用IMP将会更有效率

23、消息的转发

![44181-09a882edcb98d535.jpg](http://upload-images.jianshu.io/upload_images/6644906-9209401db1a78dc4.jpg?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
runtime在调用方法的时候如果找不到会直接奔溃。如果在当前类找不到的时候，会开始顺序的调用系统的三个方法，如果再找不到会直接crash。
````
1.+(Bool) resolveInstanceMethod:(SEL)sel // 对应实例方法
  + (Bool)resolveClassMethod:(SEL)sel // 对应类方法
我们可以在这个方法中调用class_addMethod将方法加入就不会crash
2.- (id)forwardingTargetForSelector:(SEL)aSelector//可以改变查找方法的对象
3.- (void)forwardInvocation:(NSInvocation *)anInvocation//搭配- (NSMethodSignature *)methodSignatureForSelector:(SEL)aSelector这个使用。通过传递过来的NSInvocation对象，我们可以对此重新注入调用对象、方法、参数等
````

24、`NSObject`的description和debugDescription方法
当我们在工程中使用model(继承NSObject)的时候，在断点或者log调试中有时候需要将对象打印到控制台中，但是我们自己写的model打印出的是对象的地址。如果需要打印出有效的信息，这时我们可以重写descrition或者debugDescription方法，打印出我们想要的信息。例如对象的所有属性的值。***默认情况下debugDescription调用的是description方法***

25、 UIView的setNeedsLayout、layoutIfNeeded 、setNeedsDisplay
* ***setNeedsLayout*** 标记为需要重新布局，不立即刷新，但layoutSubviews一定会被调用
配合layoutIfNeeded立即更新
* ***layoutIfNeeded*** 如果有有需要刷新的标记，立即调用layoutSubviews进行布局，并且进行页面刷新
例如我在使用约束做动画的时候:
````Objective-C
leftContrain.constant = 100
UIView.animateWithDuration(0.8, delay: 0, usingSpringWithDamping: 0.5, initialSpringVelocity: 0.5, options: UIViewAnimationOptions.AllowAnimatedContent, animations: {
                self.view.layoutIfNeeded() //立即实现布局
            }, completion: nil)
````
如果不调用`layoutIfNeeded`则动画一点卵用没有
* ***setNeedsDisplay或setNeedsDisplayInRect***   会调用自动调用drawRect方法，这样可以拿到  UIGraphicsGetCurrentContext，就可以画画了。而setNeedsLayout会默认调用layoutSubViews，
    * setNeedsDisplay 对应画布的大小就是UIView的大小，所以会将整个UIView进行绘制
    * setNeedsDisplayInRect 对应画布的大小是调用此方法传入的rect，如果此前UIView已经绘制过一次，则会保留上次的绘制，仅仅对对应传入的rect进行刷新绘制
      >This method makes a note of the request and returns immediately. The view is not actually redrawn until the next drawing cycle, at which point all invalidated views are updated.


>setNeedsDisplay方便绘图，而layoutSubViews方便出来数据。

26、通知的命名的正确姿势

>[Name of associated class]+[Did|Will]+[UniquePartOfName]+Notification

我们可以按照苹果模板写

.h文件

>UIKIT_EXTERN NSNotification const FOFAnimalDidBecomePersonNotification

.m文件

>NSNotification FOFAnimalDidBecomePersonNotificationFOFAnimalDidBecomePersonNotification = @"FOFAnimalDidBecomePersonNotification";

27、对象、类、元类、isa

类对象代表类,Class类型。如[类名 class];

所有类的实例都由类对象生成,**类对象会把实例的isa的值修改成自己的地址,每个实例的isa都指向该实例的类对象**

因为类也是一个对象,那它也必须是另一个类的实例,这个类就是元类 `metaclass`

>元类保存了**类方法**(静态方法"+")的列表。当一个类方法被调用时,元类会首先查找它本身是否有该类方法的实现,如果没有则该元类会向它的父类查找该方法,直到一直找到继承链的头。

  > 元类(metaclass)也是一个对象,那么元类的isa指针又指向哪里呢?为了设计上的完整,所有的元类的isa指针都会指向一个根元类(root metaclass)。

![图.png](http://upload-images.jianshu.io/upload_images/6644906-f8bf2f90a3aa0f7c.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

28、Cocoapods导入第三方包Xcode没有自动补全提示功能
选择Target -> Build Settings 菜单，找到\”User Header Search Paths\”设置项
新增一个值"${SRCROOT}"，并且选择\”Recursive\”，这样xcode就会在项目目录中递归搜索文件

29、__bridge

OC对象和CF(CoreFoundation)对象之间转换

* __bridge:OC对象转换成CF对象，转化时只涉及对象类型不涉及对象所有权的转化。（内存还是oc管理）

* __bridge_transfer:常用在讲CF对象转换成OC对象时，将CF对象的所有权交给OC对象，此时ARC就能自动管理该内存；（作用同CFBridgingRelease()）

* __bridge_retained:（与__bridge_transfer相反）常用在将OC对象转换成CF对象时，将OC对象的所有权交给CF对象来管理；(作用同CFBridgingRetain())，最后调用CFRelease将其手动释放.

30、tableview 分割线靠边
设置tableview的separate line 的时候，如果已经设置了Separator Inset (left=0)但是线还是不靠边。
>cell.layoutMargins = UIEdgeInsets.zero

31、倒计时UIButton数字闪烁解决方法
默认`type`是`system`，将`type`改成`custom`即可。[UIbutton+Timer](https://github.com/FlyOceanFish/UIButtonTimer)
