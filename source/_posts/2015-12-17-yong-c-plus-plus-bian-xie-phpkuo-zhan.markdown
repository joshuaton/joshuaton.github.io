---
layout: post
title: "用c++编写PHP扩展"
date: 2015-12-17 19:56:01 +0800
comments: true
categories: 
---
#背景
有时候老业务用的是C++接口，但是为了快速开发，Web类业务已经转移到PHP上，所以如何用PHP封装老业务的C++接口进行调用，就是这篇文章所要解决的问题。
<!--more-->

#编写扩展的步骤
1.创建扩展目录，目录名叫store_pic


	cd /usr/local/php-5.3.6/ext
	./ext_skel --extname=store_pic

2.修改扩展目录里的config.m4

去掉下列语句前的`dnl`注释

	PHP_ARG_ENABLE(store_pic, whether to enable store_pic support,	Make sure that the comment is aligned:	[  --enable-store_pic           Enable store_pic support])

在倒数第二行增加`PHP_REQUIRE_CXX()`,将.c后缀改为.cpp，代码如下

```java
PHP_REQUIRE_CXX()PHP_NEW_EXTENSION(store_pic, store_pic.cpp, $ext_shared)
```

3.修改模块头文件php_store_pic.h  
增加extern标签

```java
extern "C"
{#ifdef ZTS#include "TSRM.h"#endif}
```

增加方法upload_tfs_pic

```java
PHP_FUNCTION(confirm_store_pic_compiled);       /* For testing, remove later. */PHP_FUNCTION(upload_tfs_pic);
```

4.修改store_pic.c文件后缀为store_pic.cpp，并修改文件内容

增加extern标签，注意不要包含php_store_pic.h，引入扩展需要的.h文件

```java
extern "C"{#ifdef HAVE_CONFIG_H#include "config.h"#endif#include "php.h"#include "php_ini.h"#include "ext/standard/info.h"}#include "php_store_pic.h"
//引入扩展需要的.h文件
#include "store_define.h"#include "store_pic.h"#include "secret_url.h"#include "store_decrypt_url.h"#include "oi_tea.h"
```

增加方法申明

```java
const zend_function_entry store_pic_functions[] = {        PHP_FE(confirm_store_pic_compiled,      NULL)           /* For testing, remove later. */        PHP_FE(upload_tfs_pic,  NULL)  /*增加方法申明*/        {NULL, NULL, NULL}      /* Must be the last line in store_pic_functions[] */};
```

增加BEGIN_EXTERN_C()和END_EXTERN_C()

```java
#ifdef COMPILE_DL_STORE_PICBEGIN_EXTERN_C()ZEND_GET_MODULE(store_pic)END_EXTERN_C()#endif
```

在文件最后增加方法的实现

```javaPHP_FUNCTION(upload_tfs_pic)
{
	/*省略若干逻辑代码*/
}
```

5.执行预处理，生成configure文件

```java
/usr/local/bin/phpize
```

6.执行编译配置

```java
 ./configure --enable-store_pic --with-apxs=/usr/local/apache2/bin/apxs
```

7.将第4部中的.h文件拷贝到本目录

8.修改Makefile，引入需要的静态库

```java
LDFLAGS = -lstdc++ /data/lib/store_sdk/libopenapi.a /data/lib/store_sdk/libprotobuf.a /data/lib/tfs_encode/oi_tea.o /data/lib/tfs_encode/secret_url.o
```

9.编译安装

```java
make 
make test
make install
```

注意make test的时候有没有报错，passed就ok了，否则需要注意，根据出错信息进行修改。

10.将生成的store_pic.so拷贝到相关目录，修改php.ini文件引入这个so文件，重启apache即可。

#可能遇到的问题
1.不同so库的libprotobuf.a版本不一致，会导致apache不能拉起

```java
[libprotobuf FATAL google/protobuf/stubs/common.cc:72] This program was compiled against version 2.4.1 of the Protocol Buffer runtime library, which is not compatible with the installed version (2.5.0).  Contact the program author for an update.  If you compiled the program yourself, make sure that your headers are from the same version of Protocol Buffers as your link-time library.  (Version verification failed in "/data/zorroyliu/tinybusiness/sdk/sdk_newest/protocol/store_cloud_all.pb.cc".)
```

解决办法是去掉冲突的so，或者两个so用同一版本的动态库进行编译。

2.make test的时候报错

```java
undefined symbol: _ZN6google8protobuf11MessageLite15ParseFromStringERKSs in Unknow on Line 0
```

这个是Makefile中的LDFLAGS参数指定的相关动态库没有加载进来，重新指定路径到正确的.a或.o文件即可










