---
layout: post
title: "记一次数据库被攻击的经历"
date: 2019-05-14 15:05:33 +0800
comments: true
categories: security
---

### 1. 问题出现

之前为了练手做了一个基于nodejs的后台系统。有一天突然发现http api访问没有数据了，赶快打开浏览器看了下报错信息，发现数据库的表找不到了，于是觉得问题可能有点大，马上登录服务器查看。

登录服务器后一看，发现数据库里的表被删掉了，新建了一个库叫PLEASE_READ_ME_XMG，里边有个表叫WARNING，WARNING表的结构如下图

![](https://raw.githubusercontent.com/joshuaton/img/master/20190514154008.png)

猜测warning是警告信息，Bitcoin_Address是比特币钱包地址，Email是黑客联系邮箱。不过发现这个表里并没有数据，真不知道黑客是什么目的，还是只是恶作剧。上网搜索关键字“PLEASE_READ_ME_XMG”也印证了跟比特币勒索相关。

### 2. 查找问题

然后开始查找问题，先去mysql数据文件夹查看表文件`WARNING.frm`和`WARNING.ibd`的生成时间，确定了攻击发生的时间点。

用命令查看linux机器登录记录

```bash
who /var/log/wtmp
```

发现在攻击发生的时间点，没有登录日志。那就怀疑用户是远程登录mysql直接操作数据库。于是登录数据库查看授权情况

```bash
show grants;
```

显示

```bash
GRANT ALL PRIVILEGES ON *.* TO 'root'@'%'
```

%代表可以从任意机器登录。

### 3. 解决问题

#### 3.1 修改mysql权限

首先把数据库授权改为localhost。

```bash
grant all privileges on *.* to 'root'@'localhost' with grant option
```

#### 3.2 增加binlog

其次，由于mysql没有开启binlog，所以无法查看数据库的改动记录，也就无法看到删除数据库表的那条命令信息，经过这次教训，将binlog打开。

找到mysql配置文件`my.cnf`

```
[mysqld]
log-bin=/***/***/logs
```

在mysqld项下增加log-bin的配置，指定binlog的输出路径，注意这里要修改下logs文件夹的权限，保证可以正常写入。

然后重新启动mysql

```bash
systemctl restart mysqld
```

再尝试进行mysql update操作，通过命令查看binlog，发现日志正常写入。

```bash
mysqlbinlog -d xxx logs.000001
```

其中xxx是数据库名。

#### 3.3 定时备份数据

除此之外，为了更加保险，写了备份脚本，通过mysqldump进行备份

```bash
#! /bin/sh

time=$(date "+%Y%m%d-%H%M")
mysqldump -uxxx -pxxx --databases dbname > /path/to/bak/$time.sql
```

 将脚本保存，修改执行权限，加入crontab中定时每个小时执行备份

```bash
0 * * * * /path/to/back/bak_db.sh > /dev/null 2>&1 &
```

### 4. 总结

通过这次的攻击，数据全部丢失，虽然通过其他途径找回了信息，但是mysql数据本身并没有得到恢复。得到的教训如下：

1. mysql操作权限要严格限制，通过白名单的方式放开，其他全部拒绝

2. mysql要打开binlog功能，可以用来恢复数据和查看操作记录

3. 数据要定时备份，以防丢失后能够恢复
