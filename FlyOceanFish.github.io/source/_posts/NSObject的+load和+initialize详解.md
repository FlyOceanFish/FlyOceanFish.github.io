---
title: NSObject的+load和+initialize详解 # 这是标题
date: 2017-10-14 09:27:00
categories:  # 这里写的分类会自动汇集到 categories 页面上，分类可以多级
- iOS # 一级分类
tags:   # 这里写的标签会自动汇集到 tags 页面上
- iOS
---
文章比较长，第一部分通过runtime的源码介绍了load和initialize两个方法的本质；第二部门通过实例演示了这个两个方法的调用；第三部分就是结论和应用场景

#通过runtime源码解析load和initialize
## +load

![ADDBC8A1-E2D9-4BCE-8A87-B60FDCC2FB13.png](http://upload-images.jianshu.io/upload_images/6644906-034736b3848e3b2d.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
通过调用堆栈，我们可以看出系统首先调用的是load_images方法

### load_images
````
void load_images(const char *path __unused, const struct mach_header *mh)
{
    // Return without taking locks if there are no +load methods here.
    if (!hasLoadMethods((const headerType *)mh)) return;

    recursive_mutex_locker_t lock(loadMethodLock);

    // Discover load methods
    {
        rwlock_writer_t lock2(runtimeLock);
        prepare_load_methods((const headerType *)mh);
    }

    // Call +load methods (without runtimeLock - re-entrant)
    call_load_methods();
}
首先是将所有的load搜索到，放到一个列表中等待调用，然后call_load_methods循环遍历调用
````
#### prepare_load_methods

````Objective-C
void prepare_load_methods(const headerType *mhdr)
{
    Module mods;
    unsigned int midx;

    header_info *hi;
    for (hi = FirstHeader; hi; hi = hi->getNext()) {
        if (mhdr == hi->mhdr()) break;
    }
    if (!hi) return;

    if (hi->info()->isReplacement()) {
        // Ignore any classes in this image
        return;
    }

    // Major loop - process all modules in the image
    mods = hi->mod_ptr;
    for (midx = 0; midx < hi->mod_count; midx += 1)
    {
        unsigned int index;

        // Skip module containing no classes
        if (mods[midx].symtab == nil)
            continue;

        // Minor loop - process all the classes in given module
        for (index = 0; index < mods[midx].symtab->cls_def_cnt; index += 1)
        {
            // Locate the class description pointer
            Class cls = (Class)mods[midx].symtab->defs[index];
            if (cls->info & CLS_CONNECTED) {
                schedule_class_load(cls);
            }
        }
    }


    // Major loop - process all modules in the header
    mods = hi->mod_ptr;

    // NOTE: The module and category lists are traversed backwards
    // to preserve the pre-10.4 processing order. Changing the order
    // would have a small chance of introducing binary compatibility bugs.
    midx = (unsigned int)hi->mod_count;
    while (midx-- > 0) {
        unsigned int index;
        unsigned int total;
        Symtab symtab = mods[midx].symtab;

        // Nothing to do for a module without a symbol table
        if (mods[midx].symtab == nil)
            continue;
        // Total entries in symbol table (class entries followed
        // by category entries)
        total = mods[midx].symtab->cls_def_cnt +
            mods[midx].symtab->cat_def_cnt;

        // Minor loop - register all categories from given module
        index = total;
        while (index-- > mods[midx].symtab->cls_def_cnt) {
            old_category *cat = (old_category *)symtab->defs[index];
            add_category_to_loadable_list((Category)cat);
        }
    }
}
````
这个方法主要是两个核心方法schedule_class_load和add_category_to_loadable_list主要干了两件事
1、获取了所有类后,遍历列表，将其中有+load方法的类加入loadable_class；
2、获取所有的类别，遍历列表，将其中有+load方法的类加入loadable_categories.
接下来让我们看看schedule_class_load
````
static void schedule_class_load(Class cls)
{
    if (cls->info & CLS_LOADED) return;
    if (cls->superclass) schedule_class_load(cls->superclass);
    add_class_to_loadable_list(cls);
    cls->info |= CLS_LOADED;
}
````
从以上代码可以看出在加载类的load方法的时候，首先是先将父类加入到loadable_class，之后才是子类。所以保证了**父类一定是在子类先调用**！

####call_load_methods
````
void call_load_methods(void)
{
    static bool loading = NO;
    bool more_categories;

    loadMethodLock.assertLocked();

    // Re-entrant calls do nothing; the outermost call will finish the job.
    if (loading) return;
    loading = YES;

    void *pool = objc_autoreleasePoolPush();

    do {
        // 1. Repeatedly call class +loads until there aren't any more
        while (loadable_classes_used > 0) {
            call_class_loads();
        }

        // 2. Call category +loads ONCE
        more_categories = call_category_loads();

        // 3. Run more +loads if there are classes OR more untried categories
    } while (loadable_classes_used > 0  ||  more_categories);

    objc_autoreleasePoolPop(pool);

    loading = NO;
}
````
call_load_methods循环遍历，首先是调用了class的load方法，然后调用了category的方法
接下来让我们看看call_class_loads的代码实现
````
static void call_class_loads(void)
{
    int i;

    // Detach current loadable list.
    struct loadable_class *classes = loadable_classes;
    int used = loadable_classes_used;
    loadable_classes = nil;
    loadable_classes_allocated = 0;
    loadable_classes_used = 0;

    // Call all +loads for the detached list.
    for (i = 0; i < used; i++) {
        Class cls = classes[i].cls;
        load_method_t load_method = (load_method_t)classes[i].method;
        if (!cls) continue;

        if (PrintLoading) {
            _objc_inform("LOAD: +[%s load]\n", cls->nameForLogging());
        }
        (*load_method)(cls, SEL_load);
    }

    // Destroy the detached list.
    if (classes) free(classes);
}
````
核心方法是 (*load_method)(cls, SEL_load),`typedef void(*load_method_t)(id, SEL);`可以看到load_method是一个函数指针，所以是直接调用了内存地址！

##+initialize


![98779EDD-737F-498D-AD9E-B02508958DDB.png](http://upload-images.jianshu.io/upload_images/6644906-6cb08b0bb9e17df1.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
一样我们通过调用堆栈可以看到系统调用的是_class_initialize方法

##_class_initialize

````
void _class_initialize(Class cls)
{
    assert(!cls->isMetaClass());

    Class supercls;
    bool reallyInitialize = NO;

    // Make sure super is done initializing BEFORE beginning to initialize cls.
    // See note about deadlock above.
    supercls = cls->superclass;
    if (supercls  &&  !supercls->isInitialized()) {
        _class_initialize(supercls);
    }

    // Try to atomically set CLS_INITIALIZING.
    {
        monitor_locker_t lock(classInitLock);
        if (!cls->isInitialized() && !cls->isInitializing()) {
            cls->setInitializing();
            reallyInitialize = YES;
        }
    }

    if (reallyInitialize) {
        // We successfully set the CLS_INITIALIZING bit. Initialize the class.

        // Record that we're initializing this class so we can message it.
        _setThisThreadIsInitializingClass(cls);

        // Send the +initialize message.
        // Note that +initialize is sent to the superclass (again) if
        // this class doesn't implement +initialize. 2157218
        if (PrintInitializing) {
            _objc_inform("INITIALIZE: calling +[%s initialize]",
                         cls->nameForLogging());
        }

        // Exceptions: A +initialize call that throws an exception
        // is deemed to be a complete and successful +initialize.
        @try {
            callInitialize(cls);

            if (PrintInitializing) {
                _objc_inform("INITIALIZE: finished +[%s initialize]",
                             cls->nameForLogging());
            }
        }
        @catch (...) {
            if (PrintInitializing) {
                _objc_inform("INITIALIZE: +[%s initialize] threw an exception",
                             cls->nameForLogging());
            }
            @throw;
        }
        @finally {
            // Done initializing.
            // If the superclass is also done initializing, then update
            //   the info bits and notify waiting threads.
            // If not, update them later. (This can happen if this +initialize
            //   was itself triggered from inside a superclass +initialize.)
            monitor_locker_t lock(classInitLock);
            if (!supercls  ||  supercls->isInitialized()) {
                _finishInitializing(cls, supercls);
            } else {
                _finishInitializingAfter(cls, supercls);
            }
        }
        return;
    }

    else if (cls->isInitializing()) {
        // We couldn't set INITIALIZING because INITIALIZING was already set.
        // If this thread set it earlier, continue normally.
        // If some other thread set it, block until initialize is done.
        // It's ok if INITIALIZING changes to INITIALIZED while we're here,
        //   because we safely check for INITIALIZED inside the lock
        //   before blocking.
        if (_thisThreadIsInitializingClass(cls)) {
            return;
        } else {
            waitForInitializeToComplete(cls);
            return;
        }
    }

    else if (cls->isInitialized()) {
        // Set CLS_INITIALIZING failed because someone else already
        //   initialized the class. Continue normally.
        // NOTE this check must come AFTER the ISINITIALIZING case.
        // Otherwise: Another thread is initializing this class. ISINITIALIZED
        //   is false. Skip this clause. Then the other thread finishes
        //   initialization and sets INITIALIZING=no and INITIALIZED=yes.
        //   Skip the ISINITIALIZING clause. Die horribly.
        return;
    }

    else {
        // We shouldn't be here.
        _objc_fatal("thread-safe class init in objc runtime is buggy!");
    }
}
````
核心方法就是callInitialize接下来让我们看看该方法的实现

##callInitialize
````
void callInitialize(Class cls)
{
    ((void(*)(Class, SEL))objc_msgSend)(cls, SEL_initialize);
    asm("");
}
````
通过该实现我们可以看出其实initialize的本质就是objc_msgSend，所以遵循消息的转发机制。
***
#示例演示
**Animal 父类**
````Objective-c
@implementation Animal
+(void)load{
    NSLog(@"Animal load");
}
+(void)initialize{
    NSLog(@"Animal initialize");
}
@end
````
**Dog 子类**
````Objective-C
+(void)load{
    NSLog(@"Dog load");
}
//分两种情况，一种注释掉该代码，一种打开该代码
//+(void)initialize{
//    NSLog(@"Dog initialize");
//}

**测试代码**
````Objective-C
    Animal *animal = [[Animal alloc] init];
    Animal *animal2 = [[Animal alloc] init];
    Animal *animal3 = [[Animal alloc] init];
    Dog *dog = [[Dog alloc] init];
    Dog *dog2 = [[Dog alloc] init];
````
- 未注释代码时，打印结果
2017-07-31 14:24:47.671 test[53134:5737864] Animal load
2017-07-31 14:24:47.809 test[53134:5737864] Dog load
2017-07-31 14:24:48.112 test[53134:5737864] Animal initialize
2017-07-31 14:24:48.112 test[53134:5737864] Dog initialize
- 注释掉代码时，打印结果
2017-07-31 14:39:03.916 testaa[53659:5814782] Animal load
2017-07-31 14:39:03.917 testaa[53659:5814782] Dog load
2017-07-31 14:39:03.966 testaa[53659:5814782] Animal initialize
2017-07-31 14:39:03.967 testaa[53659:5814782] Animal initialize

通过两次打印看到以下现象：
- 第一次测试load initialize各打印了一次，并且load比initialize提前打印
 **原因：+ load是应用一启动就调动，+ initialize是我们调用该类方法的时候才会调用，而且这两个方法理论上只会调用一次。**
- 第二次测试Animal的initialize打印了两次
 **原因：initialize遵循的是objc_msgSend消息的转发机制，第一次打印是因为我们实例化了Animal并且调用了方法；第二次打印是因为Dog子类没有实现该方法，根据消息转发机制的原理，所以会向上查找父类是否实现了该方法，所以调用了父类的initialize的方法了。所以父类的initialize可能会被调用多次所以建议以下写法：**

```+ (void)initialize {
  if (self == [ClassName self]) {
    // ... do the initialization ...
  }
}
```

我们可以做个以下尝试Dog不继承Animal，然后在Compile Source中将Dog的顺序拖到Animal之前。
如下图：
![D790F51D-98BD-4963-B7A9-D7DEE037FE45.png](http://upload-images.jianshu.io/upload_images/6644906-88388b9eb906803e.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
 **可以观察到Dog的Load在Animal之前打印。所以在没有继承关系的时候Load的调用顺序跟我们的Compile Source的排列顺序有关。有继承关系的，父类一定比子类先调用**

#结论
- +load方法是在程序一启动的时候就会调用，并且在main函数之前，是根据Xcode中Compile Sources的顺序调用的。其内部本质是通过函数**内存地址**的方式实现的。所以在有继承关系的时候子类与父类没有任何关系，不会相互影响。
- +initialize方法是我们在第一次使用该类的时候即调用某个方法的时候系统开始调用 ，是一种懒加载的方式。其内部本质是通过**objc_msgSend**发送消息实现的,因此也拥有objc_msgSend带来的特性,也就是说子类会继承父类的方法实现,而分类的实现也会覆盖元类。

我们常用[Method Swizzling](http://nshipster.com/method-swizzling/)建议一定要在Load方法中实现。
[stackoverflow使用场景讨论](https://stackoverflow.com/questions/30585088/objective-c-cases-when-it-is-useful-to-override-initialize-and-load?lq=1)
