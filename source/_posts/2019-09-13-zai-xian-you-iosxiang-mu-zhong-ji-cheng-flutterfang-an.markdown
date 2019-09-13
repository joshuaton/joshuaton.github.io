---
layout: post
title: "在现有iOS项目中集成Flutter方案"
date: 2019-09-13 22:09:38 +0800
comments: true
categories: ios
---
在现有项目中继承Flutter，可以用module的模式，官方已经支持。详细见文章[Add Flutter to existing apps](https://github.com/flutter/flutter/wiki/Add-Flutter-to-existing-apps)

官方文章中的集成有一个问题，native工程和flutter工程耦合性很强，导致如果一个项目组负责原生开发和flutter开发是不同人的话，原生开发必须安装flutter环境，才能跑起来原生工程。所以更好的办法是将flutter工程生成的framework集成进native工程里，这样native工程可以脱离flutter环境执行。有赞用了这样的方法，详见文章[有赞 Flutter 混编方案](https://tech.youzan.com/you-zan-flutter-hun-bian-fang-an/)

看了有赞的文章，知道了这样的事情，需要将App.framework和Flutter.framework这两个文件集成到native工程里，就可以跑起来了，并且Flutter.framework有分不同的版本，jit和aot，所以要根据需要集成不同的framework。但是这篇文章还是没有将清楚，从哪里获取这两个framework，通过实践提取验证了下，所以有了这篇文章。

有赞的文章还给了一个非常重要的灵感，framework可以放入pod私有库里进行管理，这样方便切换不同的framework库。于是我们的项目有了这样的架构。

![](https://raw.githubusercontent.com/joshuaton/img/master/20190913222429.png)

Natvie分别依赖两个私有库，FlutterFrameworkDebug和FlutterFrameworkRelease，两个私有库分别包含debug和release版本的App.framework和Flutter.framework。

下边说一下这些Framework从哪里取。

* Flutter.framework

（1）Debug版本

[Flutter工程Path]/.ios/Flutter/engine/Flutter.framework

（2）Release版本

[Flutter安装版本Path]/bin/cache/artifacts/engine/ios-release/Flutter.framework

* App.framework

（1）Debug版本

[Flutter工程Path]/.ios/Flutter/App.framework

（2）Release版本(先运行flutter build ios)

[Flutter工程Path]/build/ios/iphoneos/Runner.app/Frameworks/App.framework

取到相应的framework后，就可以部署好FlutterFrameworkDebug和FlutterFrameworkRelease私有库。搞好之后在native工程里修改Podfile

```
#pod 'FlutterFrameworkDebug',  '0.1.1'    #在模拟器上和真机上都能运行
pod 'FlutterFrameworkRelease',  '0.1.1'   #只能在真机上运行
```

切换一下重新pod install就可以了。

