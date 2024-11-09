---
layout: post
title: "npm和pnpm对第三方库的安装和查找区别"
date: 2024-11-04 13:04:06 +0800
comments: true
categories: web
---

### npm 和 pnpm 安装的区别

npm是将依赖安装到项目根目录node_modules里。

pnpm虽然也将依赖安装到node_modules里，但真实的包是在.pnpm里的，每个node_modules里都指向.pnpm里的唯一数据。

举个例子，package.json的内容如下

```json
{
    "dependencies": {
        "A": "^1.0.0",
        "B": "^1.0.0"
    }
}
```

其中A依赖了C@2.0.0，B依赖了C@3.0.0，两种安装方式的目录结构如下：

* npm安装

```
node_modules
    A@1.0.0
        node_modules
            C@2.0.0
    B@1.0.0
        node_modules
            C@3.0.0
```

C如果安装在node_modules里，会造成版本冲突，所有将不同版本的C安装到了子node_modules里。当然，如果C的版本相同，会提取到根目录的node_modules里，这样不会造成冲突，也节省空间，但会丢失依赖树结构。

* pnpm安装
  
  最后用pnpm安装的目录为

```
node_modules
    .pnpm
        A@1.0.0
            node_modules
                A
                C -> .pnpm/C@2.0.0/node_modules/C
        B@1.0.0
            node_modules
                B
                C --> .pnpm/C@3.0.0/node_modules/C
        C@2.0.0
            node_modules
                C
        C@3.0.0
            node_modules
                C
    A -> .pnpm/A@1.0.0/node_modules/A
    B -> .pnpm/B@1.0.0/node_modules/B
```

可以看到：

* pnpm所有安装的库都在.pnpm目录下，每个版本号一个单独的目录，其余的地方都是引用到这里的

* 根node_modules下只有项目直接使用的库A和B，指向.pnpm对应位置

* A和B的子依赖放在子node_modules目录里，并指向唯一的存储.pnpm目录

pnpm解决了版本冲突的问题，节省了硬盘空间，并保留了依赖树结构。

### pnpm安装后库的查找过程

1. 代码引用A@1.0.0，在node_modules/A查找，实际位置是node_modules/.pnpm/A@1.0.0/node_modules/A

2. A依赖C，向上一级目录的node_modules里能查找到C，即node_modules/.pnpm/A@1.0.0/node_modules/C，实际指向位置是.pnpm/C@2.0.0/node_modules/C

### vite项目打包遇到的问题和解决方案

在项目使用pnpm后，如果A引入了B，当pnpm install A时，如果项目不直接安装B，会在编译时出现报错，查找不到B模块

经过查找资料，发现vite打包用的rollup，只能打包相对模块，对于其他模块是不会被打包到bundle里的。

解决方案在vite.config.ts文件中增加配置

```javascript
import { defineConfig } from "vite";
import uni from "@dcloudio/vite-plugin-uni";
import { nodeResolve } from '@rollup/plugin-node-resolve';

// https://vitejs.dev/config/
export default defineConfig({
  plugins: [
    uni(),
  ],
  build: {
    rollupOptions: {
      plugins: [nodeResolve()],
    }
  }
});
```

参考资料 [Troubleshooting | Rollup](https://rollupjs.org/troubleshooting/#warning-treating-module-as-external-dependency)

除了以上方法，还发现另外一个解决方案，将第三方库pnpm install的时候进行平铺安装，需要在.npmrc文件里增加配置

```
shamefully-hoist = true
```

这种方式安装，会将所有的库都安装在node_modules下，这时候是可以成功查找二级依赖并打包成功的。
