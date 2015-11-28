---
layout: post
title: "Shadowsocks上网攻略"
date: 2015-08-02 16:27:32 +0800
comments: true
categories: other
---

由于某些原因，某博士的KX上网快要停止服务了，平时离不开Google，离不开……，为了自己能够更方便的工作和娱乐，也不再求人，于是研究了下如何自己搭建KX上网服务。下边就说一下如何利用Shadowsocks来搭建KX上网服务。
<!--more-->
##1. 为什么选择Shadowsocks
这里主要比较下Shadowsocks和VPN，Shadowsocks有以下两个好处： 

（1）精细化的配置  
VPN是全局的网络代理，如果连上VPN后，所有的网络流量都会经过VPN。Shadowsocks是通过客户端的配置文件来决定哪些域名经过代理，是针对http来区分的，可以做到精细的控制。比如，如果我用VPN访问国内的网站，例如优酷看视频，这样就不是很划算，速度可能比直接访问更慢。这种情况下就更适合用Shadowsocks，配置以下，访问Google走Shadowsocks代理，访问优酷就正常访问。但是，如果你在手机上需要用Facebook这种APP，那么全局的VPN代理更加合适。

（2）客户端支持多终端  
Shadowsocks还有一个好处是客户端是多终端的，Mac，Windows，iOS，Android都有对应的客户端软件，配置起来都比较方便。

##2. 申请VPS
我第一次用国外的VPS，选了DigitalOcean，原因是便宜，最低配置的5美元一个月，并且注册就会送10美元，相对于免费使用2个月。大家可以根据自己的需要选择合适的机器。

![](http://i3.tietuku.com/bd093cfc22b1713c.png)

可以通过这个邀请链接申请主机  
[https://www.digitalocean.com/?refcode=9e02e540da9b](https://www.digitalocean.com/?refcode=9e02e540da9b)  
如果通过这个邀请注册成功，你每花25美元，我也能得到25美元的奖励。

##3. 在服务器端使用Shadowsocks
以CentOS操作系统为例。

（1）安装Shadowsocks
```javascript
yum install python-setuptools && easy_install pip
pip install shadowsocks
```

（2）生成配置文件config.json
新建配置文件/etc/shadowsocks/config.json，内容如下
```javascript
{
    "server":"my_server_ip",
    "server_port":8080,
    "local_port":1080,
    "password":"password",
    "timeout":1000,
    "method":"bf-cfb"
}
```
各字段含义如下  
server          服务器 IP (IPv4/IPv6)，注意这也将是服务端监听的 IP 地址  
server_port     服务器端口  
local_port      本地端端口  
password        用来加密的密码  
timeout         超时时间（秒）  
method          加密方法，可选择 "bf-cfb", "aes-256-cfb", "des-cfb", "rc4", 等等。默认是一种不安全的加密，推荐用 "aes-256-cfb"。这里我用的是bf-cfb，因为aes-256-cfb默认不支持  

（3）启动Shadowsocks
```javascript
nohup ssserver -c /etc/shadowsocks/config.json &gt; log &amp;
```

##4. 在客户端使用Shadowsocks
以iMac为例，下载Shadowsocks，运行后点击小飞机的图标，然后点击“打开服务器设定”，按照服务器配置就好了
![](http://i1.tietuku.com/12f7418860175172.jpg)

到这里KX上网搭建工作完成，开始愉快的享受吧！

***
参考资料：  
[shadowsocks 服务端部署](http://hceasy.com/2013/12/shadowsocks-%E6%9C%8D%E5%8A%A1%E7%AB%AF%E9%83%A8%E7%BD%B2/)  
[Shadowsocks wiki](https://wiki.archlinux.org/index.php/Shadowsocks_(%E7%AE%80%E4%BD%93%E4%B8%AD%E6%96%87)