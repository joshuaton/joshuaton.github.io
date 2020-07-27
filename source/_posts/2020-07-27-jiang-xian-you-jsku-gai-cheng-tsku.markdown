---
layout: post
title: "将现有js库改成ts库"
date: 2020-07-27 17:48:48 +0800
comments: true
categories: 
---

### 1. 背景

现在的公共库用js来书写，有两个问题：

1. 调用方无法获得语法提示，且对调用函数的参数类型不明

2. 公共库中的一些变量类型模糊不清

以上两个问题虽然不大，但是一定程度上影响了项目的开发效率和质量。通过typescript来改写公共库，有利于解决上述两个问题。

### 2. 操作步骤

#### 2.1 命令行安装Typescript

```shell
npm install --save-dev typescript
npm install --save-dev @vue/cli-plugin-typescript
```

### 2.2 编写typescript配置

在根目录下新建文件tsconfig.json，以下边的内容为例

```javascript
{
  "compilerOptions": {
    "target": "esnext",
    "module": "esnext",
    "strict": true,
    "importHelpers": true,
    "moduleResolution": "node",
    "experimentalDecorators": true,
    "esModuleInterop": true,
    "allowSyntheticDefaultImports": true,
    "sourceMap": true,
    "baseUrl": ".",
    "allowJs": false,
    "noEmit": true,
    "types": [
      "webpack-env"
    ],
    "paths": {
      "@/*": [
        "src/*"
      ]
    },
    "lib": [
      "esnext",
      "dom",
      "dom.iterable",
      "scripthost"
    ]
  },
  "exclude": [
    "node_modules"
  ]
}
```

由于实际项目中改写的工作量比较多，暂时先将`"strict": true`改为`"strict": false`，如果设置为true的话，`npm run dev`的时候会有很多告警，会提示函数定义的时候没有指定参数类型。

#### 2.3 新增shims-vue.d.ts

根目录下新建文件`shims-vue.d.ts`，让 `ts` 识别 `*.vue` 文件

```javascript
declare module '*.vue' {
  import Vue from 'vue'
  export default Vue
}
```

如果vue文件的script段需要用ts来写，需要这一步操作。由于我这次是改写公共js库，所以暂时跟这里不相关。

#### 2.4 修改公共库文件后缀

将js文件改为ts文件后缀。这个时候会报很多错误，大部分是类型不匹配相关的，这里需要逐一解决。

### 3. 转ts代码时遇到的问题和改写方法

#### 3.1 类型“{}”上不存在属性“get”

```javascript
var Cookies = {}
Cookies.get = function(key, defaultval) {
    //省略若干代码
}
```

.get方法的定义会报错，这里是js转ts最常见的错误，解决方法是将Cookies定义为class

```typescript
class Cookies {
  static get(key: string, defaultval?: string) {
    //省略若干代码
  }
}
```

这里需要将get方法定义成static，这样才可以以`Cookies.get`的形式调用，和之前保持不变。

如果要使用的对象{}里只包含属性，不包含方法，也可以将{}定义为接口interface，例如

```typescript
interface Env{
  ua: string;
  currPage: string;
  isWeixin: boolean;
  isWorkWeixin: boolean;
  isQQ: boolean;
  isPvpApp: boolean;
  isTipApp: boolean;
  isAndroid: boolean;
  isIos: boolean;
  isMsdk: boolean;
  isSlugSdk: boolean;
  isInGame: boolean;
  isGHelper: boolean;
  isGHelper20004: boolean;
  isMiniProgram: boolean;
  isWindowsPhone: boolean;
  isSymbian: boolean;
  isPc: boolean;
  version: string;
  loginType: string;
  authQuery: Object;
  urlmodule: string;
  vconsole: any;
}
```

使用的时候如下

```typescript
var env = {} as Env;
env.ua = useragent.toLowerCase()
env.isWeixin = env.ua.indexOf('micromessenger') != -1
env.isWorkWeixin = env.ua.indexOf('micromessenger') != -1 && env.ua.indexOf('wxwork') != -1
env.isQQ = env.ua.indexOf(' qq/') != -1
env.isPvpApp = env.ua.indexOf(' igameapp/') != -1
env.isTipApp = env.ua.indexOf(' gamelife/') != -1
env.isAndroid = env.ua.indexOf('android') != -1
env.isIos = (env.ua.indexOf('iphone') != -1) || (env.ua.indexOf('ipad') != -1)
env.isMsdk = env.ua.indexOf(' msdk/') != -1 // msdk
env.isSlugSdk = env.ua.indexOf('ingame') != -1 // 微社区sdk
env.isInGame = env.isMsdk || env.isSlugSdk // 是否游戏内
env.isGHelper = env.ua.indexOf('gamehelper') != -1
env.isGHelper20004 = env.ua.indexOf('gamehelper_20004') != -1
env.isMiniProgram = env.ua.match(/miniprogram/i) != null || window.__wxjs_environment === 'miniprogram'
```

#### 3.2 typeof后边的内容未定义

```javascript
if (typeof node_cookie !== 'undefined') {
    cookie = node_cookie
  }
```

这里node_cookie是未定义的内容，可能是在其他环境里载入的变量，目前因为用不到，所以暂时是去掉了

#### 3.3 window下的属性不存在

```javascript
window.__wxjs_environment === 'miniprogram'
```

这行代码会报错，类型“Window & typeof globalThis”上不存在属性“__wxjs_environment”，解决方法是声明一个Window下的全局变量

```typescript
declare global {
  interface Window { __wxjs_environment: any; }
}
```

#### 3.4 赋值时类型不匹配

##### 3.4.1 第一个例子

```javascript
env.isMiniProgram = env.ua.match(/miniprogram/i)
```

不能将类型“RegExpMatchArray”分配给类型“boolean”。这里就是typescript的好处，会提示隐示的类型转换，避免可能出现的bug。字符串的match方法，可能返回Array或者null，所以这里的解决方案是改成

```typescript
env.isMiniProgram = env.ua.match(/miniprogram/i) != null
```

##### 3.4.2 第二个例子

```javascript
let tmpd = new Date(new Date(timestamp).toLocaleDateString())// 今日0点
tmpd = Math.floor(tmpd.getTime() / 1000)
return tmpd + 86400 - 1
```

错误提示：不能将类型“number”分配给类型“Date”。出错在第2行，右边计算出来是number，但是tmpd是Date类型，出现类型不匹配，可以简单粗暴的将tmpd改为any类型

```typescript
let tmpd : any = new Date(new Date(timestamp).toLocaleDateString())// 今日0点
tmpd = Math.floor(tmpd.getTime() / 1000)
return tmpd + 86400 - 1
```

##### 3.4.3 第三个例子

```javascript
var mistiming = Math.round(new Date() / 1000) - timestamp
```

错误提示：算术运算左侧必须是 "any"、"number"、"bigint" 或枚举类型。原因是Date类型和number类型做除法运算，需要修改为

```typescript
var mistiming = Math.round(new Date().getTime() / 1000) - timestamp
```

#### 3.5 类型“Object”上不存在属性“tipmdl”

```javascript
if (typeof env.authQuery.tipmdl !== 'undefined' && env.authQuery.tipmdl != '') {
    env.urlmodule = env.authQuery.tipmdl
}
```

这里authQuery是一个Object，里边存储了url的参数对，可能有任意key，所以修改成

```typescript
if (typeof env.authQuery['tipmdl'] !== 'undefined' && env.authQuery['tipmdl'] != '') {
    env.urlmodule = env.authQuery['tipmdl']
}
```

### 4. 给typescript代码加上注释

待补充

### 5. 参考资料

- Vue-cli3项目引入Typescript https://juejin.im/post/5da6e5c1f265da5b8c03c58f
