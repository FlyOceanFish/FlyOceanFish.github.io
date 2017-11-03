---
title: iOS越狱检查 # 这是标题
date: 2017-11-05 15:00:00
categories:  # 这里写的分类会自动汇集到 categories 页面上，分类可以多级
- iOS # 一级分类
- 安全
tags:   # 这里写的标签会自动汇集到 tags 页面上
- iOS
- 安全
---

## 沙盒完整性检查

通过`fork()`函数检测。如果沙盒被越狱工具破坏或者是程序在沙盒外运行，那么`fork()`函数将会执行成功。如果沙盒仍处于激活状态，你的程序就会执行失败，这就说明沙盒没有被篡改。

```Objective-C
int result = fork();
if(!result)
    exit(0);
if (result>=0) {
    //沙盒被破坏，处于越狱环境
}
```

## 文件系统检测

### 越狱文件是否存在
```Objective-C
#import <sys/stat.h>

struct stat s;
int is_jailbroken = stat("/Application/Cydia.app", &s) == 0;
```
我们同时也可以检测其他文件列表

* /Application/Cydia.app Cydia安装目录
* /var/cache/apt apt仓库的路径
* /var/lib/apt apt仓库相关的数据
* /var/lib/cydia cydia相关的数据文件
* /var/log/syslog syslog文件
* /var/tmp/cydia.log Cydia运行时的临时日志记录
* /bin/bash /bin/sh bash总段交互程序
* /usr/sbin/sshd SSH终端
* /usr/libexec/ssh-keysign SSH的密钥签名工具
* /etc/ssh/sshd_config sshd的配置文件
* /etc/apt apt配置文件路径

### /etc/fstab 文件大小

fstab文件包含文件系统挂载点，这个文件通常被越狱工具替换用于将root分区读写打开。我们程序不能读取该文件，但是可以使用stat函数查看文件大小。通常80B，越狱的只有65B。也有可能随着系统变化的
```Objective-C
#import <sys/stat.h>

struct stat s;
stat("/etc/fstab", &s);
s.st_size;
```

### 符号链接检测

通过lstat函数，可以检查/Applications是一个目录还是一个符号链接，如果是一个链接，可以确认设备已经越狱。

```Objective-C
struct stat s;
if (lstat("/Applications", &s)!=0) {
    if (s.st_mode&S_IFLNK) {
        /*设备被越狱*/
        exit(-1);
    }
}
```

### 分页执行检查
内存分页不能标记为可执行属性，但是越狱可以修改这一限制。通过vm_protect函数检测内核完整性，当内核未被修改时，该函数会调用失败

```Objective-C
#import <sys/stat.h>
#import <mach/mach_init.h>
#import <mach/vm_map.h>


void *mem = malloc(getpagesize()+15);
void *ptr = (void *)(((uintptr_t)mem+15) & ~0x0F);
vm_address_t pagePtr = (uintptr_t)ptr/getpagesize()*getpagesize();
int is_jailbroken = vm_protect(mach_task_self(), (vm_address_t)pagePtr, getpagesize(), FALSE, VM_PROT_READ|VM_PROT_WRITE|VM_PROT_EXECUTE) == 0;
```
