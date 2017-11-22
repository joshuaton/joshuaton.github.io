---
layout: post
title: "升级到macOS High Sierra pod不能执行的解决办法"
date: 2017-11-22 10:42:20 +0800
comments: true
categories: other
---
macOS 升级到 High Sierra 后，执行pod命令出错，出错信息如下：
```
JUNSHAO-MC0:~ junshao$ pod
-bash: /usr/local/bin/pod: /System/Library/Frameworks/Ruby.framework/Versions/2.0/usr/bin/ruby: bad interpreter: No such file or directory
```
解决办法是重装cocoapods
```
sudo gem uninstall cocoapods
sudo gem install -n /usr/local/bin cocoapods
```