---
layout: post
title: "Weex中js和Native的交互-iOS篇"
date: 2018-03-27 11:21:56 +0800
comments: true
categories: weex
---

终端集成Weex后，看了下js调用Native的方法，做了个展示列表的demo，做下记录。

![](http://jason5.cn/images/QQ20180327-105246.png)

<!--more-->

##Native端
要实现Weex调用Native，需要实现自定义的module，暴露相应的方法，并且注册。完成这个过程后，js里可以直接注册过的module中的方法，并且通过callback拿到返回结果。下边以weex调用native的网络模块获取数据并进行展示的例子，进行介绍。

###1. 实现WXModuleProtocol协议
自定义module，需要实现WXModuleProtocol协议

WXCustomNetworkModule.h

```objc
@interface WXCustomNetworkModule : NSObject<WXModuleProtocol>
@end
```

###2. 实现网络请求数据的方法，并且暴露给js

WXCustomNetworkModule.m

```objc
@interface WXCustomNetworkModule()

@property (nonatomic, strong) NSMutableArray *merchants;
@property (nonatomic, copy) NSString *pageIndex;

@end

@implementation WXCustomNetworkModule

WX_EXPORT_METHOD(@selector(getMerchantList:))

-(void)getMerchantList:(WXModuleCallback)callback{
    
    TipCredit_QueryMerchantCreditListReq *req = [[TipCredit_QueryMerchantCreditListReq alloc] init];
    req.userLng = [CPLocationAPI getLongitude];
    req.userLat = [CPLocationAPI getLatitude];
    req.pageidx = self.pageIndex;
    req.num = 10;
    
    @weakify(self);
    [TipNetWorkManager requestWithReq:req withRspClass:[TipCredit_QueryMerchantCreditListRsp class] withCmd:90086 successBlock:^(id respondObjc) {
        @strongify(self);
        TipCredit_QueryMerchantCreditListRsp *rsp = (TipCredit_QueryMerchantCreditListRsp *)respondObjc;
        self.pageIndex = rsp.pageidx;
        [self.merchants addObjectsFromArray:rsp.merchantcreditsArray];
        
        for(int i=0; i<self.merchants.count; i++){
            self.merchants[i] = [self.merchants[i] toJson];
        }
        callback(self.merchants);

    } failedBlock:^(id failedMsg, int resultCode) {
        @strongify(self);
    }];
}
```

getMerchantList是发送网络请求获取后台数据的方法。

```objc
WX_EXPORT_METHOD(@selector(getMerchantList:))
```
这行代码可以将此函数暴露给js调用。

### 3. 初始化的时候注册自定义module，并且指定module name
```objc
[WXSDKEngine registerModule:@"network" withClass:[WXCustomNetworkModule class]];
```
module name命名为network，表示网络模块。

### 4. js拿到回调数据
实现的方法，在最后可以加一个WXModuleCallback类型的callback参数

```objc
callback(self.merchants);
```

通过这行代码将结果回调给js。回调的参数类型支持NSString, NSArray, NSDictionary。所以这里回调之前，将网络返回的自定义类转换成了NSDictionary，再进行回调。

##js端
```javascript
<template>
  <list class="list">
    <cell class="cell" v-for="merchant in lists">
      <div class="panel">
        <text class="text">{{merchant.merchantinfo.merchantid}}</text>
      </div>
    </cell>
  </list> 
</template>

<script>
export default {
  name: 'App',
  components: {
    HelloWorld
  },
  data () {
    return {
      logo : 'https://gw.alicdn.com/tfs/TB1yopEdgoQMeJjy1XaXXcSsFXa-640-302.png',
      lists : []
    }
  },
  methods: {
  
  },
  created: function () {
    var self = this
    weex.requireModule('network').getMerchantList(function(rsp){
      self.lists = rsp
    })
  }
}
</script>
```
index.vue的关键代码如上。created方法在页面创建时候会执行。通过

```javascript
weex.requireModule('network').getMerchantList
```
这行代码，调用原生网络模块，并且拿到回调数据进行展示。

##总结
本文介绍了Weex中js与Native的交互方式。通过此方法，界面部分完全可以在js里实现，iOS和Android双端只写一份，原生部分只需要提供负责网络请求的module就可以了。
