---
layout: post
title: "通过源码理解Autorelease Pool原理"
date: 2019-06-01 20:52:49 +0800
comments: true
categories: ios
---

## 1. Autorelease Pool 是什么

iOS 的内存管理使用引用计数机制。当对象被初始化或者被强引用赋值时，对象的引用计数 +1，当对象离开所在函数作用域或者被设置为 nil 后，引用计数 -1。当对象的引用计数为 0 时，操作系统会释放掉对象所占用的内存。

我们先来看一下这段代码：

```objectivec
-(NSString *)getStr{
    NSString *str = [NSString stringWithFormat:@"12"];
    return str;
}
```

在 getStr 执行完后，str 的作用域已经结束，str 的引用计数为 0，应该马上被系统回收。那么问题就出现了，str 是作为函数的返回给调用者的，被回收后调用者拿到的对象就是nil了，明显不符合调用者的预期。这时候 Autorelease Pool 就派上用场了，当 getStr 函数结束时，str 并没有进行引用计数 -1 操作，而是将 str 放入了 Autorelease Pool。Autorelease Pool 是一个可以存放多个对象指针的对象池，当 Autorelease Pool 被销毁时，会对所有 Autorelease Pool 中的对象执行引用计数 -1 操作，这时候才会回收 str。相当于放入 Autorelease Pool 的对象被延迟释放了。这样的机制能够保证调用者能够正常拿取到 str 的引用。

那么 Autorelease Pool 是什么时候被创建和销毁的呢？对于 ARC 来讲，大多数情况下，是不需要开发人员自己创建和销毁 Autorelease Pool 的（后面再讲少数情况）。Autorelease Pool 是在 Runloop 的一次循环中，被创建和释放的，是系统自己做的，开发人员不能控制创建和释放的时机，所以开发人员也不能知道 Autorelease Pool 里的对象什么时候被释放的。下边是网上看到的一个图，说明了 Autorelease Pool 创建和释放的时机。

![](https://raw.githubusercontent.com/joshuaton/img/master/20190601222200.png)



## 2. AutoRelease Pool如何使用

在 ARC 情况下，AutoRelease Pool 的使用非常简单，以 iOS 工程里的 main.m 代码为例：

```objectivec
int main(int argc, char * argv[]) {
    @autoreleasepool {
        return UIApplicationMain(argc, argv, nil, NSStringFromClass([AppDelegate class]));
    }
}
```

UIApplicationMain 的调用被 @autoreleasepool{} 整个包裹起来，表示 UIApplicationMain 函数执行之前，创建了一个 AutoRelease Pool，在函数返回之后，释放了之前创建的 AutoRelease Pool。在此期间，如果有对象要加入 AutoRelease Pool，就是加入的这个创建的 AutoRelease Pool。

上边提到，在大多数情况下，开发人员不需要自己创建和销毁自动释放池，现在谈一下少数情况。开发人员需要自己使用 AutoRelease Pool 的情形，通常是如下情况：

```
for (int i = 0; i < 1000000; i++) {
    @autoreleasepool {
        NSString *str = [NSString stringWithFormat:@"hi + %d", i];
    }
}
```

如果不加上 @autoreleasepool{} 代码块，循环里的临时变量 str 会被加入到当前的 AutoRelease Pool，而这个 AutoRelease Pool 的释放时机，如上所说，是需要等到当前 Runloop 一个循环后才会释放，而这个时机我们并不能控制。这样，在 Runloop 一个循环结束前，就会出现很多临时变量 str 不用了，但是占用内存的情况。所以这里手动加上 @autoreleasepool{} 代码块，每次循环都创建一个新的 AutoRelease Pool， str 会被加入到这个新的 AutoRelease Pool，在每次 for 循环结束时，AutoRelease Pool 被释放，从而 str 也被及时释放，内存能够得到及时的清理。

## 3. Autorelease Pool的实现原理

我们从系统使用 @autoreleasepool{} 的代码入手，将 main.m 代码编译成 main.cpp 代码进行进一步分析，在 main.m 文件目录执行下面的编译命令：

```bash
clang -x objective-c -rewrite-objc -isysroot /Applications/Xcode.app/Contents/Developer/Platforms/iPhoneSimulator.platform/Developer/SDKs/iPhoneSimulator.sdk main.m
```

执行完后会生成文件 main.cpp，在文件最后会看到如下代码：

```cpp
int main(int argc, char * argv[]) {
    /* @autoreleasepool */ { __AtAutoreleasePool __autoreleasepool;
        return UIApplicationMain(argc, argv, __null, NSStringFromClass(((Class (*)(id, SEL))(void *)objc_msgSend)((id)objc_getClass("AppDelegate"), sel_registerName("class"))));
    }
}
```

可以看到，UIApplicationMain 执行前，增加了一行代码 __AtAutoreleasePool __autoreleasepool，这里声明了一个类型为 __AtAutoreleasePool 的对象。在文件里搜索  __AtAutoreleasePool，发现如下代码：

```cpp
struct __AtAutoreleasePool {
  __AtAutoreleasePool() {atautoreleasepoolobj = objc_autoreleasePoolPush();}
  ~__AtAutoreleasePool() {objc_autoreleasePoolPop(atautoreleasepoolobj);}
  void * atautoreleasepoolobj;
};
```

__AtAutoreleasePool 是一个结构体，在构造函数和析构函数里，分别调用了 objc_autoreleasePoolPush() 和 objc_autoreleasePoolPop(atautoreleasepoolobj) 方法。也就是说，在 UIApplicationMain 执行前，首先先执行了 objc_autoreleasePoolPush 方法，然后执行了 objc_autoreleasePoolPop 方法，objc_autoreleasePoolPush 是在创建 Autorelease Pool，objc_autoreleasePoolPop 是在销毁 Autorelease Pool。接下来我们通过源码分析创建和销毁 Autorelease Pool 都做了什么。

这两个方法的代码在`NSObject.mm`里，代码是开源的，可以到 [https://opensource.apple.com/release/macos-10141.html](https://opensource.apple.com/release/macos-10141.html) 下载，笔者查看的是最新的 objc4-750.1 版本。所有的历史版本可以在这里浏览 [https://opensource.apple.com/source/objc4/](https://opensource.apple.com/source/objc4/) 。

### 3.1 创建Autorelease Pool

首先看 objc_autoreleasePoolPush 的实现：

```cpp
void *
objc_autoreleasePoolPush(void)
{
    return AutoreleasePoolPage::push();
}
```

objc_autoreleasePoolPush 的实现很简单，直接调用了AutoreleasePoolPage::push() 。先来看下 AutoreleasePoolPage 是什么：

```cpp
class AutoreleasePoolPage 
{
    magic_t const magic;
    id *next;
    pthread_t const thread;
    AutoreleasePoolPage * const parent;
    AutoreleasePoolPage *child;
    uint32_t const depth;
    uint32_t hiwat;
}
```

省去其他的宏定义、常量定义和方法，AutoreleasePoolPage 有如上属性，parent 和 child 同样指向AutoreleasePoolPage， 很容易猜测 AutoreleasePoolPage 是双向链表中的一个节点，后续的代码会印证这个猜测。next 是一个指针，是一个比较重要的属性，先留意一下，后边会讲。其余的属性对理解 Autorelease Pool 原理不是特别重要，暂时先都忽略。

AutoreleasePoolPage 对象分配内存方法如下：

```cpp
static void * operator new(size_t size) {
    return malloc_zone_memalign(malloc_default_zone(), SIZE, SIZE);
}
```

SIZE 被定义为 PAGE_MAX_SIZE，PAGE_MAX_SIZE 是虚拟内存一页的大小，网上查资料说是0x1000字节。

```cpp
    static size_t const SIZE = 
#if PROTECT_AUTORELEASEPOOL
        PAGE_MAX_SIZE;  // must be multiple of vm page size
#else
        PAGE_MAX_SIZE;  // size and alignment, power of 2
#endif
```

所以，一个 AutoreleasePoolPage 对象所占用的内存大小是 PAGE_MAX_SIZE。

看到这里我们已经清楚 AutoreleasePoolPage 的内部结构，用一张图来表示：

![](https://raw.githubusercontent.com/joshuaton/img/master/20190602152848.png)

除了存储 AutoreleasePoolPage 的成员变量外，其余空间会用来存储加入到 Autorelease Pool 的对象指针。

继续看 AutoreleasePoolPage::push 方法的实现：

```cpp
static inline void *push() 
{
    id *dest;
    if (DebugPoolAllocation) {
        // Each autorelease pool starts on a new pool page.
        dest = autoreleaseNewPage(POOL_BOUNDARY);
    } else {
        dest = autoreleaseFast(POOL_BOUNDARY);
    }
    assert(dest == EMPTY_POOL_PLACEHOLDER || *dest == POOL_BOUNDARY);
    return dest;
}
```

会调用 autoreleaseFast 方法，方法的参数是 POOL_BOUNDARY ，关于 POOL_BOUNDARY 是什么，这个之后再说：

```cpp
static inline id *autoreleaseFast(id obj)
{
    AutoreleasePoolPage *page = hotPage();
    if (page && !page->full()) {
        return page->add(obj);
    } else if (page) {
        return autoreleaseFullPage(obj, page);
    } else {
        return autoreleaseNoPage(obj);
    }
}
```

首先拿到当前的 hotPage，hotPage 可以理解为正在使用的 AutoreleasePoolPage，也就是双向链表末端的 AutoreleasePoolPage。然后分为三种情况：

1. 如果有 hotPage，并且 hotPage 没有满的时候，调用 page->add(obj)

2. 如果有 hotPage，但是 hotPage 已经满的时候，调用 autoreleaseFullPage(obj, page)

3. 如果没有 hotPage，调用 autoreleaseNoPage(obj)

以下对 3 种情况分别进行说明：

第 1 种情况，查看 AutoreleasePoolPage 的 add 方法：

```
id *add(id obj)
{
    assert(!full());
    unprotect();
    id *ret = next;  // faster than `return next-1` because of aliasing
    *next++ = obj;
    protect();
    return ret;
}
```

将 next 指针指向 obj， 然后next++，返回obj。所以，这里我们可以知道，AutoreleasePoolPage 的 next 指针是指向下一个空位置，当有对象要被加入到 AutoreleasePoolPage 的时候，会加入到这个位置。

第 2 种情况，查看 autoreleaseFullPage 的实现：

```cpp
id *autoreleaseFullPage(id obj, AutoreleasePoolPage *page)
{
    // The hot page is full. 
    // Step to the next non-full page, adding a new page if necessary.
    // Then add the object to that page.
    assert(page == hotPage());
    assert(page->full()  ||  DebugPoolAllocation);

    do {
        if (page->child) page = page->child;
        else page = new AutoreleasePoolPage(page);
    } while (page->full());

    setHotPage(page);
    return page->add(obj);
}
```

新建一个 page，将新建的 page 设置为 hotPage，并且将 obj 加入到此 page 中，通过进一步查看 AutoreleasePoolPage 的构造函数会发现，新 page 的 parent 指针会设置成这个函数传入的老 page，新老 page 就形成了双向链表的结构。

第 3 种情况，查看 autoreleaseNoPage 的实现：

```cpp
id *autoreleaseNoPage(id obj)
{
    bool pushExtraBoundary = false;

    // Install the first page.
    AutoreleasePoolPage *page = new AutoreleasePoolPage(nil);
    setHotPage(page);
    
    // Push a boundary on behalf of the previously-placeholder'd pool.
    if (pushExtraBoundary) {
        page->add(POOL_BOUNDARY);
    }
    
    // Push the requested object or pool.
    return page->add(obj);
}
```

新建一个 AutoreleasePoolPage ，然后再加入 obj ，创建 Autorelease Pool 的时候，obj 的值是 POOL_BOUNDARY。

我们用一张图来表示 Autorelease Pool 创建时候的情况：

![](https://raw.githubusercontent.com/joshuaton/img/master/20190602153813.png)

在这里我们来说一下 POOL_BOUNDARY 是什么。我们可以发现其定义是为 nil

```objectivec
#define POOL_BOUNDARY nil
```

从字面意义上来讲，这是一个边界标记，当每次创建一个新的 Autorelease Pool 时，我们都会首先加入一个 POOL_BOUNDARY 标记在内存中，这样我们就知道了不同 Autorelease Pool 的分割位置在哪里。当我们需要最后创建的 Autorelease Pool 中的所有对象时，我们就只要释放这个 POOL_BOUNDARY 位置之后的对象。

### 3.2 将对象加入Autorelease Pool

创建 Autorelease Pool 的代码到此就基本看完了，我们马上再来看下将一个对象加入 Autorelease Pool 会干些什么。将对象加入 Autorelease Pool 会调用 NSObject 的 autorelease 方法，实现如下：

```cpp
static inline id autorelease(id obj)
{
    assert(obj);
    assert(!obj->isTaggedPointer());
    id *dest __unused = autoreleaseFast(obj);
    assert(!dest  ||  dest == EMPTY_POOL_PLACEHOLDER  ||  *dest == obj);
    return obj;
}
```

实际上是在调用 autoreleaseFast 方法。原来，创建一个 Autorelease Pool 和将一个 obj 加入 Autorelease Pool 其实代码流程是一样的，不同的是创建时候添加的是 POOL_BOUNDARY，添加时候添加的是 obj。

通过以上代码，我们知道往 Autorelease Pool 里添加多个对象后是什么情况了，用一张图来表示：

![](https://raw.githubusercontent.com/joshuaton/img/master/20190602154506.png)

假设我们有 obj0 到 obj4 一共 5 个对象需要添加进 Autorelease Pool。第一个 AutorelasePoolPage 没有用满时，直接往里边加，满了之后，新建一个 AutorelasePoolPage，在往里边继续加。所以，obj0、obj1、obj2、obj3 被添加到了第 1 个 AutorelasePoolPage 中，obj4 被添加到了第 2 个 AutorelasePoolPage 中。真实情况下，AutorelasePoolPage 当然不只存储 4 个对象，这里只是方便举例说明。

如果在 Autorelase Pool 没有销毁的时候，再新建一个 Autorelase Pool，则往 AutorelasePoolPage 的 next 位置加入 POOL_BOUNDARY。如果又有对象要添加进新的 Autorelase Pool，则往 AutorelasePoolPage 继续添加 obj5 和 obj6，如下图：

![](https://raw.githubusercontent.com/joshuaton/img/master/20190602154735.png)

可以看到，POOL_BOUNDARY 是边界对象，标识了多个 Autorelease Pool 的分割边界。

### 3.3 销毁Autorelease Pool

前边提到，销毁 Autorelease Pool 会调用 objc_autoreleasePoolPop 方法：

```objectivec
void
objc_autoreleasePoolPop(void *ctxt)
{
    AutoreleasePoolPage::pop(ctxt);
}
```

直接查看 AutoreleasePoolPage::pop 代码：

```objectivec
static inline void pop(void *token) 
{
    AutoreleasePoolPage *page;
    id *stop;

    page = pageForPointer(token);
    stop = (id *)token;

    if (PrintPoolHiwat) printHiwat();

    page->releaseUntil(stop);

    // memory: delete empty children
    if (DebugPoolAllocation  &&  page->empty()) {
        // special case: delete everything during page-per-pool debugging
        AutoreleasePoolPage *parent = page->parent;
        page->kill();
        setHotPage(parent);
    } else if (DebugMissingPools  &&  page->empty()  &&  !page->parent) {
        // special case: delete everything for pop(top) 
        // when debugging missing autorelease pools
        page->kill();
        setHotPage(nil);
    } 
    else if (page->child) {
        // hysteresis: keep one empty child if page is more than half full
        if (page->lessThanHalfFull()) {
            page->child->kill();
        }
        else if (page->child->child) {
            page->child->child->kill();
        }
    }
}
```

回顾下之前的代码，token 为创建 Autorelease Pool 时返回的 POOL_BOUNDARY，这个会作为 pageForPointer 的输入参数。 pageForPointer 函数的实现如下：

```objectivec
static AutoreleasePoolPage *pageForPointer(const void *p) 
{
    return pageForPointer((uintptr_t)p);
}

static AutoreleasePoolPage *pageForPointer(uintptr_t p) 
{
    AutoreleasePoolPage *result;
    uintptr_t offset = p % SIZE;

    assert(offset >= sizeof(AutoreleasePoolPage));

    result = (AutoreleasePoolPage *)(p - offset);
    result->fastcheck();

    return result;
}
```

通过 POOL_BOUNDARY 的内存地址和 AutoreleasePoolPage 的内存占用 SIZE，可以算出 POOL_BOUNDARY 相对于 AutoreleasePoolPage 起始地址的偏移量，从而计算出创建 Autorelease Pool 时候的那个 AutoreleasePoolPage 的内存起始地址。所以，pageForPointer 函数返回当前 Autorelease Pool 创建时候的 AutoreleasePoolPage。

接下来看 page->releaseUntil(stop) 的实现：

```objectivec
void releaseUntil(id *stop) 
{
    // Not recursive: we don't want to blow out the stack 
    // if a thread accumulates a stupendous amount of garbage
    
    while (this->next != stop) {
        // Restart from hotPage() every time, in case -release 
        // autoreleased more objects
        AutoreleasePoolPage *page = hotPage();

        // fixme I think this `while` can be `if`, but I can't prove it
        while (page->empty()) {
            page = page->parent;
            setHotPage(page);
        }

        page->unprotect();
        id obj = *--page->next;
        memset((void*)page->next, SCRIBBLE, sizeof(*page->next));
        page->protect();

        if (obj != POOL_BOUNDARY) {
            objc_release(obj);
        }
    }

    setHotPage(this);
}
```

从当前的 hotPage 开始，依次对 AutoreleasePoolPage 里的对象执行 objc_release 操作，直到遇到 POOL_BOUNDARY 对象。这就是对当前 Autorelease Pool 里的所有对象进行释放操作。用一张图来表示这个过程会更加直观：

![](https://raw.githubusercontent.com/joshuaton/img/master/Canvas%205.jpg)

我们可以思考一下为什么要这么设计 Autorelease Pool。由于要加入 Autorelease Pool 的对象个数是不固定的，所以系统只能一次分配固定大小的内存，也就是一个 AutoreleasePoolPage的大小。当加满了之后，再在双向链表的最后加上一个 AutoreleasePoolPage。这里其实跟操作系统给应用程序分配内存空间是一样的，也是按页分配。而如何区分多个 Autorelease Pool，就是用了 POOL_BOUNDARY 来做边界标记。

## 4. 总结

到此位置，我们已经分析完了创建 Autorelease Pool，往 Autorelease Pool 里添加对象，释放 Autorelease Pool 的主要代码。其中还有一些分支代码和异常情况的处理被省略，感兴趣的同学可以自行查看其余源码。

最后我们总结一下 Autorelease Pool 的实现原理：

1. Autorelease Pool 是由多个 AutoreleasePoolPage 对象以双向链表的方式组织起来的数据结构。

2. 每个 AutoreleasePoolPage 只能存储有限个对象指针。当新的对象加入 Autorelease Pool 的时候，如果当前的 AutoreleasePoolPage 存储空间不够，会新初始化一个 AutoreleasePoolPage，加入到链表末端。

3. Autorelease Pool 可以被嵌套创建。创建一个新的 Autorelease Pool 的时候，会在当前 AutoreleasePoolPage 中插入边界对象 POOL_BOUNDARY，以和上一个 Autorelease Pool 以区分。

4. 当 Autorelease Pool 销毁的时候，对 AutoreleasePoolPage 里存储的所有对象依次从后往前调用 release，直到遇到对象 POOL_BOUNDARY，表明当前 Autorelease Pool 中的对象已经被全部释放。

## 5. 参考资料

[https://juejin.im/post/5a66e28c6fb9a01cbf387da1](https://juejin.im/post/5a66e28c6fb9a01cbf387da1)


