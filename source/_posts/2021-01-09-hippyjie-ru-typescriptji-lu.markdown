---
layout: post
title: "Hippy接入typescript记录"
date: 2021-01-09 17:32:08 +0800
comments: true
categories: web
---

项目的package.json如下

```javascript
  "scripts": {
    "dev": "vue-cli-service serve --https true --port 443 --host test.igame.qq.com",
    "build": "vue-cli-service build",
    "lint": "eslint --ext .js,.vue,.ts .",
    "lint:fix": "eslint --fix --ext .js,.vue,.ts .",
    "hippy:debug": "hippy-debug",
    "hippy:dev": "webpack --config ./scripts/hippy-webpack.dev.js",
    "hippy:vendor": "webpack --config ./scripts/hippy-webpack.ios-vendor.js --config ./scripts/hippy-webpack.android-vendor.js",
    "hippy:build": "webpack --config ./scripts/hippy-webpack.ios.js --config ./scripts/hippy-webpack.android.js",
    "hippy:publish": "hippy-publish igame-match-test",
    "hippy:publish:prod": "NODE_ENV='release' hippy-publish igame-match-release"
  },
```

可以看到，如果是sudo npm run dev的方式来启动服务的话，用的是vue-cli-service，这个是@vue/cli-service的命令，其默认是去读取根目录的配置文件vue.config.js。如果是以npm run hippy:dev 去启动的话，读取的是配置文件hippy-webpack.dev.js。由于两个配置文件不一样，所以，对ts的支持需要两边配置文件都要修改。

<!--more-->

## 1. vue.config.js的配置

vue.config.js这边，我们不需要去配置ts-loader，参考现有的项目igame-match，只要安装@vue/cli-plugin-typescript和typescript，就可以自动帮我们配置好。这里有个问题，如果不在vue.config.js里加上entry配置的话，默认项目启动会去寻找main.ts文件，这个文件是不存在的，所以这里起不来，解决方法是修改entry文件就可以了，在vue.config.js里加上如下配置

```javascript
const path = require('path');

module.exports = {
  runtimeCompiler: true,
  configureWebpack: {
    entry: path.resolve(__dirname, 'src', 'main.js'),
  },
};
```

这里会用到的两个小技巧

1. 初始化tsconfig.json文件
   
   ```shell
   tsc init
   ```

2. 检查webpack的配置，由于vue.config.js会对webpack的配置进行一些封装，并不是很直观，所以可以运行以下的命令，查看真正的配置
   
   ```shell
   vue inspect
   vue inspect --rule vue
   ```
   
   详细参考 [webpack 相关 | Vue CLI](https://cli.vuejs.org/zh/guide/webpack.html#%E4%BF%AE%E6%94%B9%E6%8F%92%E4%BB%B6%E9%80%89%E9%A1%B9)

## 2. hippy-webpack.dev.js的配置

hippy-webpack.dev.js这边，需要加上ts-loader对ts文件进行解析，加上如下的配置

```javascript
modules: {
    rules: [
        {
            test: /\.(ts|tsx)?$/i,
            use: [{
              loader: 'ts-loader',
              options: {
                appendTsSuffixTo: [/\.vue$/],
              },
            }],
            exclude: /node_modules/,
        }
    ] 
},
resolve: {
   extensions: ['.js', '.vue', '.json', '.ts']
}
```
