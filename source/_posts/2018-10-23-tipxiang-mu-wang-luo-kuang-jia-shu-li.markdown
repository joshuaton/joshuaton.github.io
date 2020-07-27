---
layout: post
title: "TIP项目网络框架梳理"
date: 2018-10-23 16:07:12 +0800
comments: true
categories: iOS
---
前段时间小伙伴重构了项目的网络层代码，将之前的过程式的代码，面向对象化了，职责分离，更易维护。不过也增加了理解成本，这里记录一下。
<!--more-->
整个网络模块的类图如下。PMD开头的类下沉到了基础库，IG开头的类仍然在项目中。

[![](https://jason5.cn/images/tip-network-uml.jpg)](https://jason5.cn/images/tip-network-uml.jpg)

* IGNetworkManager作为项目中使用网络层的入口类，不多做介绍。
* PMDNetworking是发起网络请求的类，这里首先要用PMDCallFactory工厂类，生成一个实现PMDCall协议的对象，然后调用makeCallWithRequest方法进行网络请求。
* PMDCallFactory用于生成PMDCall协议对象
* PMDCall协议对象为了避免被回收，放到了PMDCallPool里进行管理
* PMDBaseCall实现了PMDCall协议，完成了主要的网络请求逻辑。分为以下几个步骤

1.callWithRequest准备发起网络请求

2.dealWithInterceptResult遍历所有PMDIntercept，在真正发起网络请求前进行逻辑处理，处理的过程中可以中断。

3.realCallWithRequest真正发起网络请求，这里的实现交给继承类IGCall来实现，具体的实现可以是http，也可以是tcp，在TIP项目中用到了IGNetworkObject去发网路请求。

4.convertResponse将请求回来的数据，遍历PMDConverter进行处理。

* PMDInterceptor和PMDConverter协议分别是需要在网络请求发出之前和之后要处理的逻辑，只要实现此协议，加入到PMDBaseCall中就可以了。

基本的结构就是这样了，除此之外，框架还实现了取消发送，重新发送等逻辑，这里不再详细介绍。
