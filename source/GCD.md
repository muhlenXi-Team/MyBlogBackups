---
title: 关于 GCD
date: 2016-10-09 10:02:59
categories: blog
tags: [GCD,Objective-C]
---

 *版权声明：本文为 muhlenXi 原创文章，欢迎转载，转载请注明来源: [http://muhlenxi.com/2016/10/09/GCD](http://muhlenxi.com/2016/10/09/GCD)*

#### 前言：

> GCD 是 Grand Central Dispatch 的缩写。GCD 是一个底层框架，它可以通过操作系统来管理并发和异步执行任务。

学习贵在记录和总结收获！点击阅读全文了解更多！　　

<!-- more -->

#### 正文

##### 在主线程中执行

```objc
dispatch_async(dispatch_get_main_queue(), ^{
   
   NSLog(@"主线程执行");
});
```

##### 异步线程执行

```objc
dispatch_async(dispatch_get_global_queue(0, 0), ^{
       
   NSLog(@"异步执行");
});
```

##### 一次线程执行

```objc
static dispatch_once_t onceToken; dispatch_once(&onceToken, ^{
        
    NSLog(@"只执行一次");
});
```

##### 延迟执行

```objc
dispatch_after(dispatch_time(DISPATCH_TIME_NOW, (int64_t)(5 * NSEC_PER_SEC)), dispatch_get_main_queue(), ^{
        
     NSLog(@"延迟五秒执行");
});
```

##### 在自定义线程中执行

```objc
dispatch_queue_t my_que = dispatch_queue_create("blog.muhlenxi.com", NULL);
dispatch_async(my_que, ^{
       
   NSLog(@"自定义线程汇中执行");
        
});
```
##### 多线程并行执行

```objc
dispatch_group_t group = dispatch_group_create();
    
dispatch_async(dispatch_get_global_queue(0, 0), ^{
        
        NSLog(@"并行线程，下载图片1");
        sleep(1);
});
    
dispatch_async(dispatch_get_global_queue(0, 0), ^{
        
        NSLog(@"并行线程，下载图片2");
        sleep(2);
});
    
dispatch_async(dispatch_get_global_queue(0, 0), ^{
        
        NSLog(@"并行线程，下载图片3");
        sleep(3);
});
    
dispatch_group_notify(group, dispatch_get_global_queue(0, 0), ^{
        
        NSLog(@"这里汇总下载图片结果");
        
});
```

*感谢您的阅读，一起学习，一起成长，加油！*