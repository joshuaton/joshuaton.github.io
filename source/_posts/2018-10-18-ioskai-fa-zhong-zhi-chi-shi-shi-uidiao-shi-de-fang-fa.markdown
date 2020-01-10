---
layout: post
title: "iOS开发中支持实时UI调试的方法"
date: 2018-10-18 14:02:46 +0800
comments: true
categories: iOS
---
###背景
平时在开发iOS界面的过程中，如果修改了布局代码，需要重新启动模拟器，进行效果查看。如果工程较大，启动将耗时比较长，比较浪费时间。这里介绍一个工具InjectionIII，使用后可以不重启应用，保存文件后直接查看修改效果，极大提升界面开发工作的效率。
<!--more-->
###使用方法
1.在App Store下载InjectionIII

2.在应用启动函数 加入以下代码

```objective-c
#if DEBUG
    NSBundle *bundle = [NSBundle bundleWithPath:@"/Applications/InjectionIII.app/Contents/Resources/iOSInjection10.bundle"];
    [bundle load];
#endif
```

3.启动InjectionIII，重启XCode，然后用模拟器启动应用。状态栏上有InjectionIII的小图标，确认File Watcher选项已经勾选。这时候修改文件，只要保存，在模拟器界面上会立即更新效果

PS: 这个工具的缺点是只支持模拟器，原因参见原理部分。

###原理
[Injection：iOS热重载背后的黑魔法](https://mp.weixin.qq.com/s?__biz=MjM5NTQ2NzE0NQ==&mid=2247483999&idx=1&sn=bc88d37b6f819bd6bd7d8b76e9787620&chksm=a6f958b9918ed1af9a084ce2c2732aaee715193e37fdb830dc31d8f0174c0314b22dc5c0dd1e&mpshare=1&scene=1&srcid=0612tT8PS1pePiL5EmqMr9HH#rd)
