---
layout: post
title: "iOS 中的 NSProxy"
date: 2019-06-10 20:03:28 +0800
comments: true
categories: ios
---

在日常开发中，NSObject 经常会被使用到。但是 NSProxy 却很少用。这个类顾名思义，是用来做代理的，任何消息都可以对它发送，在它内部，再指向具体的实现。

![](https://raw.githubusercontent.com/joshuaton/img/master/20190610195500.png)

如上图所示，调用者可以给 NSProxy 发送消息，而不用关心内部实现，NSProxy 则可以根据具体的 SEL 去调用真正的实现着 ClassA 或者 ClassB。需要说明的是，NSProxy 不能直接使用，需要自己写一个类继承它。

接下来我们来看实例代码。

先定义一个类 MyProxy 继承自 NSProxy：

```objectivec
//MyProxy.h
#import <Foundation/Foundation.h> 

@interface MyProxy : NSProxy
-(instancetype)init;
@end
```

```objectivec
//MyProxy.m
#import "MyProxy.h"
#import <objc/runtime.h>
#import "Real.h"

@interface MyProxy()

@property (nonatomic, strong) Real *real;

@end
@implementation MyProxy

-(instancetype)init{
    self.real = [[Real alloc] init];
    return self;
}

- (void)forwardInvocation:(NSInvocation *)anInvocation{
    SEL sel = anInvocation.selector;
    if([self.real respondsToSelector:sel]){
        [anInvocation invokeWithTarget:self.real];
    }
}

- (NSMethodSignature *)methodSignatureForSelector:(SEL)sel{
    if([self.real respondsToSelector:sel]){
        return [self.real methodSignatureForSelector:sel];
    }
    return NULL;
}

@end
```

MyProxy 需要实现两个方法 forwardInvocation: 和 methodSignatureForSelector:

在 forwardInvocation: 方法中，将真实的实现转发给了 Real 的实例，由其去实现。

```objectivec
//Real.h
#import <Foundation/Foundation.h>

@interface Real : NSObject
-(void)hello;
@end
```

```objectivec
//Real.m
#import "Real.h"

@implementation Real

-(void)hello{
    NSLog(@"hello11");
}

@end
```

下边是调用方代码：

```objectivec
MyProxy *proxy = [[MyProxy alloc] init];
[proxy performSelector:@selector(hello)];
```

运行后，在控制台输出 hello11，表明 Real 实例的 hello 方法得到执行。

NSProxy 可以用来实现双继承，更多好玩的玩法可以去探索。
