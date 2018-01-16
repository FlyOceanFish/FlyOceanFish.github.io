---
title: iOS超级超级详细介绍GCD # 这是标题
date: 2018-1-16 21:27:00
categories:  # 这里写的分类会自动汇集到 categories 页面上，分类可以多级
- iOS # 一级分类
tags:   # 这里写的标签会自动汇集到 tags 页面上
- iOS
---
# 什么是GCD
Grand Central Dispatch（GCD）是异步执行任务的技术之一。一般将应用程序中记述的线程管理用的代码在系统级中实现。开发者只需要定义想执行的任务并追加到适当的Dispatch Queue中，GCD就能生成必要的线程并计划执行任务。由于线程管理是作为系统的一部分来实现的，因此可统一管理，也可执行任务，这样就比以前的线程更有效率。
# Dispatch Queue

> 将想执行的任务追加到适当的Dispatch Queue

|Dispatch Queue种类 | 说明
|--:|:---:|
|Serial Dispatch Queue|串行队列，任务按照追加顺序处理(FIFO)
|Concurrent Dispatch Queue|并行队列
## 串行队列
![并行队列.jpg](http://upload-images.jianshu.io/upload_images/6644906-eca58dad92d8b9e0.jpg?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
## 并行队列
![1516067701212.jpg](http://upload-images.jianshu.io/upload_images/6644906-8799ef9dc9c97fe3.jpg?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
## 重点
串行队列只有一个线程，并行队列有多个线程。
## 自建Queue

```
dispatch_queue_t queue = dispatch_queue_create("com.test.queue", DISPATCH_QUEUE_CONCURRENT);//创建了一个并行队列

dispatch_queue_t queue = dispatch_queue_create("com.test.queue", DISPATCH_QUEUE_SERIAL);//创建了一个串行队列

```


## 系统Queue
- Main DispatchQueue
在主线程中执行的Dispatch Queue。**因为主线程只有1个，所以`Main Dispatch Queue`是串行队列**。加入到主队列中的任务一定不会生成新的线程，因为主队列必须有且只有一条主线程。
- Global Dispatch Queue
一个所有应用程序都能够使用的**并发队列**。加入到该队列中的任务不一定会生成线程。因为有线程重用的现象

iOS8.0之后的权限

|名称|描述|
|:--:|:--:|
|QOS_CLASS_USER_INTERACTIVE|与用户交互的任务，这些任务通常跟UI级别的刷新相关，比如动画，这些任务需要在一瞬间完成|
|QOS_CLASS_USER_INITIATED| 由用户发起的并且需要立即得到结果的任务，比如滑动scroll view时去加载数据用于后续cell的显示，这些任务通常跟后续的用户交互相关，在几秒或者更短的时间内完成|
|QOS_CLASS_DEFAULT| 优先级介于user-initiated 和 utility，当没有 QoS信息时默认使用，开发者不应该使用这个值来设置自己的任务|
|QOS_CLASS_UTILITY|一些可能需要花点时间的任务，这些任务不需要马上返回结果，比如下载的任务，这些任务可能花费几秒或者几分钟的时间|
|QOS_CLASS_BACKGROUND|这些任务对用户不可见，比如后台进行备份的操作，这些任务可能需要较长的时间，几分钟甚至几个小时|
|QOS_CLASS_UNSPECIFIED|未指定|

# Dispatch Group
dispatch_group是GCD的一项特性，能够把任务分组。调用者可以等待这组任务执行完毕，也可以提供回调函数之后继续往下执行，这组任务完成时，调用者会得到通知。常用场景比如说，下载一个大的文件，分块下载，全部下载完成后再合成一个文件。再比如同时下载多个图片，监听全部下载完后的动作

- 创建group
>dispatch_group_t group = dispatch_group_create();
- 添加任务
  - dispatch_group_async(dispatch_group_t group,
	dispatch_queue_t queue,
	dispatch_block_t block);
  
    将一个任务添加到指定group中
  - dispatch_group_enter(dispatch_group_t group);
  dispatch_group_leave(dispatch_group_t group);
  
    这两个函数同上边一样的效果，不过一定要注意这两个函数必须成对出现！否则这一组任务就永远执行不完。
- 监听任务完成
  - dispatch_group_notify(dispatch_group_t group,
	dispatch_queue_t queue,
	dispatch_block_t block);
  开发者可以传入block，等dispatch group 执行完毕之后，块会在特定的线程上执行，而不阻塞线程。
  - long dispatch_group_wait(dispatch_group_t group, dispatch_time_t timeout);
  
  timeout参数表示函数在等待dispatch group执行完毕后，应该阻塞多久。如果执行dispatch group所需的时间小于timeout，则返回0，否则返回非0值.此参数可以取常量DISPATCH_TIME_FOREVER，这表示函数会一直等着dispatch group 执行完，而不会超时。此方法会阻塞线程。
  
*** 使用dispatch_group_notify***

//这个例子演示将将一个很大的字符串删除指定几个字符串。比如“abcdefghicladedgdsfs”删除"ace"三个字符串
```Objective-C
  int const COUNT = 6;
@interface ViewController ()
{
    NSMutableArray *values;
    dispatch_group_t group;
    dispatch_queue_t queue;
}
@end

@implementation ViewController

- (void)viewDidLoad {
    [super viewDidLoad];
    values = [NSMutableArray array];
    group = dispatch_group_create();
    queue = dispatch_queue_create("com.aa.test", DISPATCH_QUEUE_CONCURRENT);
    [self removeChars:@"abcdefghicladedgdsfs" target:@"ace"];
    dispatch_group_notify(group, queue, ^{
        NSString *str = [values componentsJoinedByString:@""];
        NSLog(@"%@",str);
    });
}
- (void) removeChars:(NSString *)pBuffer target:(NSString *)target {
    for (int i = 0; i<COUNT; i++) {
        if (i==COUNT-1) {
            NSString *str = [pBuffer substringWithRange:NSMakeRange(pBuffer.length/(COUNT-1)*i,pBuffer.length-pBuffer.length/(COUNT-1)*i)];
            [values addObject:str];
        }else{
            NSString *str = [pBuffer substringWithRange:NSMakeRange(pBuffer.length/(COUNT-1)*i, pBuffer.length/(COUNT-1))];
            [values addObject:str];
        }
    }
    for (int i = 0; i<values.count; i++) {
       [self handlerString:values[i] target:target index:i];
    }
}
- (void)handlerString:(NSString *)string target:(NSString *)target index:(int)index{
    dispatch_group_async(group, queue, ^{
        values[index] = [self replace:string target:target];
    });
}
- (NSString *)replace:(NSString *)string target:(NSString *)target{
    for (int i = 0; i<target.length; i++) {
        char temp = [target characterAtIndex:i];
        string = [string stringByReplacingOccurrencesOfString:[NSString stringWithFormat:@"%c", temp] withString:@""];
    }
    return string;
}

@end
```
**使用dispatch_group_enter和dispatch_group_leave**

```Objective-C
dispatch_group_t group =dispatch_group_create();
dispatch_queue_t globalQueue=dispatch_get_global_queue(0, 0);


dispatch_group_enter(group);

//模拟多线程耗时操作
dispatch_group_async(group, globalQueue, ^{
    sleep(3);
    NSLog(@"%@---block1结束。。。",[NSThread currentThread]);
    dispatch_group_leave(group);
});
NSLog(@"%@---1结束。。。",[NSThread currentThread]);

dispatch_group_enter(group);
//模拟多线程耗时操作
dispatch_group_async(group, globalQueue, ^{
    sleep(3);
    NSLog(@"%@---block2结束。。。",[NSThread currentThread]);
    dispatch_group_leave(group);
});
NSLog(@"%@---2结束。。。",[NSThread currentThread]);

dispatch_group_notify(group, dispatch_get_global_queue(0, 0), ^{
    NSLog(@"%@---全部结束。。。",[NSThread currentThread]);
});
```
# `dispatch_async`和`dispatch_sync`
- dispatch_async

`dispatch_async`将指定的Block`异步`的追加到指定的Dispatch Queue中。`dispatch_async`函数不会做任何等待

- dispatch_sync

`dispatch_sync`将指定的Block`同步`的追加到指定的Dispatch Queue。<font color='red'>此时`dispatch_sync`会一直等待Block执行结束之后，才会返回。线程才能接着继续执行其他代码。</font>


> 如dispatch_group_wait函数一样，"等待"意味着当前线程停止。

例：
```
dispatch_sync(dispatch_get_main_queue(), ^{
        NSLog(@"测试2");
    });
```
执行以上代码则会直接崩溃。主要是因为`dispatch_sync`在追加Block的过程中同时在等待，等待意味着当前主线程已经停止，所以主线程无法执行追到到Main Dispatch Queue的Block。


>Serial Dispatch Queue也会引起同样的问题


```
dispatch_queue_t queue = dispatch_queue_create("com.test.queue", NULL);
dispatch_async(queue, ^{
        dispatch_sync(queue, ^{
            NSLog(@"测试");
        });
    });
```
一样的道理，串行队列中只会生成一条线程，而`dispatch_sync`函数使该线程处于`等待`状态即停止状态，所以无法将Block追加到`com.test.queue`这个队列中。

## 总结:

- `dispatch_sync`函数

    - **当前queue是串行队列。** 当前queue上调用sync函数，并且sync函数中指定的queue也是当前queue。需要执行的block被放到当前queue的队尾等待执行，因为这是一个串行的queue，调用sync函数会阻塞当前队列,等待block执行 这个block永远没有机会执行sync函数不返回，所以当前队列就永远被阻塞了，这就造成了死锁。（这就是问题中在主线程调用sync函数，并且在sync函数中传入main_queue作为queue造成死锁的情况）
    - **当前queue是并行队列。** 在并行的queue上面调用sync函数，同时传入当前queue作为参数，并不会造成死锁，因为block会马上被执行，所以sync函数也不会一直等待不返回造成死锁。（并且Block是在当前线程上执行。例如如果是在主线程上调用了`dispatch_sync`,则Block是在主线程上执行的）
    
- `dispatch_async`代表异步**任务**，意思不是一定会生成一条**线程**。如果在MainQueue中执行，则不会生成线程；如果在Global Queue中有可能会生成。因为线程有一个线程池，会重用已经完成任务了的线程。

# 栅栏(dispatch_barrier_sync和dispatch_barrier_async)

作用：与并发队列结合，可以高效率的避免数据竞争的问题

相同点：`dispatch_barrier_sync`和`dispatch_barrier_async`函数功能一样就是在并发队列中将此代码插入的地方上下隔开，如果栅栏一样，两部分不影响。只有上边的并发队列都执行结束之后，下边的并发队列才能够执行。
![1516067701212.jpg](http://upload-images.jianshu.io/upload_images/6644906-3815e1b22e537694.jpg?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

不同点:`dispatch_barrier_sync`代码后边的任务直到`dispatch_barrier_sync`执行完才能被追加到队列中；`dispatch_barrier_async`不用代码执行完，后边的任务也会被追加到队列中。代码上的体现就是`dispatch_barrier_sync`后边的代码不会执行，`dispatch_barrier_async`后边的代码会执行，但是Block不会被执行。

# dispatch_apply
> This function submits a block to a dispatch queue for multiple invocations and waits for all iterations of the task block to complete before returning. If the target queue is a concurrent queue returned by dispatch_get_global_queue, the block can be invoked concurrently, and it must therefore be reentrant-safe. Using this function with a concurrent queue can be useful as an efficient parallel for loop.

该函数指定次数将指定的Block追加到指定的Queue中，并等待全部处理执行结束。


```
dispatch_apply(10, dispatch_get_global_queue(QOS_CLASS_DEFAULT, 0), ^(size_t index) {
        
 });
```
作用：与并发队列结合，可以更有效执行并发任务。

# dispatch_suspend/dispatch_resume

挂起指定的Dispatch Queue
````
dispatch_suspend(queue);
````
恢复指定的Dispatch Queue
```
dispatch_resume(queue);
```
# Dispatch Semaphore
当计数为0时等待，计数为1或大于1时不等待

```
dispatch_semaphore_create(1);//创建1个信号
```

```
dispatch_semaphore_wait(semaphore, DISPATCH_TIME_FOREVER);//当计数值大于1时，或者在待机中计数值大于1时，对该计数减1并且返回。
```

```
dispatch_semaphore_signal(semaphore);//对计数值加1
```
# dispatch_once
单例模式，保证在应用程序中只执行一次指定处理的api。

```
static dispatch_once_t onceToken;
dispatch_once(&onceToken, ^{
        
 });
```

# Dispatch I/O

## 异步串行读取文件
```
NSString *path = [[NSBundle mainBundle] pathForResource:@"iosintroduce" ofType:@"md"];
dispatch_queue_t queue = dispatch_get_global_queue(QOS_CLASS_DEFAULT, 0);
dispatch_io_t pipe_chanel = dispatch_io_create_with_path(DISPATCH_IO_STREAM,[path UTF8String], 0, 0,queue , ^(int error) {
        
    });
size_t water = 1024;
dispatch_io_set_low_water(pipe_chanel, water);
dispatch_io_set_high_water(pipe_chanel, water);
NSMutableData *totalData = [[NSMutableData alloc] init];
    
dispatch_io_read(pipe_chanel, 0, SIZE_MAX, queue, ^(bool done, dispatch_data_t  _Nullable data, int error) {
    if (error == 0) {
        size_t len = dispatch_data_get_size(data);
            if (len > 0) {
                [totalData appendData:(NSData *)data];
        }
    }
    if (done) {
        NSString *str = [[NSString alloc] initWithData:totalData encoding:NSUTF8StringEncoding];
        NSLog(@"%@", str);
    }

  });
```
## 异步并行读取文件


```
NSString *path = [[NSBundle mainBundle] pathForResource:@"iosintroduce" ofType:@"md"];
dispatch_queue_t queue = dispatch_queue_create("queue", DISPATCH_QUEUE_CONCURRENT);
dispatch_fd_t fd = open(path.UTF8String, O_RDONLY);
dispatch_io_t io = dispatch_io_create(DISPATCH_IO_RANDOM, fd, queue, ^(int error) {

    close(fd);

});

off_t currentSize = 0;

long long fileSize = [[NSFileManager defaultManager] attributesOfItemAtPath:path error:nil].fileSize;

size_t offset = 1024*1024;

dispatch_group_t group = dispatch_group_create();

NSMutableData *totalData = [[NSMutableData alloc] initWithLength:fileSize];

for (; currentSize <= fileSize; currentSize += offset) {

    dispatch_group_enter(group);

    dispatch_io_read(io, currentSize, offset, queue, ^(bool done, dispatch_data_t  _Nullable data, int error) {

        if (error == 0) {

            size_t len = dispatch_data_get_size(data);

            if (len > 0) {

                const void *bytes = NULL;

                (void)dispatch_data_create_map(data, (const void **)&bytes, &len);

                [totalData replaceBytesInRange:NSMakeRange(currentSize, len) withBytes:bytes length:len];

            }

        }
        if (done) {

            dispatch_group_leave(group);

        }

    });

}
dispatch_group_notify(group, queue, ^{

    NSString *str = [[NSString alloc] initWithData:totalData encoding:NSUTF8StringEncoding];

    NSLog(@"%@", str);

});
```




- `dispatch_io_create_with_path`通过路径创建一个`dispatch_io_t`。与之相似的也好几个方法，比如`dispatch_io_create_with_io`
- `dispatch_io_set_high_water`和`dispatch_io_set_low_water` 分割文件大小，分别可以设置一次最少读取和一次最多读取多大。
- `dispatch_io_read`读取文件。与之对应的是`dispatch_io_write`将文件存储到指定路径

如果想提高文件读取速度，可以尝试使用Dispatch I/O。
# Dispatch Source
## 倒计时功能
```
dispatch_source_t timer = dispatch_source_create(DISPATCH_SOURCE_TYPE_TIMER, 0, 0, dispatch_get_main_queue());
    
dispatch_source_set_timer(timer, dispatch_time(DISPATCH_TIME_NOW, 15ULL*NSEC_PER_SEC), DISPATCH_TIME_FOREVER, 1ull*NSEC_PER_SEC);
    
dispatch_source_set_event_handler(timer, ^{
        NSLog(@"wake up");
        dispatch_source_cancel(timer);
  });
    
dispatch_source_set_cancel_handler(timer, ^{
        NSLog(@"canceled");
 });
    
dispatch_resume(timer);
```
# dispatch_after
延迟执行任务
```
dispatch_after(dispatch_time(DISPATCH_TIME_NOW, (int64_t)(3 * NSEC_PER_SEC)), dispatch_get_main_queue(), ^{
        
    });
```
最快3秒后执行，最慢3秒+一个runloop循环时间
# dispatch_set_target_queue
改变生成的Dispatch Queue的执行优先级

```
//将myQueue优先级设置为与globalQueue相同

dispatch_queue_t myQueue = dispatch_queue_create("com.test.queue", NULL);
dispatch_queue_t globalQueue = dispatch_get_global_queue(QOS_CLASS_DEFAULT, 0);
dispatch_set_target_queue(myQueue, globalQueue)
```

第一个参数需要变更优先级的Dispatch Queue，第二个参数指定要使用的执行优先级相同优先级的Queue。
# 个人博客
[FlyOceanFish的博客](http://flyoceanfish.top/)

