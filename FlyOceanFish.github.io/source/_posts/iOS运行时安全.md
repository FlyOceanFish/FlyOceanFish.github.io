---
title: iOS运行时安全 # 这是标题
date: 2017-11-03 09:00:00
categories:  # 这里写的分类会自动汇集到 categories 页面上，分类可以多级
- iOS # 一级分类
- 安全
tags:   # 这里写的标签会自动汇集到 tags 页面上
- iOS
- 安全
---
## 暗桩和自杀分支

添加一些直观看上去有用的迷惑的方法到类中，并且看起来比较有用。

```Objective-C
- (BOOL)userIsAuthenticated:(BOO)auth{
  /* 无提示地擦除加密密钥 */
  /* 无提示地擦除用户数据 */
  /* 无提示地禁用所有认证 */
}
```
自杀分支不仅能够擦除用户数据，还可以混淆攻击者让他放弃攻击

## 进程调试检测

当一个应用程序被调试的时候，内核会自动设置一个进程状态标志表示进程正在被调试。我们可以检测这个标志，一旦知道自己程序被调试，我们就可以做一些相应的安全措施了。如退出程序、禁用功能等等。

```Objective-C
#define DEBUGGER_CHECK {\
size_t size = sizeof(struct kinfo_proc);\
struct kinfo_proc info;\
int ret,name[4];\
\
memset(&info, 0, sizeof(struct kinfo_proc));\
name[0] = CTL_KERN;\
name[1] = KERN_PROC;\
name[2] = KERN_PROC_PID;\
name[3] = getpid();\
\
if ((ret = (sysctl(name, 4, &info, &size, NULL, 0)))) {\
    exit(EXIT_FAILURE);\
}\
if (info.kp_proc.p_flag&P_TRACED) {\
    NSLog(@"正在调试");\
    /*代码调试在这里做出响应*/\
}\
}\

```

## 阻挡调试器

保证我们程序不被调试，这个只能说增加了难度。如果一个专业的可以通过解密应用程序永久移除它等等的手段。所以我们要越狱检测、调试检测等等相互辅助才能达到最好的效果。

```
#import <sys/ptrace.h>

ptrace(PT_DENY_ATTACH,0,0,0);
```
但是iPhone的环境中没有开发这个ptrace.h文件，所以只能使用dlopen来拿到

```Objective-C
#import <dlfcn.h>
#import<sys/types.h>

typedef int(*ptrace_ptr_t)(int _request, pid_t _pid, caddr_t _addr, int _data);

#if!defined(PT_DENY_ATTACH)

#define PT_DENY_ATTACH 31

#endif  // !defined(PT_DENY_ATTACH)

void disable_gdb() {

    void* handle = dlopen(0, RTLD_GLOBAL |RTLD_NOW);

    ptrace_ptr_t ptrace_ptr = dlsym(handle,"ptrace");

    ptrace_ptr(PT_DENY_ATTACH, 0, 0, 0);

    dlclose(handle);

}
```
在工程的main.m中增加以下代码
```Objective-C
int main(int argc, char *argv[]) {
    /**防止GDB挂起*/
#ifndef DUBUG
    disable_gdb();
#endif
    @autoreleasepool {
        return UIApplicationMain(argc, argv, nil,NSStringFromClass([AppDelegate class]));

    }

}
```
## 运行时库类完整性检查

### 检查内存地址空间

恶意代码的注入，都是需要加载到内存地址空间中，所以可以检查方法的内存位置

**通过dladdr函数检查**

```Objective-C
#import <objc/runtime.h>
#import <dlfcn.h>

static int inline check_func(const char *cls,const char *sel){
    Dl_info info;
    IMP imp = class_getMethodImplementation(objc_getClass(cls), sel_registerName(sel));
    printf("pointer %p",imp);
    if (dladdr(imp,&info)) {
        if (strcmp(info.dli_fname,"System/Library/Frameworks/Foundation.framework/Foundation")||
            strncmp(info.dli_sname, "-[NSURLConnection init]", 22)) {
            printf("Danger");
            exit(0);
        }
        printf("相同");
    }else{
        printf("error:不相同");
        exit(0);
    }
    return 1;
}
```
**使用：**
```
check_func("NSURLConnection", "initWithRequest:delegate:");
```
我们也可以通过runtime来实现验证类中所有的方法

**注意点**
* 函数命名未无法理解的函数
* 在不同位置检查类。因为攻击者可以在任意点上将代码注入
* 考虑检查通用类。比如NSStrin个、NSData等。这些类可以很轻易的劫持大量内存信息

## 内联函数
`inline或__attribute__(always_inline)`

内联函数式编译器将函数功能**插入到每处代码**被调用的地方，所以让攻击者看不到内联函数的调用。<font color=red>(声明内联要用static)</font>它是增加文件的体积来提高程序性能。因为内联函数是没有函数调用的开销，可以有效提高性能；它也能够有效的复用代码。

## 反汇编复杂化
