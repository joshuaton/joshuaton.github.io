---
layout: post
title: "用Weex实现三端开发架构和思路"
date: 2018-08-25 21:59:32 +0800
comments: true
categories: weex
---
基于目前项目的已有架构，若要实现Weex三端开发，设计的架构如下图：

[![](http://jason5.cn/images/weex-http-proxy.png)](http://jason5.cn/images/weex-http-proxy.png)
<!--more-->
1.iOS和Android客户端维持tcp+pb方式不变

2.新增http proxy模块，提供http接口给Weex h5访问

职责是将已有的tcp+pb方式访问的服务转换成http+json形式，提供给Weex h5调用。目前http proxy用java实现，与Android客户端网络层复用代码。

3.改造svr接入层，验证Weex h5登录态，进行openid转换

以微信登录为例，之前iOS和Android用的是App授权登录，由于Weex h5是微信公众号授权登录，两者appid不同，授权得到的openid和accesstoken都不一样，需要支持对公众号进行登录校验。校验完登录态后，将公众号openid转换为App openid，再与svr逻辑层进行通信。