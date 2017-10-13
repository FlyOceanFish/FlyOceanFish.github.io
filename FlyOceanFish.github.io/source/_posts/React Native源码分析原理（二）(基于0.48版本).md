---
title: React Native源码分析原理（二）(基于0.48版本) # 这是标题
categories:  # 这里写的分类会自动汇集到 categories 页面上，分类可以多级
- React Native # 一级分类
tags:   # 这里写的标签会自动汇集到 tags 页面上
- React Native # 可配置多个标签，注意格式
- iOS
---
* [React Native源码分析原理（一）(基于0.48版本)](http://www.jianshu.com/p/8324b379c020)
* [React Native源码分析原理（二）(基于0.48版本)](http://www.jianshu.com/p/0688b24950f4)

上一篇文章大家如果仔细阅读揣摩对RN有了一个初步的认识了，接下来将基于上一篇文章的这种初步认识然我们详细了解一下RN的启动过程
### 启动过程
[RCTRootView initWithBundleURL:...]
[RCTBridge initWithBundleURL:...]
[RCTBridge setUp]
初始化batchedBridge
[RCTCxxBridge start]
开启一个线程jsThread用于js
[RCTCxxBridge _initModulesWithDispatchGroup]
初始化JSCExecutorFactory, 用于执行js代码以及处理回调 JSCExecutor::JSCExecutor()
JSCExecutor::initOnJSVMThread()
installGlobalProxy -> nativeModuleProxy
RCTJavaScriptLoader -> 加载js代码
[RCTCxxBridge executeSourceCode]
[RCTRootView initWithBridge:...]
[RCTRootView bundleFinishedLoading]
初始化RCTRootContentView
[RCTRootView runApplication]//是Native调用js的一个完美例子。本质调用了AppRegistry的runApplication方法
我们初始化RN工程一般都是如下代码：
````
  jsCodeLocation = [[RCTBundleURLProvider sharedSettings] jsBundleURLForBundleRoot:@"index.ios" fallbackResource:nil];

  RCTRootView *rootView = [[RCTRootView alloc] initWithBundleURL:jsCodeLocation
                                                      moduleName:@"AwesomeProject"
                                               initialProperties:nil
                                                   launchOptions:launchOptions];
````
1、 获取到app的js入口文件的NSURL
2 、初始化RCTRootView, 传递的参数moduleName:@"AwesomeProject"是我们在入口的js文件中注册的, initialProperties:nil, 将作为js中的跟view的props, 可以用来传递需要的初始数据

进入RCTRootView初始化中，其实很简单
````
  RCTBridge *bridge = [[RCTBridge alloc] initWithBundleURL:bundleURL
                                            moduleProvider:nil
                                             launchOptions:launchOptions];

  return [self initWithBridge:bridge moduleName:moduleName initialProperties:initialProperties];
````
就实例化了一个RCTBridge，这个RCTBridge的实例其实就是一个外壳而已，接下来我们看看内部，为啥是一个外壳。
````
[self setUp];
````
内部中这个方法就是关键，这个方法里边实例化了一个真正的Bridge，即`RCTCxxBridge`或`RCTBatchedBridge`，所以说一开始实例化的Bridge只是一个壳子而已。为什么会出现两个不同的Bridge呢？主要是0.44之前的都是用的`RCTBatchedBridge`，之后RN用的是`RCTCxxBridge`，根据它的官方资料`RCTCxxBridge`会慢慢取代`RCTBatchedBridge`。(`RCTCxxBridge`有个问题就是依赖了4个包，而且这4个包被墙了具体可以看[将RN工程嵌入到现有原生iOS应用](http://www.jianshu.com/p/1927785a3ba7))。
````
  self.batchedBridge = [[bridgeClass alloc] initWithParentBridge:self];
  [self.batchedBridge start];
````
首先是bridgeClass的获取，具体代码是优先加载`RCTCxxBridge`，如果获取不到的话，其次加载`RCTBatchedBridge`。本次分析是基于0.48并且工程初始化是使用了`RCTCxxBridge`，所以下边只要提到Bridge说的就是`RCTCxxBridge`。

start是一个非常有料的方法。就像之前说的Bridge其实就是实现了一个调度管理的作用，那调度管理什么呢？所有需要调度管理的前提环境和类全部都是在start方法进行了初始化。

1、  首先做的是实例化了一个js线程，我们之前所说的js thread就是它，用来执行js代码的线程
````
  // Set up the JS thread early
  _jsThread = [[NSThread alloc] initWithTarget:self
                                      selector:@selector(runJSRunLoop)
                                        object:nil];
  _jsThread.name = RCTJSThreadName;
  _jsThread.qualityOfService = NSOperationQualityOfServiceUserInteractive;
  [_jsThread start];
````
这个线程执行的方法是`runJSRunLoop`通过名字大家也能猜出一个大概吧。其实就是启动了线程的runloop，保证了_jsThread能够一直运行。

2、创建了一个group，用来初始化Birdge。接下来就开始加载我们本地所有的已经实现了RCTBridgeModule协议的modules
````
   dispatch_group_t prepareBridge = dispatch_group_create();
  // Initialize all native modules that cannot be loaded lazily
  [self _initModulesWithDispatchGroup:prepareBridge];
````
3、让我们看看initModulesWithDispatchGroup内部
````
 for (Class moduleClass in RCTGetModuleClasses()) {
    NSString *moduleName = RCTBridgeModuleNameForClass(moduleClass);

    // Don't initialize the old executor in the new bridge.
    // TODO mhorowitz #10487027: after D3175632 lands, we won't need
    // this, because it won't be eagerly initialized.
    if ([moduleName isEqual:@"RCTJSCExecutor"]) {
      continue;
    }

    // Check for module name collisions
    RCTModuleData *moduleData = moduleDataByName[moduleName];
    if (moduleData) {
      if (moduleData.hasInstance) {
        // Existing module was preregistered, so it takes precedence
        continue;
      } else if ([moduleClass new] == nil) {
        // The new module returned nil from init, so use the old module
        continue;
      } else if ([moduleData.moduleClass new] != nil) {
        // Both modules were non-nil, so it's unclear which should take precedence
        RCTLogError(@"Attempted to register RCTBridgeModule class %@ for the "
                    "name '%@', but name was already registered by class %@",
                    moduleClass, moduleName, moduleData.moduleClass);
      }
    }

    // Instantiate moduleData
    // TODO #13258411: can we defer this until config generation?
    moduleData = [[RCTModuleData alloc] initWithModuleClass:moduleClass
                                                     bridge:self];
    moduleDataByName[moduleName] = moduleData;
    [moduleClassesByID addObject:moduleClass];
    [moduleDataByID addObject:moduleData];
  }
````
主要是循环遍历用2个数组分别存储了所有的class和RCTModuleData(存储着我们class所有的信息包括class名称、所有的方法、等一些有效的信息和一个instance。还有一个Dictionary key是class，value是moudleData。这个就是通过class名称去取moduleData用的。

这个方法中有一个很重要的方法`RCTGetModuleClasses()`。
````
void RCTRegisterModule(Class);
void RCTRegisterModule(Class moduleClass)
{
  static dispatch_once_t onceToken;
  dispatch_once(&onceToken, ^{
    RCTModuleClasses = [NSMutableArray new];
  });

  RCTAssert([moduleClass conformsToProtocol:@protocol(RCTBridgeModule)],
            @"%@ does not conform to the RCTBridgeModule protocol",
            moduleClass);

  // Register module
  [RCTModuleClasses addObject:moduleClass];
}
````
这个就是我们一直说的必须要实现了RCTBridgeModule这个协议，如果实现其实会直接报错的。这个加载是通过`RCT_EXPORT_MODULE()`宏来实现的，宏是通过运行时`+load()`方法在我们应用启动的时候其实已经将所有的module加入到了数组之中。

4 、以上终于完成了Modules信息的保存, 接着会进行JSExecutorFactory的初始化和相关的设置, JSExecutorFactory内部会持有一个JSCExecutor 所有与JS的通信，一定都通过JSCExecutor来进行, 所以需要重点关注, 它的构造函数中调用了initOnJSVMThread(), 里面进行了很多的JS上下文的准备, 创建JSClass, 全局的context, 以及添加全局的回调.注意在这个里面使用到的JSC_JSXXX的宏的作用, 实际上是会转换为调用苹果的JavaScriptCore对应的方法(去掉JSC_), 同时还要留意JSCExecutor构造函数中为js的上下文中设置了一个Proxy, nativeModuleProxy.
````
void JSCExecutor::initOnJSVMThread()
  // 在js的上下文中注册全局的闭包回调 , 这两个是我们需要重点关注的回调.
  nativeFlushQueueImmediate
  nativeCallSyncHook

    installNativeHook<&JSCExecutor::nativeFlushQueueImmediate>("nativeFlushQueueImmediate");
    // 这一个注册的回调, 可以用于在js中直接调用oc, 注意, 在rn中js调用oc一般都是由oc发起的,
    // 这就像我们的界面一般是有用户的操作才会触发系统调用, rn中类似的,
    // 当oc调了js后, 在对应的js回调中才会实现调用oc的逻辑

 // 在react native MessageQueue.js中的源码中可以找到 这样一段代码,
 // 就是当oc没有及时的在消息队列中调js的时候, 大于5ms后js就会调用这个回调, 主动调用oc

   MIN_TIME_BETWEEN_FLUSHES_MS = 5ms;
      if (global.nativeFlushQueueImmediate &&
        (now - this._lastFlush >= MIN_TIME_BETWEEN_FLUSHES_MS ||
         this._inCall === 0)) {
      var queue = this._queue;
      this._queue = [[], [], [], this._callID];
      this._lastFlush = now;
      global.nativeFlushQueueImmediate(queue);
    }

// 这个回调是用来调用查找native中对应的方法相关信息的

installNativeHook<&JSCExecutor::nativeCallSyncHook>("nativeCallSyncHook");
在react native的 NativeModules.js中有相关的调用来获取native方法的相关信息
function genMethod(moduleID: number, methodID: number, type: MethodType) {
//...
    return global.nativeCallSyncHook(moduleID, methodID, args);
}
````
为js的上下设置 nativeModuleProxy, 这一个属性很重要, js端用来获取所有注册的nativeModule. 在js中我们需要引入native端的时候回写这样的类似代码 var {NativeModules} from 'react-native', 实际上就是获取到我们这里设置的.
````
JSCExecutor::JSCExecutor()
  {
  // 这个函数在js的全局上下文中设置了`nativeModuleProxy`,
// 使得在js端可以获取到, 同时注意这个函数的第三个参数中
// 还包装了一个方法用于调用, 里面涉及到JSCNativeModules这个类,
// 就是获取NativeModule的操作, 后面分析
    installGlobalProxy(m_context, "nativeModuleProxy",
                       exceptionWrapMethod<&JSCExecutor::getNativeModule>());
  }

在react-native 的NativeModules.js底部有这样的代码, 就直接调用了上面在native端设置的全局属性了

let NativeModules : {[moduleName: string]: Object} = {};
if (global.nativeModuleProxy) {
  NativeModules = global.nativeModuleProxy;
}
module.exports = NativeModules;
````
那么js中是怎么获取到我们之前在native端已经生成好的'配置表'信息的呢? 这个问题需要我们重点关注下, 在之前的react-native版本中, 是通过native端直接在js的上下文中注入这样一个__fbBatchedBridgeConfig配置表,
传过去一个json(remoteModuleConfig). 现在的版本中改了一些. 还是上面这段代码. 在如下的调用栈中,我们只需要关注最后一个函数.

JSCExecutor::getNativeModule()
JSCNativeModules::getModule()
JSCNativeModules::createModule()
ModuleRegistry::getConfig()
RCTNativeModule::getMethods()
NSStringFromSelector(selector) hasPrefix:@"rct_export"(初始化RCTModuleMethod信息)
````
folly::Optional<Object> JSCNativeModules::createModule(const std::string& name, JSContextRef context) {
  if (!m_genNativeModuleJS) {
// 获取js中contenxt中的global
    auto global = Object::getGlobalObject(context);  
// 获取 global中的__fbGenNativeModule对象
    m_genNativeModuleJS = global.getProperty("__fbGenNativeModule").asObject();
    m_genNativeModuleJS->makeProtected();
  }

 // 这个方法最终会调到 RCTNativeModule::getMethods()中,
 // 取得之前使用宏暴露的所有的方法  -> 前缀是 "__rct_export__"
 auto result = m_moduleRegistry->getConfig(name);


  if (!result.hasValue()) {
    return nullptr;
  }
// 调用 js中的__fbGenNativeModule方法, 并且传递过去native端的配置表, 用于端js处理
  Value moduleInfo = m_genNativeModuleJS->callAsFunction({
    Value::fromDynamic(context, result->config),
    Value::makeNumber(context, result->index)
  });
  return moduleInfo.asObject().getProperty("module").asObject();
}

// 下面是对应的js中的处理, NativeModules.js
// 暴露一个全局的属性给native(就是上面我们调用的js中的方法, 触发了后面的js后取到native端的配置表, 并且保存)
// export this method as a global so we can call it from native
global.__fbGenNativeModule = genModule;

function genModule(...) {
  // 获取到native端调用时传递过来的配置信息
  const [moduleName, constants, methods, promiseMethods, syncMethods] = config;
  const module = {};
  // 遍历 所有的native的方法列表, 获取所有native的方法的详细信息
  methods && methods.forEach((methodName, methodID) => {
    module[methodName] = genMethod(moduleID, methodID, methodType);
  });

}
function genMethod(...) {
  /// ...
  // 调用之前native端在js中注入的回调, 获取native中的所有方法的详细信息
  return global.nativeCallSyncHook(moduleID, methodID, args);
}
````
5、 使用dispatch_group方式初始化Instance(执行代码需要), 和加载js代码RCTJavaScriptLoader,这个类比较单纯，就是用来加载js代码的，没有其他用处
````
NSData *sourceCode = [RCTJavaScriptLoader attemptSynchronousLoadOfBundleAtURL:self.bundleURL
                                                                 runtimeBCVersion:bcVersion
                                                                     sourceLength:NULL
                                                                            error:&error];
````
6、 modules and source code都已经准备好了, 在notify中JSCExecutor专属的Thread内执行jsbundle代码执行js代码. 执行完毕后会将_displayLink添加到runloop(注意是在jsThread所在的runloop)中, 开始运行。

[RCTCxxBridge executeSourceCode: ]
[RCTCxxBridge enqueueApplicationScript:]
void Instance::loadScriptFromString()
void NativeToJsBridge::loadApplication()
void JSCExecutor::loadApplicationScript()
void JSCExecutor::flush()
void JSCExecutor::bindBridge()
上面的调用栈中我们主要关注后面几个函数

`void JSCExecutor::loadApplicationScript()`, 在这个函数中注意下面两行代码, 注意, RN中有个很明显的问题就是, 首次进入RN模块的时候, 加载的很慢, 会有几秒的加载时间, 其实就是在这个函数中加载jsBundle造成的, 如果要优化加载的时间, 以及不同模块页面切换流畅, 就需要对jsBundle文件进行精简, 缓存等操作。
````
  // 执行js代码, 加载jsBundle
  evaluateScript(m_context, jsScript, jsSourceURL);
  // 在加载完jsbundle后主动调用一次, 为js的上下文环境添加必要的全局属性和回调
  flush();
````
void JSCExecutor::bindBridge()保存js的上下文环境中必要的全局属性和回调, 这些在 react native的 MessageQueue.js中定义的调用方法, 当native需要调用js的方法的时候, 需要通过调用这些js方法, 在这些方法中, js端会根据传递的信息查找到在js中需要调用的方法.
````
    auto global = Object::getGlobalObject(m_context);
    auto batchedBridgeValue = global.getProperty("__fbBatchedBridge");
// 这些是在MessageQueue.js中定义的调用方法
    auto batchedBridge = batchedBridgeValue.asObject();
    m_callFunctionReturnFlushedQueueJS = batchedBridge.getProperty("callFunctionReturnFlushedQueue").asObject();
    m_invokeCallbackAndReturnFlushedQueueJS = batchedBridge.getProperty("invokeCallbackAndReturnFlushedQueue").asObject();
    m_flushedQueueJS = batchedBridge.getProperty("flushedQueue").asObject();
    m_callFunctionReturnResultAndFlushedQueueJS = batchedBridge.getProperty("callFunctionReturnResultAndFlushedQueue").asObject();
  }
````
7 、上面的工作完成之后, 初始化bridge的工作就完成了, jsBundle已经加载完成, oc端的'配置表'也已经处理好, 并且成功已经传递给js端, js的上下文配置已经准备好, 下面就是开始执行我们的js代码了, React就会开始计算好所有的布局信息, 以及Component层级关系等, 等待native端完成对应的真正的页面渲染和布局. 回到RCTRootView的初始化方法中, 注意在[RCTRootView initWithBridge:...]的初始化方法中, 注册了几个js执行情况的通知, 我们重点关注js执行完毕后的通知RCTJavaScriptDidLoadNotification
````
- (instancetype)initWithBundleURL:(NSURL *)bundleURL
                       moduleName:(NSString *)moduleName
                initialProperties:(NSDictionary *)initialProperties
                    launchOptions:(NSDictionary *)launchOptions
{
  RCTBridge *bridge = [[RCTBridge alloc] initWithBundleURL:bundleURL
                                            moduleProvider:nil
                                             launchOptions:launchOptions];
// 上面完成了bridge的初始化工作 , 下面开始完成rootView的初始化
  return [self initWithBridge:bridge moduleName:moduleName initialProperties:initialProperties];
}
````
下面是js执行完毕后的通知RCTJavaScriptDidLoadNotification中的一系列的处理

[RCTRootView javaScriptDidLoad]
[RCTRootView bundleFinishedLoading]
RCTRootContentView (创建contentView)
[RCTRootView runApplication]
在创建RCTRootContentView的时候, 注意有个参数是reactTag, 这个属性很重要, 每一个reactTag都应该是唯一的, 从1开始, 每次递增10. RCTRootContentView初始化时, 还需要在RCTUIManager中通过reactTag去注册, 从而由RCTUIManager来统一管理所有的js端使用Component对应的每个原生view(_viewRegistry[tag]表), 有了这个, 我们就可以很方便的在其他地方通过reactTag获取到我们的Component所在的rootView.
````
    /**
     * Every root view that is created must have a unique react tag.
     * Numbering of these tags goes from 1, 11, 21, 31, etc
     *
     * NOTE: Since the bridge persists, the RootViews might be reused, so the
     * react tag must be re-assigned every time a new UIManager is created.
     */

- (NSNumber *)allocateRootTag
{
  NSNumber *rootTag = objc_getAssociatedObject(self, _cmd) ?: @1;
  objc_setAssociatedObject(self, _cmd, @(rootTag.integerValue + 10), OBJC_ASSOCIATION_RETAIN_NONATOMIC);
  return rootTag;
}
````
然后才是[RCTRootView runApplication], 这里就会调用js里面AppRegistry对应的方法runApplication了, 在AppRegistry.js中有下面的一段注释, 已经解释得很清楚了
````
/**
 * `AppRegistry` is the JS entry point to running all React Native apps.  App
 * root components should register themselves with
 * `AppRegistry.registerComponent`, then the native system can load the bundle
 * for the app and then actually run the app when it's ready by invoking
 * `AppRegistry.runApplication`.
 *
 * To "stop" an application when a view should be destroyed, call
 * `AppRegistry.unmountApplicationComponentAtRootTag` with the tag that was
 * passed into `runApplication`. These should always be used as a pair.
 *
 * `AppRegistry` should be `require`d early in the `require` sequence to make
 * sure the JS execution environment is setup before other modules are
 * `require`d.
 */

- (void)runApplication:(RCTBridge *)bridge
{
  NSString *moduleName = _moduleName ?: @"";
  NSDictionary *appParameters = @{
    @"rootTag": _contentView.reactTag,
    @"initialProps": _appProperties ?: @{},
  };
// 调用AppRegistry.js中的runApplication开始运行
  RCTLogInfo(@"Running application %@ (%@)", moduleName, appParameters);
  [bridge enqueueJSCall:@"AppRegistry"
                 method:@"runApplication"
                   args:@[moduleName, appParameters]
             completion:NULL];
}
````
8、 因为执行js代码的时候, js端会计算好每个view的布局, 属性等信息, 然后通过调用native的系统方法来完成, 页面的渲染, 页面的布局. 这个过程中就涉及到了js 和native的互调通信了. 这个过程分为两个部分.
