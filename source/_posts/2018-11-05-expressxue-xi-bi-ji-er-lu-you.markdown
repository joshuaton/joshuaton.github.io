---
layout: post
title: "Express学习笔记(二)-路由"
date: 2018-11-05 16:08:45 +0800
comments: true
categories: web
---
这一篇主要介绍路由的用法

###基本用法
```javascript
var express = require('express');
var app = express();

// GET method route
app.get('/', function (req, res) {
  res.send('GET request to the homepage');
});

// POST method route
app.post('/', function (req, res) {
  res.send('POST request to the homepage');
});

app.all('/secret', function (req, res, next) {
  console.log('Accessing the secret section ...');
  next(); // pass control to the next handler
});
```
all表示接受所有的http方法

###路由路径
支持正则表达式

```javascript
app.get('/ab*cd', function(req, res) {
  res.send('ab*cd');
});
```
此路由路径将匹配 abcd、abxcd、abRABDOMcd、ab123cd 等。


###路由处理顺序
```javascript
var cb0 = function (req, res, next) {
  console.log('CB0');
  next();
}

var cb1 = function (req, res, next) {
  console.log('CB1');
  next();
}

app.get('/example/d', [cb0, cb1], function (req, res, next) {
  console.log('the response will be sent by the next function ...');
  next();
}, function (req, res) {
  res.send('Hello from D!');
});
```
可以写数组，也可以写多个回调函数参数，挨着执行，前一个函数必须执行`next()`，否则http请求会挂起

###模块封装
可以封装一个bird.js模块

```javascript
var express = require('express');
var router = express.Router();

// middleware that is specific to this router
router.use(function timeLog(req, res, next) {
  console.log('Time: ', Date.now());
  next();
});
// define the home page route
router.get('/', function(req, res) {
  res.send('Birds home page');
});
// define the about route
router.get('/about', function(req, res) {
  res.send('About birds');
});

module.exports = router;
```

使用模块后，url从外部使用模块开始，接着模块定义的路径，就可以访问

```
var birds = require('./birds');
...
app.use('/birds', birds);
```

此时，可以相应`/birds`和`/birds/about`






