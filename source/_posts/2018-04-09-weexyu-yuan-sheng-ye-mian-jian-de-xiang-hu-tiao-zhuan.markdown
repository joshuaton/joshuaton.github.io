---
layout: post
title: "weex与原生页面间的相互跳转"
date: 2018-04-09 15:33:36 +0800
comments: true
categories: ios
---
我们在现有应用中集成Weex，难免会遇到Weex页面与原生页面相互跳转的问题。通常的一种场景是，某一个中间的原生页面我们用Weex来替换，这样就存在原生页面跳转到Weex页面，再由Weex页面跳转到原生页面的场景。这篇文章讲述如何实现这种场景。

<!--more-->

###1. 原生页面跳转到Weex页面
这个其实很简单。首先需要知道的是，Weex页面其实也是一个原生页面，只是这个原生页面的View是由Weex渲染出来的而已。所以这一步实际上是原生页面跳原生页面，用navigationController的push方法就可以了。代码如下

```objc
IGWeexDemoViewController *vc = [IGWeexDemoViewController alloc] init];
[[CPBaseViewController currentViewController].navigationController pushViewController:vc animated:YES];
```

###2. Weex页面跳转到原生页面
Weex要跳回原生页面，是需要借用原生代码的能力的。实现方式就是自定义Module，Weex将需要跳转的页面和参数传递个Module，然后Module利用原生代码，控制navigationController进行跳转。

首先自定义跳转的Module，代码如下：

```objc
@implementation WXNavigationModule

WX_EXPORT_METHOD(@selector(navigationToUrl:))

-(void)navigationToUrl:(NSString *)url{
    [IGJumpMediator jumpActionWithUrl:url];
    [[CPBaseViewController currentViewController].navigationController setNavigationBarHidden:NO animated:YES];

}

@end
```

关键在方法中的第一行代码 [IGJumpMediator jumpActionWithUrl:url]，通过拿到Weex里传递进来的参数url，进行解析，然后利用Runtime机制，通过字符串得到要跳转的ViewController类，然后再设置需要的初始化参数，最后push就可以了。方法具体的代码实现这里不再赘述。

第二行代码是设置了导航栏为显示，下边的篇幅讲下为什么要这样设置。

###3. Weex页面中的自定义导航栏实现
Weex页面我没有用原生的导航栏，而是参考了一些资料和代码自己实现了导航栏，主要的原则是，一切可以在Weex中实现的，都放在Weex中，不放在原生代码中，这样可以更好的进行跨平台代码复用。

原生跳转到Weex的时候，先隐藏原生里的导航栏。Weex调回原生的时候，在显示原生导航栏。所以有了上边那一行代码。

在Weex中实现导航栏，最好封装成一个组件，这样方便所有页面一起复用。以下是导航栏NavigationBar.vue的代码。

模板部分

```html
<template>
    <div>
        <!--iPhoneX -->
        <div class="iPhoneXDiv navbar" v-if="isiPhoneX"></div>
        <!--其他iOS设备 -->
        <div class="iOSDiv navbar" v-else-if="isiOS"></div>
        <!--安卓设备 -->
        <div class="android navbar" v-else-if="isAndroid"></div>

        <div class="subviews">
            <!--Title-->
            <text class="titletext">{{titleText}}</text>
            <!--左边图片-->
            <div class="left" @click="leftButtonClicked" v-if="showLeft">
                <image src="local:///team_navi_back@2x.png" class="left-button"></image>
            </div>

            <div class="right" @click="rightButtonClicked" v-if="showRight">
                <!--如果显示右边item ， 图片或者文字 2选1 -->
                <text class="right-text" v-if="rightText">{{rightText}}</text>
                <image :src="rightImage" class="left-button" v-if="rightImage"></image>
            </div>
        </div>
    </div>
</template>
```

样式部分

```css
<style scoped>
    .iPhoneXDiv {
        height: 86px;
    }

    .iOSDiv {
        height: 38px;
    }

    .android {
        height: 0px;
    }

    .navbar {
        width: 750px;
        top: 0px;
        left: 0px;
        background-color: #1A1B2F;
    }

    /*大佈局样式*/
    .subviews {
        height: 90px;
        width: 750px;
        left: 0px;
        background-color: #1A1B2F;
    }

    /*中间文字样式*/
    .titletext {
        font-size: 36px;
        color: white;
        position: relative;
        width: 750px;
        height: 90px;
        text-align: center;
        line-height: 90px;
        bottom: 0;
    }

    /*左边图片*/
    .left {
        justify-content: center;
        position: absolute;
        align-items: center;
        flex: 1;
        height: 90px;
        left: 30px;
        bottom: 0px;
        width: 50px;
    }

    /*图片按钮*/
    .left-button {
        width: 50px;
        height: 50px;
        /*margin-left: 40px;*/
    }

    /*右边图片*/
    .right {
        justify-content: center;
        position: absolute;
        align-items: center;
        flex: 1;
        height: 90px;
        right: 20px;
        bottom: 0px;
        width: 50px;
        /*background-color: white;*/
    }

    /*右边文字*/
    .right-text {
        font-size: 30px;
        color: white;
        position: absolute;
        right: 20px;
        width: 100px;
        height:90px;
        text-align: center;
        bottom: 0;
    }
</style>
```

js代码部分

```javascript
var device = weex.config.env;
const Navigator = weex.requireModule('navigator');

export default {
    /*props 属性列表*/
    props: {
        /*返回图片*/
        leftImage: {
            type: String,
            default: "img/other/backbtn.png"
        },
        /*Title*/

        titleText: {
            type: String,
            default: "Title"
        },
        /*是否显示左边图片*/
        showLeft: {
            type: Boolean,
            default: true
        },
        /*showLeft=true时,左边是否是点击返回事件，否，则显示其他图片，重新给leftImage属性赋值*/
        isBack: {
            type: Boolean,
            default: true
        },
        /*是否显示右边item*/
        showRight: {
            type: Boolean,
            default: false
        },
        /*右边文字*/
        rightText: {
            type: String,
            default: ""
        },

        /*右边图片*/
        rightImage: {
            type: String,
            default: ""
        }
    },
    data() {
        return {
            isiPhoneX: (device.platform === 'iOS') && (device.deviceWidth === 1125) && (device.deviceHeight === 2436),
            isiOS: (device.platform === 'iOS'),
            isAndroid: (device.platform === 'android'),
            TitleText: "",
        }
    },
    methods: {
        //左边点击事件
        leftButtonClicked() {
            if (this.showLeft) {
                if (this.isBack)  //点击pop返回
                {
                    console.log('click back');
                    Navigator.pop({}, e => {
                    });
                }
                else //其他操作
                {
                    console.log('LeftItemClicked');
                    this.$emit('LeftItemClicked');
                }
            }
        },
        //右边点击事件
        rightButtonClicked() {
            if (this.showRight) {

                console.log('RightItemClicked');
                this.$emit('RightItemClicked');
            }
        },
    }
};
```
导航栏支持设置标题，左边和右边部分的显隐，左图片，右图片和右文字。对不同的平台做了下适配。

调用的时候比较简单，在页面头部加上如下代码就可以了

```javascript
<NavBar
    :show-left="true"
    :title-text="titleText"
    :show-right="false">
</NavBar>
```

### 4.效果
最后看一下效果，看得出来哪部分是原生实现，哪部分是Weex实现吗
[![](http://jason5.cn/images/weex-navi-demo.gif)](http://jason5.cn/images/weex-navi-demo.gif)


***

参考资料

- [Weex系列（1）-App端自定义导航条](https://www.jianshu.com/p/e9bbd8a2244a)

