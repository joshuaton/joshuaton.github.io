---
layout: post
title: "__bridge __bridge\_retained __bridge\_transfer的区别"
date: 2019-08-03 10:25:04 +0800
comments: true
categories: ios
---
iOS开发中，经常会接触到两种对象，Objective-C对象和Core Foundation对象，他们之间有所不同，可以互相转换。最大的不同之处在于，在ARC模式下，前者不用开发者手动管理内存，后者需要开发者手动管理内存，即调用CFRelease方法释放对象，否则会造成内存泄漏。转换的话主要会用到以下3个方法：

* __bridge, 
* __bridge\_retained 
* __bridge\_transfer

__bridge可以用于OC对象和CF对象互转，例如

```objective-c
NSObject *obj = [[NSObject alloc] init];    //retain count 1
CFTypeRef cfObj1 = (__bridge CFTypeRef)obj; //retain count 1
NSObject *obj1 = (__bridge id)cfObj1;       //retain count 2
```
在这种转换方式下，如果是OC对象转换成CF对象，引用计数不变。如果是CF对象转换成OC对象，因为OC对象的默认修饰符是__strong，引用计数会+1，即以下两种写法是一样的。

```objective-c
NSObject *obj1 = (__bridge id)cfObj1;
NSObject __strong *obj1 = (__bridge id)cfObj1;
```

__bridge\_retained用于OC对象转换为CF对象，例如

```objective-c
NSObject *obj = [[NSObject alloc] init];                //retain count 1
CFTypeRef cfObj1 = (__bridge_retained CFTypeRef)obj;    //retain count 2

//等价写法
NSObject *obj = [[NSObject alloc] init];                //retain count 1
CFTypeRef cfObj1 = (CFTypeRef)CFBridgingRetain(obj);    //retain count 2
```
这种情况下，obj的引用计数会+1，obj的释放不会影响到cfObj1的使用

__bridge\_transfer用于CF对象转换为OC对象，例如

```objective-c
NSObject *obj = [[NSObject alloc] init];                //retain count 1
CFTypeRef cfObj1 = (__bridge_retained CFTypeRef)obj;    //retain count 2
NSObject *obj1 = (__bridge_transfer id)cfObj1;          //retain count 2

//等价写法
NSObject *obj = [[NSObject alloc] init];                  //retain count 1
CFTypeRef cfObj1 = (__bridge_retained CFTypeRef)obj;      //retain count 2
NSObject *obj1 = (NSObject *)CFBridgingRelease(cfObj1);   //retain count 2

```

最后来做个练习，看下以下代码输出什么

```objective-c
NSObject *obj = [[NSObject alloc] init];                //retain count 1
CFTypeRef cfObj = (__bridge_retained CFTypeRef)obj;     //retain count 2
CFTypeRef cfObj1 = (__bridge CFTypeRef)obj;             //retain count 2
CFTypeRef cfObj2 = (__bridge_retained CFTypeRef)obj;    //retain count 3

NSObject *obj1 = (__bridge_transfer id)cfObj1;          //retain count 3
NSObject *obj2 = (__bridge id)cfObj2;                   //retain count 4
    
NSLog(@"obj retainCount %ld", (long)CFGetRetainCount(cfObj));

NSString *str = [NSString stringWithFormat:@"testStr"];
CFStringRef cfStr = (__bridge CFStringRef)str;
NSLog(@"str retainCount %ld", (long)CFGetRetainCount(cfStr));   //retain count 9223372036854775807 = 0x7FFFFFFFFFFFFFFF
```

答案是

```
obj retainCount 4
str retainCount 9223372036854775807
```

str特殊一点，会被当做是字符串常量，retainCount是一个最大值，保证不会被系统回收。


