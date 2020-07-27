---
layout: post
title: "在新电脑上恢复Octopress"
date: 2018-10-18 15:56:46 +0800
comments: true
categories: other
---
换了新电脑，如何在新电脑上继续使用OctoPress呢，只需要执行以下命令

1.首先将博客的源文件clone到本地的octopress文件夹内

```
$ git clone -b source https://github.com/username/username.github.io.git octopress  

```

2.将博客文件clone到octopress的_deploy文件夹内

```
$ cd octopress  
$ git clone https://github.com/username/username.github.io.git _deploy   
```

username为github用户名