---
title: iOS中加密、解密 # 这是标题
date: 2017-11-01 15:43:00
categories:  # 这里写的分类会自动汇集到 categories 页面上，分类可以多级
- iOS # 一级分类
- 安全
tags:   # 这里写的标签会自动汇集到 tags 页面上
- iOS
- 加密
- 安全
---
# 对称加密
## 无状态加密

> `#import <CommonCrypto/CommonCrypto.h>`

分组密码(块加密)即是无状态加密，加密之后除了密文其他信息都会丢失

````Objective-C
#import <CommonCrypto/CommonCrypto.h>

CCCrypt(CCOperation op,
        <CCAlgorithm alg>,
        <CCOptions options>,
        <const void *key>,
        <size_t keyLength>,
        <const void *iv>,
        <const void *dataIn>,
        <size_t dataInLength>,
        <void *dataOut>,
        <size_t dataOutAvailable>,
        <size_t *dataOutMoved>)
````
## 有状态加密

> `#import <CommonCrypto/CommonCrypto.h>`

流密码主要用于大型或流式集合这些难以一次性加密的情况，操作速度快。流密码称为有状态加密，因为他们
知道加密处理的位置

* 创建CCCryptor、CCCryptorRef

    CCCryptorRef 贯穿于整个加密过程，所以有状态其实也是主要因为这个参数

````Objective-C
CCCryptorCreate(CCOperation op,
                CCAlgorithm alg,    
                CCOptions options,
                const void *key,
                size_t keyLength,
                const void *iv,
                CCCryptorRef *cryptorRef)
````
* 获取输出数据的最大长度
````Objective-C
size_t CCCryptorGetOutputLength(
    CCCryptorRef cryptorRef,
    size_t inputLength,
    bool final)
````
* 加密处理update写入缓存区

````Objective-C
CCCryptorUpdate(CCCryptorRef cryptorRef,
                const void *dataIn,
                size_t dataInLength,
                void *dataOut,
                size_t dataOutAvailable,
                size_t *dataOutMoved)

````

* 刷新所有数据，所有输出被写入

````Objective-C
CCCryptorFinal(CCCryptorRef cryptorRef,
   void *dataOut,
    size_t dataOutAvailable,
     size_t *dataOutMoved)
````
* 释放

````Objective-C
CCCryptorStatus CCCryptorRelease(
    CCCryptorRef cryptorRef)
````

## 主秘钥加密

KDF(key derivation function) 秘钥生成函数

目前常见的不可逆加密算法有以下几种：

* 一次MD5（使用率很高）
* 将密码与一个随机串进行一次MD5
* 两次MD5，使用一个随机字符串与密码的md5值再进行一次md5，使用很广泛
* PBKDF2算法
* bcrypt

>PBKDF2简单而言就是将salted hash进行多次重复计算，这个次数是可选择的。如果计算一次所需要的时间是1微秒，那么计算1百万次就需要1秒钟。假如攻击一个密码所需的rainbow table有1千万条，建立所对应的rainbow table所需要的时间就是115天。这个代价足以让大部分的攻击者忘而生畏。

> `#import <CommonCrypto/CommonKeyDerivation.h>`

````Objective-C
- (NSData*)generateSalt256 {
    unsigned char salt[32];
    for (int i=0; i<32; i++) {
        salt[i] = (unsigned char)arc4random();
    }
    return [NSData dataWithBytes:salt length:32];
}

...

// Make keys!
NSString* myPass = @"MyPassword1234";
NSData* myPassData = [myPass dataUsingEncoding:NSUTF8StringEncoding];
NSData* salt = [self generateSalt256];

// How many rounds to use so that it takes 0.1s ?
int rounds = CCCalibratePBKDF(kCCPBKDF2, myPassData.length, salt.length, kCCPRFHmacAlgSHA256, 32, 100);

// Open CommonKeyDerivation.h for help
unsigned char key[32];
CCKeyDerivationPBKDF(kCCPBKDF2, myPassData.bytes, myPassData.length, salt.bytes, salt.length, kCCPRFHmacAlgSHA256, rounds, key, 32);
````

## 地理位置加密

通过经纬度与PBKDF2结合，大大增加了安全性，降低了破解可能性

## 拆分服务器秘钥

一半口令存储在用户设备上，另一半存储到服务器上，只有同时用这两个秘钥才能解密

# 内存安全

* NSData内存清除

 `memset([myData bytes],0,[myData length])`

* NSString内存清除

由于NSString对象我们使用时都是数据的一个副本，所以用`CFStringGetCStringPtr`函数获取数据指针，以此用来清除

````Objective-C
unsigned char *text = (unsigned char *)CFStringGetCStringPtr((CFStringRef)myString,
CFStringGetSystemEncoding());
memset(text, 0, [myString length]);
NSLog(@"%s",[myString UTF8String]);
````

# 公钥加密体系(非对称加密)

`Security.framework`

只支持从标准证书文件(cer, crt)中读取公钥

***RSA***

````Objective-C
//生成公钥和私钥的方法
OSStatus SecKeyGeneratePair(CFDictionaryRef parameters, SecKeyRef *publicKey,
SecKeyRef *privateKey) __OSX_AVAILABLE_STARTING(__MAC_10_7, __IPHONE_2_0);

//加密方法
OSStatus SecKeyEncrypt(
    SecKeyRef           key,
    SecPadding          padding,
    const uint8_t      *plainText,
    size_t              plainTextLen,
    uint8_t             *cipherText,
    size_t              *cipherTextLen)
__OSX_AVAILABLE_STARTING(__MAC_10_7, __IPHONE_2_0);

//解密方法
OSStatus SecKeyDecrypt(
     SecKeyRef   key,    /* Private key */
     SecPadding  padding,	/*kSecPaddingNone,kSecPaddingPKCS1,kSecPaddingOAEP */
     const uint8_t  *cipherText,
     size_t  cipherTextLen,		/* length of cipherText */
     uint8_t *plainText,
     size_t  *plainTextLen)		/* IN/OUT */
__OSX_AVAILABLE_STARTING(__MAC_10_7, __IPHONE_2_0);
````
