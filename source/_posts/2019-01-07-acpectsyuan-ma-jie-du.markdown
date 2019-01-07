---
layout: post
title: "Aspects源码解读"
date: 2019-01-07 15:25:12 +0800
comments: true
categories: ios
---
Aspects是iOS面向切面编程的第三方库，它可以在不改变原有代码的情况下，在任意函数之前或之后插入代码，也可以替换掉函数原有的代码。它的原理是基于iOS的runtime，这篇文章对Aspects进行源码解读，并阐述其原理。

##调用方式
首先我们下载官方demo，从入口代码开始看：

```objective-c
AspectsViewController *aspectsController = [AspectsViewController new];
[aspectsController aspect_hookSelector:@selector(buttonPressed:) withOptions:0 usingBlock:^(id info, id sender) {
    NSLog(@"Button was pressed by: %@", sender);
} error:NULL];
```

这段代码就是Aspects的调用方式之一，表示在对象aspectsController的buttonPressed函数执行之后，再执行block里的代码，打印一行日志。withOptions的参数写的0，这里是一个枚举值，可以控制block代码怎样执行，具体的定义如下：

```objective-c
typedef NS_OPTIONS(NSUInteger, AspectOptions) {
    AspectPositionAfter   = 0,            /// Called after the original implementation (default)
    AspectPositionInstead = 1,            /// Will replace the original implementation.
    AspectPositionBefore  = 2,            /// Called before the original implementation.
    
    AspectOptionAutomaticRemoval = 1 << 3 /// Will remove the hook after the first execution.
};
```

##hook过程
我们从入口函数进入开始跟踪代码，最后发现无论是对实例方法还是类方法进行hook，都会调用aspect_add函数，省略了一些无关代码后如下：

```objective-c
static id aspect_add(id self, SEL selector, AspectOptions options, id block, NSError **error) {
    __block AspectIdentifier *identifier = nil;
    AspectsContainer *aspectContainer = aspect_getContainerForObject(self, selector);
    identifier = [AspectIdentifier identifierWithSelector:selector object:self options:options block:block error:error];
    if (identifier) {
        [aspectContainer addAspect:identifier withOptions:options];

        // Modify the class to allow message interception.
        aspect_prepareClassAndHookSelector(self, selector, error);
    }
    return identifier;
}
```

这段代码做了两件事情。

首先生成AspectIdentifier，然后将AspectIdentifier加入到AspectsContainer中。AspectIdentifier的定义如下，它描述了一个Ascpect切片代码的信息。

```objective-c
@interface AspectIdentifier : NSObject
@property (nonatomic, assign) SEL selector;
@property (nonatomic, strong) id block;
@property (nonatomic, strong) NSMethodSignature *blockSignature;
@property (nonatomic, weak) id object;
@property (nonatomic, assign) AspectOptions options;
@end
```

AspectsContainer的定义如下，它负责容纳AspectIdentifier，可以在before，instead，after数组里放入多个AspectIdentifier，从名称可以看出这些AspectIdentifier所执行的时机。AspectsContainer将在后边取出并执行。

```objective-c
@interface AspectsContainer : NSObject
@property (atomic, copy) NSArray *beforeAspects;
@property (atomic, copy) NSArray *insteadAspects;
@property (atomic, copy) NSArray *afterAspects;
@end
```

其次调用aspect_prepareClassAndHookSelector函数，这是最关键的部分：

```objective-c
static void aspect_prepareClassAndHookSelector(NSObject *self, SEL selector, NSError **error) {
    Class klass = aspect_hookClass(self, error);
    Method targetMethod = class_getInstanceMethod(klass, selector);
    IMP targetMethodIMP = method_getImplementation(targetMethod);
    if (!aspect_isMsgForwardIMP(targetMethodIMP)) {
        // Make a method alias for the existing method implementation, it not already copied.
        const char *typeEncoding = method_getTypeEncoding(targetMethod);
        SEL aliasSelector = aspect_aliasForSelector(selector);
        if (![klass instancesRespondToSelector:aliasSelector]) {
            __unused BOOL addedAlias = class_addMethod(klass, aliasSelector, method_getImplementation(targetMethod), typeEncoding);
        }
        // We use forwardInvocation to hook in.
        class_replaceMethod(klass, selector, aspect_getMsgForwardIMP(self, selector), typeEncoding);
    }
}
```

这个函数分为两部分，第2行aspect_hookClass和后边的部分。我们先来看aspect_hookClass函数，省略后的代码如下。

```objective-c
static Class aspect_hookClass(NSObject *self, NSError **error) {
	Class statedClass = self.class;
	Class baseClass = object_getClass(self);
	NSString *className = NSStringFromClass(baseClass);

    // Default case. Create dynamic subclass.
	const char *subclassName = [className stringByAppendingString:AspectsSubclassSuffix].UTF8String;
	Class subclass = objc_getClass(subclassName);

	if (subclass == nil) {
		subclass = objc_allocateClassPair(baseClass, subclassName, 0);
		aspect_swizzleForwardInvocation(subclass);
		aspect_hookedGetClass(subclass, statedClass);
		aspect_hookedGetClass(object_getClass(subclass), statedClass);
		objc_registerClassPair(subclass);
	}

	object_setClass(self, subclass);
	return subclass;
}
```

第11行代码通过运行时的函数objc_allocateClassPair定义了一个新的子类。如果是demo执行到这里的话，生成的子类叫AspectsViewController_Aspects。第12行，将子类的forwardInvocation替换为了自定义的实现函数\_\_ASPECTS\_ARE\_BEING\_CALLED\_\_。第18行，将AspectsViewController实例的isa指针指向了子类AspectsViewController_Aspects。

接着，我们继续看aspect_prepareClassAndHookSelector函数的后半部分。第10行在AspectsViewController_Aspects类添加了一个方法aliasSelector，demo中就是aspect_buttonPressed，它的实现指向了原来AspectsViewController类的buttonPressed的实现。第13行，将AspectsViewController_Aspects类的buttonPressed实现指向了_objc_msgForward，这样调用就会启动oc的消息转发机制。

到这里，Aspects这个库的关键初始化流程就执行完了，我们用下边这个图来描述下当前类和方法实现之间的关系。

[![](https://jason5.cn/images/Aspects.png)](https://jason5.cn/images/Aspects.png)

Aspects的实现为什么要生成一个原有类的子类，个人理解是为了对原有类产生的影响尽可能小。

##hook后的执行流程
hook完成后，我们来看下hook后代码的执行流程。

**这一段很重要！！！**往AspectsViewController实例发送buttonPressed消息的时候，首先应该去查找实例所对应的类的方法列表，由于AspectsViewController的isa指向了AspectsViewController_Aspects类，就会去AspectsViewController_Aspects类中查找，结果是查找不到buttonPressed实现，然后会去查找父类AspectsViewController的方法列表，这时候查找到了buttonPressed的实现，但是实现是指向了_msg_forward，这样就进入了消息转发流程。按照消息转发流程，系统会调用AspectsViewController_Aspects类的forwardInvocation方法，forwardInvocation方法被我们替换成了自定义实现\_\_ASPECTS\_ARE\_BEING\_CALLED\_\_，最终就进入了这个方法。

\_\_ASPECTS\_ARE\_BEING\_CALLED\_\_的省略后的代码如下：

```objective-c
// This is the swizzled forwardInvocation: method.
static void __ASPECTS_ARE_BEING_CALLED__(__unsafe_unretained NSObject *self, SEL selector, NSInvocation *invocation) {
    SEL originalSelector = invocation.selector;
    SEL aliasSelector = aspect_aliasForSelector(invocation.selector);
    invocation.selector = aliasSelector;
    AspectsContainer *objectContainer = objc_getAssociatedObject(self, aliasSelector);
    AspectsContainer *classContainer = aspect_getContainerForClass(object_getClass(self), aliasSelector);
    AspectInfo *info = [[AspectInfo alloc] initWithInstance:self invocation:invocation];
    NSArray *aspectsToRemove = nil;

    // Before hooks.
    aspect_invoke(classContainer.beforeAspects, info);
    aspect_invoke(objectContainer.beforeAspects, info);

    // Instead hooks.
    BOOL respondsToAlias = YES;
    if (objectContainer.insteadAspects.count || classContainer.insteadAspects.count) {
        aspect_invoke(classContainer.insteadAspects, info);
        aspect_invoke(objectContainer.insteadAspects, info);
    }else {
        Class klass = object_getClass(invocation.target);
        do {
            if ((respondsToAlias = [klass instancesRespondToSelector:aliasSelector])) {
                [invocation invoke];
                break;
            }
        }while (!respondsToAlias && (klass = class_getSuperclass(klass)));
    }

    // After hooks.
    aspect_invoke(classContainer.afterAspects, info);
    aspect_invoke(objectContainer.afterAspects, info);
}
```
第7行，对于hook的实例方法，先拿到之前设置的切片代码信息，存储在classContainer里。第24行，通过invocation调用AspectsViewController_Aspects的aspect_buttonPressed方法，由于这个方法已经指向了原来的实现buttonPressed，所以就调用了原始的代码。在这之后，如果Container里有afterAspects，就调用切片的block。beforeAspects同理。

到此为止，就实现了在原来的实例方法执行后，再执行hook插入的block代码。

##总结
ios的runtime是黑魔法，运用起来可以做很多强大的功能。总的来讲，Aspects利用了ios的method swizzling和消息转发机制forwordInvocation，实现了对函数进行切片hook。


