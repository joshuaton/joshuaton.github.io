---
layout: post
title: "Markdown学习"
date: 2015-07-21 22:14:36 +0800
comments: true
categories: other
---
#这是H1
```
#这是H1

```
##这是H2
```
##这是H2
```
###这是H3
```
###这是H3
```
<!--more-->
>这是一段引用
>>这是引用内的引用哈哈

```
>这是一段引用
>>这是引用内的引用哈哈
```

###无序列表

*   Red
*   Green
*   Blue

```
*   Red
*   Green
*   Blue
```

###有序列表
1.  Bird
2.  McHale
3.  Parish

```
1.  Bird
2.  McHale
3.  Parish
```

###代码
```java
System.out.println("hello");

for(int i=0; i<a.size(); i++){
	System.out.println("test");
}
```

```
~~~java
System.out.println("hello");

for(int i=0; i<a.size(); i++){
	System.out.println("test");
}
~~~
PS: ~~~替换为```
```
	
###代码嵌入文字
Use the `printf()` function.

```
Use the `printf()` function.
```
	
###分割线
***

```
***
```

###文字链接
[点我](http://www.baidu.com "baidu title")

```
[点我](http://www.baidu.com "baidu title")
```

[foo](http://example.com/  "Optional Title Here")

```
[foo](http://example.com/  "Optional Title Here")
```

###隐式链接标记
I get 10 times more traffic from [Google][] than from
[Yahoo][] or [MSN][].

  [google]: http://google.com/        "Google"
  [yahoo]:  http://search.yahoo.com/  "Yahoo Search"
  [msn]:    http://search.msn.com/    "MSN Search"
  
```
I get 10 times more traffic from [Google][] than from
[Yahoo][] or [MSN][].

  [google]: http://google.com/        "Google"
  [yahoo]:  http://search.yahoo.com/  "Yahoo Search"
  [msn]:    http://search.msn.com/    "MSN Search"
```

###参考式链接
I get 10 times more traffic from [Google] [1] than from
[Yahoo] [2] or [MSN] [3].

  [1]: http://google.com/        "Google"
  [2]: http://search.yahoo.com/  "Yahoo Search"
  [3]: http://search.msn.com/    "MSN Search"
  
```
I get 10 times more traffic from [Google] [1] than from
[Yahoo] [2] or [MSN] [3].

  [1]: http://google.com/        "Google"
  [2]: http://search.yahoo.com/  "Yahoo Search"
  [3]: http://search.msn.com/    "MSN Search"
```

###强调  
**single asterisks**

```
**single asterisks**
```

un*frigging*believable

```
un*frigging*believable
```

###图片
![](http://www.baidu.com/img/bdlogo.png)

```
![](http://www.baidu.com/img/bdlogo.png)
```

###图片链接
[![](http://www.baidu.com/img/bdlogo.png)](http://www.baidu.com)

```
[![](http://www.baidu.com/img/bdlogo.png)](http://www.baidu.com)
```



