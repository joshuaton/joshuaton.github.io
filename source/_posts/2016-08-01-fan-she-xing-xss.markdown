---
layout: post
title: "反射型XSS"
date: 2016-08-01 20:18:08 +0800
comments: true
categories: security
---
早时候出现的一个反射型XSS漏洞

漏洞URL
```
http://www.icbc.com.cn/news/hotspot.jsp?column=%3Cfont%20size=10%3E%BD%F7%B4%CB%D4%AA%B5%A9%BC%D1%BD%DA%D6%AE%BC%CA%A3%AC%B9%A4%C9%CC%D2%F8%D0%D0%B8%F8%C3%BF%B8%F6%B4%A2%BB%A7%C3%E2%B7%D1%B7%A2%B7%C51000%CD%F2%C3%C0%BD%F0%3C/font%3E
```

URL decode之后
```
http://www.icbc.com.cn/news/hotspot.jsp?column=<font size=10>谨此元旦佳节之际，工商银行给每个储户免费发放1000万美金</font>
```

效果
![](http://jason5.cn/images/o_ICBC1.gif)