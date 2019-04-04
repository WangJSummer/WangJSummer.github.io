title: GCD并发多个请求(如Afn请求)
author: Jian Wang
tags: [Objective-C,GCD]
categories: iOS
date: 2019-04-04 15:34:15
---
**本篇笔记主要针对的场景问题：**需要并发执行多个AFNetworking请求，并且在多个请求成功返回结果之后，根据它们的结果来执行下一个任务。
#### 一 dispatch_group 介绍
**1.1 基本概念**：将追加到队列的一系列任务放进组中，可用于监听任务完成情况。

**1.2 常用方法**：
- dispatch_group_create() 创建一个调度任务组。
- disaptch_group_async (dispatch_group_t,dispatch_queue_t,block) 将一个追加到队列的任务提交到任务组中。
- dispatch_group_enter/dispatch_group_leave ， 跟dispath_group_async类似，可理解为手动管理任务添加进组和任务离开组。
- dispatch_group_notify 用来监听组中的任务全部执行完毕。
- dispatch_group_wait (dispatch_group_t ,dispatch_time_t) 设置的等待时间，**等待时间内所在的线程停止向下执行**，时间结束后，如果组中的任务执行完毕，则返回0，如果没有只执行完毕则返回非0。
   - 通常时间设置为dispatch_time_forever,永久等待，此时当组中的任务全部执行完毕后，wait方法就会返回。

#### 二 实际使用场景（并发执行第多个任务）
##### 2.1 并发执行非异步任务
**需求描述**：并发执行任务1和任务2，当它们执行结束后，执行任务3。

**分析**：使用dispatch_group_async搭配dispatch_group_notify即可实现并行执行任务1和任务2 ，并且监听两个任务执行完毕之后，再执行任务3。

###### 2.1.1 代码解析
``` swift
- (void)asyncWithGroup {
    //创建group
    dispatch_group_t group = dispatch_group_create();
    //全局并行队列
    dispatch_queue_t globalQueue = dispatch_get_global_queue(DISPATCH_QUEUE_PRIORITY_DEFAULT, 0);
    
    dispatch_group_async(group, globalQueue, ^{
        NSLog(@"执行任务--1--");
        sleep(1);
        NSLog(@"执行结束--1--");
    });
    
    dispatch_group_async(group, globalQueue, ^{
        NSLog(@"执行任务--2--");
        sleep(3);
        NSLog(@"执行结束--2--");
    });
    
    //监听组中任务全部执行完毕
    dispatch_group_notify(group, globalQueue, ^{
        NSLog(@"开始执行任务--3--");
    });
}
```
打印情况如下：
![image.png](https://upload-images.jianshu.io/upload_images/2203462-06856643e1e2d8af.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
可见任务1和任务2是并发执行，并且在他们执行完毕之后，才执行任务3。

##### 2.2 并发执行异步任务
**需求描述**：并行执行任务1和任务2，任务1和任务2这里都是使用AFNnetworking发送请求，等待任务1和任务2请求完成（成功或失败）之后，再继续执行任务3。

**分析**：
- 对于AFNetworking的请求，难点在于是不知道何时返回结果。如果使用dispatch_group_async的话，那么在afn请求调用之后，该任务即认定结束了（**示例看下面的注意部分**）。
- 那么此时有两种实现方法可以解决这个问题。
   1. 使用dispatch_group_enter/leave 来手动管理group中任务执行是否结束。
      - dispatch_group_async可理解为自动管理任务的进组和离开组。
   2. 使用group搭配dispatch_semaphore信号量来解决。

###### 2.2.1 代码解析（这里主要针对第一种实现方法做说明）
```
//请求连接
static NSString *url = @"http://httpbin.org/get";

/**
 封装afn请求

 @param url 使用定义的url
 @param result 用于简化，将success和failure统一描述
 */
- (void)loadDataWithURL:(NSString *)url result:(void(^)(BOOL isSuccess))result {
    
    __block BOOL isSuccess = NO;
    [self.manager GET:url parameters:nil progress:nil success:^(NSURLSessionDataTask * _Nonnull task, id  _Nullable responseObject) {
        isSuccess = YES;
        result(isSuccess);
    } failure:^(NSURLSessionDataTask * _Nullable task, NSError * _Nonnull error) {
        isSuccess = NO;
        result(isSuccess);
    }];
}

/**
 使用group并发执行两个网络请求任务，三个任务执行完之后执行任务3
 */
- (void)loadDataConcurrentWithGroup {
    //创建group
    dispatch_group_t group = dispatch_group_create();
    //全局并发队列
    dispatch_queue_t globalQueue = dispatch_get_global_queue(DISPATCH_QUEUE_PRIORITY_DEFAULT, 0);
    
    dispatch_group_enter(group);
    dispatch_async(globalQueue, ^{
        NSLog(@"任务--1--开始");
        [self loadDataWithURL:url result:^(BOOL isSuccess) {
            NSLog(@"任务--1--完成");
            dispatch_group_leave(group);
        }];
    });
    
    dispatch_group_enter(group);
    dispatch_async(globalQueue, ^{
        NSLog(@"任务--2--开始");
        [self loadDataWithURL:url result:^(BOOL isSuccess) {
            NSLog(@"任务--2--完成");
            dispatch_group_leave(group);
        }];
    });
    
    dispatch_group_notify(group, globalQueue, ^{
        dispatch_async(dispatch_get_main_queue(), ^{
            NSLog(@"开始执行任务--3--");
        });
    });
}
```
打印结果如下图所示：
![image.png](https://upload-images.jianshu.io/upload_images/2203462-d6f3ef1e540682e2.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
可见任务1和任务2是并发执行，等到任务1和任务2请求成功返回请求结果后，才执行任务3。

**注意：**上述代码如果将dispatch_group_leave(group)放在afn请求外面的话（如下所示），会是什么结果呢？
```
    dispatch_async(globalQueue, ^{
        NSLog(@"任务--1--开始");
        [self loadDataWithURL:url result:^(BOOL isSuccess) {
            NSLog(@"任务--1--完成");
//            dispatch_group_leave(group);
        }];
        dispatch_group_leave(group);
    });
```
此时在loadDataWithURL调用之后，发送afn请求，然后就会执行dispatch_group_leave（group），那么此时其实跟调用dispatch_group_async是一样的意思了。并不能监控到两个afn请求的成功返回。所以运行打印结果如下图所示：
![image.png](https://upload-images.jianshu.io/upload_images/2203462-8710d792e3947f96.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

###### 2.2.2 使用dispatch_group和disaptch_semaphore信号量实现方法会在dispatch_semaphore笔记中介绍。

**结语：** 路漫漫其修远兮，吾将上下而求索。