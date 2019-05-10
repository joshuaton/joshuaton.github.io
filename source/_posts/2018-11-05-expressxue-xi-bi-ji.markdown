---
layout: post
title: "Express学习笔记(一)-开始"
date: 2018-11-05 15:57:02 +0800
comments: true
categories: web
---
Express是一个基于nodejs的web开发框架。

###Hello World
0.安装nodejs环境

以腾讯云的centOS云服务器为例，先运行

```
yum install nodejs
```

1.安装

```
$ mkdir myapp
$ cd myapp
$ npm init
$ npm install express --save
```
2.修改app.js，然后运行

```
$ node app.js
```
打开浏览器，访问http://localhost:3000，可以看到运行结果了

###脚手架工程

1.安装express-generator

```
$ npm install express-generator -g
```

2.生成脚手架工程

```
$ express --view=pug myapp
```
pug是使用的模板引擎

```
$ cd myapp
$ npm install
```

3.运行

```
$ npm start
```

在浏览器里访问。

###路由

```javascript
app.get('/', function (req, res) {
  res.send('Hello World!');
});
```
表示根目录相应http get方法

###静态文件
```javascript
app.use('/static', express.static(__dirname + '/public'));
```

public目录下的所有文件可以作为静态资源访问。

