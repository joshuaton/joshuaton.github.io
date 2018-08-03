---
layout: post
title: "利用Weex DevTool调试Native应用-iOS篇"
date: 2018-04-03 17:48:04 +0800
comments: true
categories: ios
---
官方的文档里说提供了一个工具DevTool，用来调试原生应用，但是写得比较简略，一直跑不起来，经过几天折腾和查资料，基本上是可以调试了，记录一下步骤和问题。

<!--more-->

###1. 先在本机安装iOS的调试工具
工具地址是 [https://github.com/weexteam/weex-devtool-iOS](https://github.com/weexteam/weex-devtool-iOS)

```javascript
npm install -g weex-devtool
```
运行调试工具weex-devtool，启动成功后，在终端命令行里会显示如下两行信息。

[![](http://jason5.cn/images/QQ20180403-181423.png)](http://jason5.cn/images/QQ20180403-181423.png)


Websocket Address For Native是调试工具暴露的一个websocket地址，用于和Native之间的双向通信，之后Native中的代码会用到。
Debug Server是调试工具提供的一个web服务地址，开发者通过这个网页可以像调试web页面一样，来调试Native应用。


###2. 将WXDevTool集成到项目工程中
增加podfile文件，只在Debug模式下集成WXDevtool

```javascript
pod 'WXDevtool',   '0.15.3', :configurations => ['Debug']
```
这里遇到两个问题

（1）集成后编不过，根据提示发现这个库和项目里用到的FLEX有冲突，所以暂时去掉了FLEX。希望后续升级版本能解决这两个库之间的冲突。

（2）发现还编不过，是需要再集成一个依赖库SocketRocket

```
pod 'SocketRocket', '0.4.2'
```

###3. 在App启动的时候加入如下代码
```objc
[WXDevTool setDebug:YES];
[WXDevTool launchDevToolDebugWithUrl:@"ws://10.32.194.33:8088/debugProxy/native"];
``` 
这里launchDevToolDebugWithUrl函数需要传一个参数，就是在第一步中的websocket地址。

###4. 开始调试
如果之前一切顺利的话，现在就可以调试了。网页打开调试地址http://10.32.194.33:8088    
启动Native App，然后会看到我们的App信息出现在网页中。

[![](http://jason5.cn/images/QQ20180403-182830.png)](http://jason5.cn/images/QQ20180403-182830.png)


可以看到下边两个黑色的大按钮，是调试工具提供的两个功能，Debugger和Inspector。
Debugger用来调试js代码。在里边可以给代码设置断点，观察变量值，查看console.log的输出，跟Web开发一样的体验。

[![](http://jason5.cn/images/QQ20180403-193719.png)](http://jason5.cn/images/QQ20180403-193719.png)


Inspector用来调试UI界面。这里可以看到界面的树形结构，可以直接修改位置等属性，可以实时看到效果。这里如果在终端操作页面，浏览器里的页面也是会实时刷新的。

[![](http://jason5.cn/images/QQ20180403-193917.png)](http://jason5.cn/images/QQ20180403-193917.png)

这里遇到了一个问题，启动App后，必须首先进入Debugger页面打开调试工具，Native里的weex页面才能被正常渲染和执行，暂时还没搞清楚原因。

### 5.总结
调试工具可以让开发Native的时候，像Web开发一样去调试，一定程度上提高了效率，对熟悉Web开发的同学来讲，应该能很快上手。


PS：下一篇文章会讲一下原生页面和weex页面之间的跳转以及在weex中如何自定义导航栏。
