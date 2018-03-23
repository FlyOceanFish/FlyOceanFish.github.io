---
title: iOS线程同步 # 这是标题
date: 2018-03-23 09:46:00
categories:  # 这里写的分类会自动汇集到 categories 页面上，分类可以多级
- 算法 # 一级分类
tags:   # 这里写的标签会自动汇集到 tags 页面上
- iOS
- 锁
---
# 线程同步
提到多线程大家肯定会提到锁，其实真正应该说的是多线程同步，锁只是多线程同步的一部分。

多线程对于数据处理等方面有着优异的表现和性能，然后多线程如果存在着共享资源的时候，这时候不得不会出现脏数据或者拿不到想要的数据。苹果给我了提供了如下的同步工具
## 原子操作
原子操作其中一个就是我们常见的`atomic`属性修饰符。`atomic`仅仅只是在getter和setter的时候是原子操作,并不是线程安全。可以结合Memory Barriers一起使用确保线程安全
## Memory Barriers 和`Volatile`变量
* Memory Barriers

为了达到最佳性能，编译器通常会讲汇编级别的指令进行重新排序，从而保持处理器的指令管道尽可能的满。作为优化的一部分，编译器可能会对内存访问的指令进行重新排序(在它认为不会影响数据的正确性的前提下)，然而，这并不一定都是正确的，顺序的变化可能导致一些变量的值得到不正确的结果。

Memory Barriers是一种不会造成线程block的同步工具，它用于确保内存操作的正确顺序。Memory Barriers像一道屏障，迫使处理器在其前面完成必须的加载或者存储的操作。
>OSMemoryBarrier();//在需要的地方加上,确保了一定会按照我们书写的顺序执行
* Volatile

告诉编译器，在读取该变量数值的时候，应该直接从内存读取，而不是从寄存器读取。

简单讲就是实现了多线程资源共享的是同一份拷贝。比如一个变量，在多线程中，编译器会为每一个线程缓存一份，从而导致其中另一个线程修改了变量A，而其他线程还是使用了原来的变量A。假如加上`volatile`变量修饰符，则会保证所有的线程是用的同一份数据，其实就是内存中的数据。
>举个例子，为了避免过多的访问内存，编译器会为变量作一个cache,里面会存放上变量的copy, 这样就会提高程序执行效率，而变量如果加了volatile, 那么编译器就不会做这样的优化，每次用到该变量时，都会去内存取一次，从而保证取到的是变量的最新的值。通常下面情况下要用到该变量。

# 锁
## 锁的分类
锁 | 描述
---|---
互斥锁 | 如果多个线程同时竞争一个互斥锁,那么只有一个将被允许访问，其他将被block。
递归锁 | 递归锁是互斥锁上的一个变体。递归锁允许单个线程在释放锁之前多次获取锁。其他线程仍然被阻塞，直到锁的所有者释放锁的次数相同。递归的锁在递归迭代中主要使用，但也可以在需要单独获取锁的多个方法中使用。
读写锁 |读写锁也被称为共享独占锁。这种类型的锁通常用于更大规模的操作，如果保护的数据结构经常被频繁地读取和修改，则可以显著提高性能。在正常运行期间，多个阅读器可以同时访问数据结构。当一个线程想要写入结构时，它会阻塞，直到所有的读取器释放锁为止，这时它会获得锁并更新结构。当一个写线程在等待锁的时候，新的读取器线程阻塞，直到写线程完成。该系统只支持使用POSIX线程的读写锁。有关如何使用这些锁的更多信息，请参见pthread手册页。
分布锁 |分布式锁在流程级别上提供互斥的访问。与真正的互斥锁不同，分布式锁不会阻塞进程或阻止进程运行。它只是报告当锁很忙时，让流程决定如何进行。
自旋锁 |自旋锁定期轮询其锁状态，直到该条件变为真。自旋锁在多处理器系统中最常用，因为锁的等待时间很小。在这种情况下，进行轮询通常比阻塞线程更有效，这涉及到上下文切换和线程数据结构的更新。
双重检查锁定 |双重检查锁定试图采取一个锁的开销减少测试前锁定标准锁

## 锁的注意点
### 适当的使用同步机制
锁和其他的同步工具很影响APP的性能，所以除非我们迫不得已的时候才要使用。
### 同步机制的限制
同步工具只有在应用程序中所有线程一致使用时才有效。如果您创建了一个互斥锁来限制对特定资源的访问，那么在试图操作资源之前，所有的线程都必须获得相同的互斥量。不这样做会破坏互斥锁提供的保护，并且是程序员的错误。
### 将代码放在适当的位置
当我们使用锁和memory barriers的时候，我们要好好思量将代码放在哪里。

```Objective-C
NSLock* arrayLock = GetArrayLock();
NSMutableArray* myArray = GetSharedArray();
id anObject;

[arrayLock lock];
anObject = [myArray objectAtIndex:0];
[arrayLock unlock];//此时锁已经被释放，其他线程尽量可能将anObject删除并释放

[anObject doSomething];
```
上边代码看着确实没问题，我们取数据的时候加锁。但是有个问题。如果`anObject`在调用`doSomething`的时候被其他线程删了，某一时刻被释放了呢？

正确代码如下：
```Objective-C
NSLock* arrayLock = GetArrayLock();
NSMutableArray* myArray = GetSharedArray();
id anObject;

[arrayLock lock];
anObject = [myArray objectAtIndex:0];
[anObject doSomething];

```
### 小心死锁和活锁
避免死锁和活锁的最佳方式就是在一次仅仅获取一个锁
## 锁的详细介绍

![1521703441987.jpg](https://upload-images.jianshu.io/upload_images/6644906-e5c3e52ba1076030.jpg?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

### 互斥锁
* POSIX Mutex Lock
```C
#import <pthread.h>

pthread_mutex_t mutex;
void MyInitFunction()
{
    pthread_mutex_init(&mutex, NULL);
}

void MyLockingFunction()
{
    pthread_mutex_lock(&mutex);
    // Do work.
    pthread_mutex_unlock(&mutex);
}
```
* NSLock
```
BOOL moreToDo = YES;
NSLock *theLock = [[NSLock alloc] init];
...
while (moreToDo) {
    /* Do another increment of calculation */
    /* until there’s no more to do. */
    if ([theLock tryLock]) {
        /* Update display used by all threads. */
        [theLock unlock];
    }
}

```
`NSLock`除了提供`lock`方法，还有 `tryLock` 和 `lockBeforeDate:`两个方法
* @synchronized
```
- (void)myMethod:(id)anObj
{
    @synchronized(anObj)
    {
        // Everything between the braces is protected by the @synchronized directive.
    }
}
```
* NSConditionLock
一个NSConditionLock对象定义了一个可以锁定的<font color='red'>互斥锁</font>，并使用特定的值进行解锁。这个锁可以很好的实现`生产-消费`的模式

生产者线程：
```
id condLock = [[NSConditionLock alloc] initWithCondition:NO_DATA];

while(true)
{
    [condLock lock];
    /* Add data to the queue. */
    [condLock unlockWithCondition:HAS_DATA];
}
```
消费者线程:
```
while (true)
{
    [condLock lockWhenCondition:HAS_DATA];
    /* Remove data from the queue. */
    [condLock unlockWithCondition:(isEmpty ? NO_DATA : HAS_DATA)];

    // Process the data locally.
}

```
### 递归锁
NSRecursiveLock类定义了一个锁，它可以被同一个线程多次获取，而不会导致线程陷入死锁。
>像互斥锁，如果被同一个线程多次获取，但是只释放一次的话就会导致死锁。只有保证一个线程获取n次，同样的释放n次,才能保证不会死锁

```
NSRecursiveLock *theLock = [[NSRecursiveLock alloc] init];

void MyRecursiveFunction(int value)
{
    [theLock lock];
    if (value != 0)
    {
        --value;
        MyRecursiveFunction(value);
    }
    [theLock unlock];
}

MyRecursiveFunction(5);
```
### 条件锁
* NSCondition

NSCondition类提供与POSIX条件相同的语义，但是将所需的锁和条件数据结构封装在一个对象中。结果是一个对象，您可以像一个互斥锁那样锁定，然后像一个条件一样等待。

所以要使用条件锁，要配合`wait`和`signal`使用才能真正发挥出它的作用。

等待线程：
```Objective-C
[cocoaCondition lock];
while (timeToDoWork <= 0)
    [cocoaCondition wait];

timeToDoWork--;

// Do real work here.

[cocoaCondition unlock];
```
信号线程:
```
[cocoaCondition lock];
timeToDoWork++;
[cocoaCondition signal];
[cocoaCondition unlock];
```
* POSIX Conditions

POSIX线程条件锁定要求使用条件数据结构和互斥锁。尽管两个锁结构是分开的，互斥锁在运行时与条件结构密切相关。等待信号的线程应该始终使用相同的互斥锁和条件结构。改变配对会导致错误。
等待线程：
```
pthread_mutex_t mutex;
pthread_cond_t condition;
Boolean     ready_to_go = true;

void MyCondInitFunction()
{
    pthread_mutex_init(&mutex);
    pthread_cond_init(&condition, NULL);
}

void MyWaitOnConditionFunction()
{
    // Lock the mutex.
    pthread_mutex_lock(&mutex);

    // If the predicate is already set, then the while loop is bypassed;
    // otherwise, the thread sleeps until the predicate is set.
    while(ready_to_go == false)
    {
        pthread_cond_wait(&condition, &mutex);
    }

    // Do work. (The mutex should stay locked.)

    // Reset the predicate and release the mutex.
    ready_to_go = false;
    pthread_mutex_unlock(&mutex);
}
```
信号线程：
```
void SignalThreadUsingCondition()
{
    // At this point, there should be work for the other thread to do.
    pthread_mutex_lock(&mutex);
    ready_to_go = true;

    // Signal the other thread to begin work.
    pthread_cond_signal(&condition);

    pthread_mutex_unlock(&mutex);
}
```
### 自旋锁

何谓自旋锁？它是为实现保护共享资源而提出一种锁机制。其实，自旋锁与互斥锁比较类似，它们都是为了解决对某项资源的互斥使用。无论是互斥锁，还是自旋锁，在任何时刻，最多只能有一个保持者，也就说，在任何时刻最多只能有一个执行单元获得锁。但是两者在调度机制上略有不同。对于互斥锁，如果资源已经被占用，资源申请者只能进入睡眠状态。但是自旋锁不会引起调用者睡眠，如果自旋锁已经被别的执行单元保持，调用者就一直循环在那里看是否该自旋锁的保持者已经释放了锁，"自旋"一词就是因此而得名。
* OSSpinLock
```
// 初始化
spinLock = OS_SPINKLOCK_INIT;
// 加锁
OSSpinLockLock(&spinLock);
// 解锁
OSSpinLockUnlock(&spinLock);
```
>OSSpinLock有着优异的性能表现，然而在高并发执行(冲突概率大，竞争激烈)的时候，又或者代码片段比较耗时(比如涉及内核执行文件io、socket、thread等)，就容易引发CPU占有率暴涨的风险，因此更适用于一些简短低耗时的代码片段；

<font color='red'>由于苹果的新系统对于线程优先级的修改，导致了OSSpinLock会出现优先级反转的问题</font>
* os_unfair_lock
自旋锁已经不再安全，然后苹果新出了 os_unfair_lock_t。（<font color='red'>首先它不是自旋锁，其次网上都说它解决了优先级翻转问题，有点误导，这样说不对。由于`OSSpinLock`比较高效，所以苹果就另外实现了`os_unfair_lock_t`,锁的效果同样很高效，并且实现思路与`OSSpinLock`有一部分相同，但是不是自旋锁。</font>）
```
os_unfair_lock_t unfairLock = &(OS_UNFAIR_LOCK_INIT);
os_unfair_lock_lock(unfairLock);
//do something
os_unfair_lock_unlock(unfairLock);
```
---
>备注：因为这篇是线程的锁，GCD通常叫做任务，虽然底层也是有线程。而且GCD的信号与锁效果虽然相同，但是本质上还是有所区别的。不过可以看我的这篇文章[iOS超级超级详细介绍GCD](http://flyoceanfish.top/2018/01/16/iOS%E8%AF%A6%E8%A7%A3GCD/)
