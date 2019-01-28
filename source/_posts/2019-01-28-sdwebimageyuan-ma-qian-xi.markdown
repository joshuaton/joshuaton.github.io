---
layout: post
title: "SDWebImage源码解读"
date: 2019-01-28 16:21:16 +0800
comments: true
categories: ios
---
SDWebImage是最流行的iOS第三方图片加载库，也是github上star数目最多的objective-c第三方库。这篇文章对SDWebImage的源码进行简单的分析，主要是分析代码的执行流程。源码版本是目前最新的稳定版本4.4.4。

###源码目录
SDWebImage下分为以下几个目录

* Downloader: 负责图片的下载，基于`NSURLSession`实现，主要类是`SDWebImageDownloader`
* Cache: 负责图片的缓存，主要是对内存和磁盘进行读写，实现二级缓存功能，主要类是`SDImageCache`
* Decoder: 负责图片的解码功能，以支持不同格式的图片，例如`SDWebImageGIFCoder`是对GIF图片的支持
* Utils: 封装的工具类，主要类是`SDWebImaegeManger `，控制整个图片下载和缓存流程
* Categories: 一些扩展支持，比如如果要对GIF图片进行下载展示，需要引入`UIImage+GIF.h`（框架已默认引入）
* WebCache Categories: 对`UIView`的扩展支持，最常用的是`UIImageView+WebCache.h`，实现了`UIImageView`的图片加载和缓存功能
* FLAnimatedImage: 对`FLAnimatedImage`进行了扩展，可以对动态图片进行加载和缓存


###调用时序图
以官方Demo详情页加载图片为例，加载图片的时序图如下：

[![](https://jason5.cn/images/2018-01/SDWebImage-Source-UML.jpg)](https://jason5.cn/images/2018-01/SDWebImage-Source-UML.jpg)

1. 调用`sd_internalSetImageWithURL:placeholderImage:options:operationKey:internalSetImageBlock:progress:completed:context:`
2. 调用`loadImageWithURL:options:progress:completed:`
3. 调用`queryCacheOperationForKey:cacheOptions:done`查询缓存
4. 调用`imageFromMemoryCacheForKey`查内存, 调用`diskImageDataBySearchingAllPathsForKey`查磁盘
5. 调用`downloadImageWithURL:options:progress:completed:`从网络下载
6. 调用`storeImage:imageData:forKey:toDisk:completion:`将结果缓存

###源码分析
在官方Demo中，有一个列表页和详情页，我们从更简单的详情页来分析，详情页只有一个FLAnimatedImage控件，功能就是加载了一张图片，代码如下：

```c++
__weak typeof(self) weakSelf = self;
[self.imageView sd_setImageWithURL:self.imageURL
                  placeholderImage:nil
                           options:SDWebImageProgressiveDownload
                          progress:^(NSInteger receivedSize, NSInteger expectedSize, NSURL *targetURL) {
                              dispatch_async(dispatch_get_main_queue(), ^{
                                  float progress = 0;
                                  if (expectedSize != 0) {
                                      progress = (float)receivedSize / (float)expectedSize;
                                  }
                                  weakSelf.progressView.hidden = NO;
                                  [weakSelf.progressView setProgress:progress animated:YES];
                              });
                          }
                         completed:^(UIImage *image, NSError *error, SDImageCacheType cacheType, NSURL *imageURL) {
                             weakSelf.progressView.hidden = YES;
                             [weakSelf.activityIndicator stopAnimating];
                             weakSelf.activityIndicator.hidden = YES;
                         }];
```

对外暴露的接口比较简单，传递需要加载图片的url，placeholder图片，加载选项options，加载过程回调progressBlock，完成回调completedBlock就可以了。

接下来到时序图的第(1)步，调用`UIView+WebCache`的`sd_internalSetImageWithURL:`方法，代码如下：

```c++
[self sd_internalSetImageWithURL:url
                    placeholderImage:placeholder
                             options:options
                        operationKey:nil
               internalSetImageBlock:^(UIImage * _Nullable image, NSData * _Nullable imageData, SDImageCacheType cacheType, NSURL * _Nullable imageURL) {
                           ……
                       }
                            progress:progressBlock
                           completed:completedBlock
                             context:@{SDWebImageInternalSetImageGroupKey: group}];
```

继续跟进这个方法，时序图到第(2)步，调用`SDWebImageManager`的`loadImageWithURL:`方法，代码如下：

```c++
id <SDWebImageOperation> operation = [manager loadImageWithURL:url options:options progress:combinedProgressBlock completed:^(UIImage *image, NSData *data, NSError *error, SDImageCacheType cacheType, BOOL finished, NSURL *imageURL) {
	//加载完图片的回调
	……
}
```

时序图来到第(3)步，`SDWebImageManager`会调用`SDImageCache`的`queryCacheOperationForKey`来进行缓存查询

```c++
//SDImageCache.m
- (nullable NSOperation *)queryCacheOperationForKey:(nullable NSString *)key options:(SDImageCacheOptions)options done:(nullable SDCacheQueryCompletedBlock)doneBlock {
    
    // 读内存缓存操作
    UIImage *image = [self imageFromMemoryCacheForKey:key];
    BOOL shouldQueryMemoryOnly = (image && !(options & SDImageCacheQueryDataWhenInMemory));
    if (shouldQueryMemoryOnly) {
        if (doneBlock) {
            doneBlock(image, nil, SDImageCacheTypeMemory);
        }
        return nil;
    }
    
    NSOperation *operation = [NSOperation new];
    void(^queryDiskBlock)(void) =  ^{
    	//读磁盘缓存
    	……  
    };
    
    if (options & SDImageCacheQueryDiskSync) {
        queryDiskBlock();
    } else {
        dispatch_async(self.ioQueue, queryDiskBlock);
    }
    
    return operation;
}
```

可以看到，在上边的代码中，调用`imageFromMemoryCacheForKey `先从内存里查询是否有图片缓存，调用`dispatch_async(self.ioQueue, queryDiskBlock)`从磁盘里查询是否有图片缓存。这是时序图中的第(4)步。如果命中了缓存，则直接回调完成block，不再走下边的流程了。如果没有命中缓存，那么继续下边。

流程来到时序图中的第(5)步，查询完缓存返回后，`SDWebImageManager`会调用`SDWebImageDownloader`的`downloadImageWithURL:`方法从网络下载图片

```c++
//SDWebImageDownloader.m
- (nullable SDWebImageDownloadToken *)downloadImageWithURL:(nullable NSURL *)url
                                                   options:(SDWebImageDownloaderOptions)options
                                                  progress:(nullable SDWebImageDownloaderProgressBlock)progressBlock
                                                 completed:(nullable SDWebImageDownloaderCompletedBlock)completedBlock {
        
    LOCK(self.operationsLock);
    NSOperation<SDWebImageDownloaderOperationInterface> *operation = [self.URLOperations objectForKey:url];
    // There is a case that the operation may be marked as finished or cancelled, but not been removed from `self.URLOperations`.
    if (!operation || operation.isFinished || operation.isCancelled) {
        operation = [self createDownloaderOperationWithUrl:url options:options];
        __weak typeof(self) wself = self;
        operation.completionBlock = ^{
            __strong typeof(wself) sself = wself;
            if (!sself) {
                return;
            }
            LOCK(sself.operationsLock);
            [sself.URLOperations removeObjectForKey:url];
            UNLOCK(sself.operationsLock);
        };
        [self.URLOperations setObject:operation forKey:url];
        // Add operation to operation queue only after all configuration done according to Apple's doc.
        // `addOperation:` does not synchronously execute the `operation.completionBlock` so this will not cause deadlock.
        //执行下载操作
        [self.downloadQueue addOperation:operation];
    }
    UNLOCK(self.operationsLock);

    id downloadOperationCancelToken = [operation addHandlersForProgress:progressBlock completed:completedBlock];
    
    SDWebImageDownloadToken *token = [SDWebImageDownloadToken new];
    token.downloadOperation = operation;
    token.url = url;
    token.downloadOperationCancelToken = downloadOperationCancelToken;

    return token;
}
```

从网络下载图片的代码如上，第8行生成`NSOperation `，第26行加入叫做`downloadQueue`的`NSOperationQueue`执行。

下载完成后，时序图流程来到第（6）步，`SDWebImageManager`会调用`SDImageCache`的`storeImage:`方法将结果进行缓存，以便下次使用，然后再进行成功的回调，代码如下：

```objective-c
UIImage *transformedImage = [self.delegate imageManager:self transformDownloadedImage:downloadedImage withURL:url];
                                
if (transformedImage && finished) {
    BOOL imageWasTransformed = ![transformedImage isEqual:downloadedImage];
    NSData *cacheData;
    // pass nil if the image was transformed, so we can recalculate the data from the image
    if (self.cacheSerializer) {
        cacheData = self.cacheSerializer(transformedImage, (imageWasTransformed ? nil : downloadedData), url);
    } else {
        cacheData = (imageWasTransformed ? nil : downloadedData);
    }
    [self.imageCache storeImage:transformedImage imageData:cacheData forKey:key toDisk:cacheOnDisk completion:nil];
}
    
[self callCompletionBlockForOperation:strongSubOperation completion:completedBlock image:transformedImage data:downloadedData error:nil cacheType:SDImageCacheTypeNone finished:finished url:url];
```

到这里图片加载流程结束。

SDWebImage里还有很多实现细节，比如多线程的控制和加锁，各种控制下载行为的选项，取消正在下载的操作，URL参数的容错，PlaceHolder的设置，加载IndicatorView的显示控制，下载过程Progress的控制等。由于篇幅原因在本文中省略，等以后有时间再细致分析。



