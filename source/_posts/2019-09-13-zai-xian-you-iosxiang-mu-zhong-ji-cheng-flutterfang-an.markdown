---
layout: post
title: "在现有iOS项目中集成Flutter方案"
date: 2019-09-13 22:09:38 +0800
comments: true
categories: ios
---
## 1. 集成方式选择

官方提供了源码集成方式，详细见文章[Add Flutter to existing apps](https://github.com/flutter/flutter/wiki/Add-Flutter-to-existing-apps)。

该方式有一个问题，Native工程和Flutter工程耦合性强，Native开发人员必须安装Flutter运行环境，才能运行真个工程，CI构建机上也要求有Flutter环境。更好的办法是将Flutter工程生成的产物集成进Native工程里，这样Native工程可以脱离Flutter环境运行。
<!--more-->
经过调研和参考，我们的项目用了源码集成和私有库相结合的方式：

![](https://raw.githubusercontent.com/joshuaton/img/master/20191111161133.png)

1. Flutter开发阶段，以源码方式引入，方便代码调试和快速查看效果。

2. 编写构建脚本，脚本根据环境生成Debug和Release两个不同版本的构建产物，分别放在FlutterFrameworkDebug和FlutterFrameworkRelease私有库中。

3. Native开发人员以Pod私有库方式，将Flutter产物引入原生工程，线上版本同样以Pod私有库方式引入。

该集成方式很好的弥补了代码集成方式的缺点。Podflie的编写如下：

```python
# 是否源码集成, 0:否 / 1:是
IsFlutterSourceCode = 1

if IsFlutterSourceCode == 1
  # 需设置tip-flutter路径的环境变量
  # 如：export TIP_FLUTTER_HOME=/Users/xxx/code/tip-flutter
  flutter_application_path = ENV["TIP_FLUTTER_HOME"]
  if flutter_application_path.to_s == ''
    puts "flutter_application_path not found! Please set ENV[\"TIP_FLUTTER_HOME\"]."
    exit
  end
  load File.join(flutter_application_path, '.ios', 'Flutter', 'podhelper.rb')
end

target 'IGame' do
  if IsFlutterSourceCode == 1
    install_all_flutter_pods(flutter_application_path)
  else
    pod 'FlutterFrameworkDebug',  '0.1.68', :configurations => ['Debug']
    pod 'FlutterFrameworkRelease',  '0.1.60', :configurations => ['Release', 'Test']
  end
end
```

通过修改IsFlutterSourceCode开关配置集成方式。

接下来介绍私有库产物的收集方法。

## 2. Flutter产物收集

产物收集分为三个部分，App.framework、Flutter.framework、Plugin（包括Plugin.h和Plugin.a文件）。

framework的收集分为Debug版本和Release版本，收集Debug版本前需要以源码模式先运行工程一次，收集Release版本前需要先执行`flutter build ios`，这样对应的产物才能生成。

### 2.1 App.framework

App.framework是逻辑代码。

文件位置：

- Debug版本：$TIP_FLUTTER_HOME/.ios/Flutter/App.framework

- Release版本：$TIP_FLUTTER_HOME/build/ios/iphoneos/Runner.app/Frameworks/App.framework

### 2.2 Flutter.framework

Flutter.framework是引擎代码。

文件位置：

- Debug版本：$TIP_FLUTTER_HOME/.ios/Flutter/engine/Flutter.framework

- Release版本：$FLUTTER_HOME/bin/cache/artifacts/engine/ios-release/Flutter.framework

### 2.3 Plugin

为了减少复杂度，Plugin产物没有区分Debug版本和Release版本。

根据`.flutter-plugins`文件的内容分析用到了哪些plugin。

例如文件内容如下：

```bash
flutter_mmkv_cache=/Users/junshao/Documents/Code/Flutter/flutter/.pub-cache/hosted/pub.dartlang.org/flutter_mmkv_cache-0.0.2/
path_provider=/Users/junshao/Documents/Code/Flutter/flutter/.pub-cache/hosted/pub.dartlang.org/path_provider-1.3.0/
sqflite=/Users/junshao/Documents/Code/Flutter/flutter/.pub-cache/hosted/pub.dartlang.org/sqflite-1.1.7+1/
```

可以分析出来三个plugin，分别是`flutter_mmkv_cache`，`path_provider`，`sqflite`。

还不止这些，plugin可能会依赖native库，还需要进一步分析。

以`flutter_mmkv_cache`为例，查看podspec文件，位置在`/Users/junshao/Documents/Code/Flutter/flutter/.pub-cache/hosted/pub.dartlang.org/flutter_mmkv_cache-0.0.2/ios`，文件中有依赖的native库：

```python
  s.dependency 'Flutter'
  s.dependency 'MMKV'
```

除去`Flutter`本身，`MMKV`就是需要依赖的native库。

分析完所有plugin和依赖库后，开始构建plugin。需要针对模拟器和真机分别构建.a文件，然后将两个.a文件合并。相关命令如下：

```bash
/usr/bin/env xcrun xcodebuild build -configuration Debug ARCHS='x86_64' -target $plugin_name BUILD_DIR=$TIP_FLUTTER_HOME/build/ios -sdk iphonesimulator > /dev/null 2>&1
/usr/bin/env xcrun xcodebuild build -configuration Release ARCHS='arm64 armv7' -target $plugin_name BUILD_DIR=$TIP_FLUTTER_HOME/build/ios -sdk iphoneos > /dev/null 2>&1
lipo -create $TIP_FLUTTER_HOME/build/ios/Debug-iphonesimulator/$plugin_name/lib$plugin_name.a $TIP_FLUTTER_HOME/build/ios/Release-iphoneos/$plugin_name/lib$plugin_name.a -o $TIP_FLUTTER_HOME/build/ios/Release-iphoneos/$plugin_name/lib$plugin_name.a > /dev/null 2>&1
```

这样就得到了plugin的产物.h和.a文件。

## 3. 遗留问题

分析Plugin的依赖时，如果native工程里已有，会造成冲突。例如`flutter_mmkv_cache`需要依赖`MMKV`，但是以前的native工程也依赖了`MMKV`，会导致库重复编译不过。目前我们的做法是，去掉native工程的`MMKV`。有更好的方法欢迎指正。

## 4. 参考资料

- [https://github.com/flutter/flutter/wiki/Add-Flutter-to-existing-apps](https://github.com/flutter/flutter/wiki/Add-Flutter-to-existing-apps)

- [https://tech.youzan.com/you-zan-flutter-hun-bian-fang-an/](https://tech.youzan.com/you-zan-flutter-hun-bian-fang-an/)

