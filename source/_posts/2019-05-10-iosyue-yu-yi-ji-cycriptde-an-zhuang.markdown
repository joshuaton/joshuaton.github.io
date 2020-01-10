---
layout: post
title: "iOS越狱以及cycript的安装"
date: 2019-05-10 18:01:04 +0800
comments: true
categories: ios
---

本篇文章向iOS逆向初学者介绍如何越狱手机，并且安装cycript工具来调试第三方App。
<!--more-->
### 1. 准备

- 1台iOS越狱手机

- mac电脑

- 苹果开发者账号

### 2. 手机如何越狱

根据当前网上查到的资料，iOS 12.1.4以下（包含）是可以越狱的，如果你的手机系统版本升级到了12.1.4之上，目前来讲没有办法，可以去淘宝买一部越狱手机。

笔者的手机系统版本是iOS 11.2.5，这个版本是可以非完美越狱的。非完美越狱的意思是，手机越狱成功后，如果重启，则会恢复到未越狱状态，需要重新越狱。不过这个是可以接受的，毕竟可以用了不是。以下以iOS 11.2.5为例，介绍越狱步骤。

#### 2.1 越狱工具的选择

笔者越狱的时候尝试了两个工具Unc0ver和Electra，用了之后发现Electra好用一点，Unc0ver的问题是，越狱之后Cydia总是出现网络问题连接不上。用Electra越狱后Cydia可以使用，打开Cydia后会提示升级，正常升级后Cydia会变成Sileo，这个可以理解为新版的Cydia。

#### 2.2 如何安装Electra

首先要在mac上安装Cydia Impactor。Cydia Impactor安装后，用数据线连接手机和电脑，将Electra.ipa拖入Impactor。安装的时候会要求输入账号名和密码。这里需要输入苹果开发者账号（每年699元那个），输入的密码并不是开发者账号的密码，而是临时生成的密码，需要到[https://appleid.apple.com](https://appleid.apple.com/)生成App专用密码。这样Electra就顺利安装到手机上了。

#### 2.3 越狱

打开手机上的Electra App，点击Jailbroken按钮，注意底下会有个开关选项"Tweaks"，这里需要把选项关掉，否则后边使用cycript会出错。顺利的话，手机会黑屏，然后重新启动桌面。再打开Electra App，刚才的Jailbroken按钮变成了Already Jailbroken，说明越狱已经成功了。手机桌面应该也出现了Cydia或Sileo图标。

### 3. 如何ssh到手机

有两种方式可以使mac ssh到手机上，一种是通过同一局域网WIFI，另一种是通过手机直接连接到mac。为了获取更快的连接速度和更稳定，笔者选择了后一种方式。

首先在mac上安装usbmuxd

```bash
brew install usbmuxd
```

usbmuxd自带了一个工具iproxy，这个工具的作用是，在mac和手机之间架上一个代理，mac ssh手机本来是走22端口，需要用这个工具将其他端口，比如5678，转发到22端口上。运行命令

```bash
iproxy 5678 22
```

运行后正常情况下会显示

```bash
waiting for connection
```

这时候，在mac上打开另一个终端窗口，输入命令

```bash
ssh -p 5678 root@127.0.0.1
```

输入默认的root密码alpine，就能成功ssh到手机上了。关于如何不每次都输入密码，这里可以自行搜索下其他相关资料，这里不再累述。

除了能够使用ssh，还可以使用scp向手机传文件，这里需要注意同样需要指定端口号，例如

```bash
scp -P 5678 filexxx root@127.0.0.1:/tmp
```

ssh到手机上之后，如果我们需要其他的软件，可以使用apt-get，例如安装vim

```bash
apt-get install vim
```

### 4. cycript的安装和使用

能够以root身份ssh到手机之后，就可以做很多事情了，比如查看当前手机正在运行的App的进程号，以及各个App的沙盒路径，以及随意浏览里边的文件，很多有趣的事情可以去探索和尝试。这篇文章主要介绍cycirpt工具的使用。

#### 4.1 cycript介绍

cycript是一个脚本语言，通过编写脚本代码，可以在任意第三方App运行的时候，执行任何逻辑，通常情况下可以用来查看App的UI结构，某个页面的实现方式。其他的使用目的等待你去探索和发现。

#### 4.2 安装和使用cycript

在网上查了些资料，说cycript可以直接安装在iOS手机上的，这样只要ssh到手机上，执行cycript命令，就可以使用了。但是经过尝试，iOS 11.2.5版本不能直接安装cycript，取而代之的是一个叫bfinject的工具。bfinject工具先安装到手机上，然后启动cycript监听服务，然后mac上安装cycript客户端，通过无线的方式连接手机暴露出来的ip和端口，就可以使用cycript编写和执行脚本了。

##### 4.2.1 在手机上安装bfinject

去[https://github.com/BishopFox/bfinject](https://github.com/BishopFox/bfinject)下载bfinject，scp到手机上，然后解压，在解压目录运行`bash bfinject`启动服务。

通常情况下，会报错`Unknown jailbrek`。我们通过查看bfinject的代码，可以发现问题。

```bash
#
# Detect LiberiOS vs Electra
#
if [ -f /bootstrap/inject_criticald ]; then
    # This is Electra
    echo "[+] Electra detected."
    cp jtool.liberios /bootstrap/usr/local/bin/
    chmod +x /bootstrap/usr/local/bin/jtool.liberios
    JTOOL=/bootstrap/usr/local/bin/jtool.liberios
    cp bfinject4realz /bootstrap/usr/local/bin/
    INJECTOR=/bootstrap/usr/local/bin/bfinject4realz
elif [ -f /jb/usr/local/bin/jtool ]; then
    # This is LiberiOS
    echo "[+] Liberios detected"
    JTOOL=jtool
    INJECTOR=`pwd`/bfinject4realz
else
    echo "[!] Unknown jailbreak. Aborting."
    exit 1
fi
```

由以上代码第1个if分支可以判断，/bootstrap/inject_criticald目录不存在，这个是由于软件的目录变了，在不改变bfinject代码的前提下，需要修改一下目录，执行以下命令

```bash
ln -s /electra /bootstrap
mkdir /bootstrap/usr
mkdir /bootstrap/usr/local
mkdir /bootstrap/usr/local/bin
```

重新执行`bash bfinject`，还会出现一个错误`md5: command not found`，这个是由于需要运行md5程序，但是ios系统自带的程序叫md5sum，所以简单的办法是修改bfinject代码，将其中的md5改成md5sum就可以了。

##### 4.2.2 在手机上启动bfinject服务

要查看哪个App，需要先在手机上启动，然后找到App的进程号，例如我们要查看微信的进程号，可以用以下命令

```bash
ps aux | grep containers | grep WeChat
```

假设微信的进程号为1026，运行

```bash
bash bfinject -p 1026 -L cycript
```

成功的话，在微信App的页面内会弹出一个框，上边的意思是正在监听xx.xx.xx.xx:1337端口(xx.xx.xx.xx为手机的ip地址)，可以连接这个端口来进行cycript调试了。

##### 4.2.3 在mac上安装cycript客户端

首先去[http://www.cycript.org/](http://www.cycript.org/)下载代码，运行./cycript的时候，会报错，仔细查看是缺少ruby2.0的库，在mac上先安装ruby2.0

```bash
brew install ruby@2.0
```

安装后将`/usr/local/Cellar/ruby@2.0/2.0.0-p648_2/lib/libruby.2.0.0.dylib`拷贝到`cycript/Cycript.lib`目录下。顺利的话就可以用了。

在mac上执行

```bash
cycript -r xx.xx.xx.xx:1337
```

xx.xx.xx.xx为手机的ip，这里注意一下，因为是以无线连接的，所以手机和电脑需要在同一个局域网里，如果出现

```bash
cy#
```

代表cycript运行成功，可以通过写cycript脚本执行命令了。

### 5. 总结

本篇文章介绍了如何越狱iOS手机，以及逆向工具cycript的安装。关于cycript的详细使用，限于篇幅原因不介绍了，请自行查看相关资料。总之，这个工具非常强大，可以更深入的分析目标App的实现方式。
