---
layout: post
title: "如何查看p12文件的信息"
date: 2017-12-16 14:37:21 +0800
comments: true
categories: ios
---
```
openssl pkcs12 -in xxx.p12 -out xxx.pem -nodes
```
pem文件可以直接用文本工具打开