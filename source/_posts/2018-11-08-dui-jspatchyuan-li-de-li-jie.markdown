---
layout: post
title: "对JSPatch原理的理解"
date: 2018-11-08 15:08:47 +0800
comments: true
categories: ios
---
JSPatch利用OC语言的动态特性，让OC语言根据传入的JS代码，进行动态行为修改，以达到热更新的目的。

项目中根据JSPatch的原理，自己实现了一套简单的热更新方案。以替换方法实现为例，热更新运行的步骤如下：
<!--more-->
1.补丁下发阶段

应用启动的时候，会加载下发的补丁js文件，以下这一段热更新代码会被执行。

```javascript
replaceMethod("IGTabBarController", "onNaviBarTaskBoxClick:", false, function (invocation) {
    log("origin method");
    callOriginMethod(invocation, "origin_onNaviBarTaskBoxClick:");
});
```
OC中的JSContext在初始化的时候加载过replaceMethod函数，所以会调用到OC代码

```
ocReplaceMethod:(NSString *)className selectorName:(NSString *)selectorName isClass:(BOOL)isClass func:(JSValue *)func
```

这个函数里，做了一个重要的逻辑（这里参考了JSPatch的实现方式），将IGTabBarController的实例方法onNaviBarTaskBoxClick:指向了forwardInvocation:，然后自定义实现PMDForwardInvocation替换forwardInvocation:的行为

2.用户调用阶段

用户操作点击后，IGTabBarController的onNaviBarTaskBoxClick:会被执行，从而PMDForwardInvocation被执行，根据OC的函数转发特性，PMDForwardInvocation会拿到所有的函数参数信息invocation。然后调用`jsfunc(@[invocation])`。这样就将所有原生参数通过invocation对象传回给了js代码。js代码拿到这些参数就可以去实现任何逻辑了，以达到替换原方法的目的。

总结：这里是一个很重要的技巧，如何将需要动态更新的OC方法的参数全部传给js代码，JSPatch是利用了forwardInvocation的特性。
