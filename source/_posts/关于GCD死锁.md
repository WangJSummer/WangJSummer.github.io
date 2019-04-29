title: 关于GCD死锁
tags:
  - GCD
  - Objective-C
categories: iOS
date: 2019-04-25 16:58:10
---
### 问题
有很多文章经常会说“在主线程使用了sync函数就会造成死锁”或者是“在主线程使用了sync函数，同时将任务传入串行队列就会死锁”。那么这些问题是否正确呢？   
答：不正确！！ **GCD死锁的原因是队列阻塞而不是线程阻塞。**

> 本文的分析基于已经了解了GCD的基本知识。
### 基本知识介绍

#### 串行队列和并行队列
参照下图：

![串行队列和并行队列](/images/pasted-9.png)

上图表明了几点：
- 串行队列和并行队列都是先进先出，区别在于其队列中任务的执行方式。
- 串行队列中，下一个任务会等待上一个任务结束后才会执行。
- 并行队列中，不会等待上一个任务完全执行结束，就可以立即调用执行下一个任务。


#### 同步函数和异步函数
##### dispatch_sync函数：
- 意味着”同步“，将指定的block”同步“的追加到指定的Dispatch_queue中。
- 再追加的block执行结束之前，dispatch_sync函数会一直等待。
- ”等待“也就意味着当前线程停止。如下图所示：

![upload successful](/images/pasted-16.png)
##### dispatch_async函数：
- 意味着”非同步“，将指定的block非同步的添加到指定的Dispatch_queue中。
- dispatch_async不会等待追加的block执行结束。如下图所示：

![upload successful](/images/pasted-17.png)

### 经典GCD死锁案例和解析
#### GCD主队列死锁
经典案例如下图所示：

![upload successful](/images/pasted-10.png)
我们发现仅仅执行打印了“开始执行”，然后报错，代码中使用了sync函数往主队列中追加了一个block，然后在主线程中执行。
##### 解析
上述代码中，表示的主队列中任务情况如下图所示：

![upload successful](/images/pasted-11.png)
按照主线程中执行的顺序进行梳理：
1. 主线程首先从主队列中提取任务viewDidLoad，并开始执行。
2. 代码执行第一行，打印“开始执行--main thread”。
3. 代码执行到第二行，使用dispatch_sync函数同步的往主队列中添加一个block，此时主队列中有了两个任务。
4. 基于dispatch_sync的特点，此时该同步函数处在等待状态，等待它添加的block处理执行完毕。
5. 程序出现崩溃。

并没有打印block中的内容，那么也就是**block没有执行。**   为什么不会执行呢？
1. dispatch_sync在它追加进指定队列的block处理执行结束前处于等待状态。
2. 此时dispatch_sync函数将block任务追加进了主队列中，主队列是串行队列，那么此时主队列中有了两个任务viewDidLoad和追加的block任务。
3. 串行队列中block任务要执行的话，那么就需要等待它前面的viewDidLoad任务执行结束。
4. 而viewDidLoad任务中，此时dispatch_sync函数处于等待状态，需要等到dispatch_sync函数返回后才能继续往下执行。
5. 由此就形成了一个死锁。
 - viewDidLoad等待dispatch_sync函数返回。
 - dispatch_sync函数等待block执行结束。
 - 而block任务等待viewDidLoad执行结束

如下图所示：


![upload successful](/images/pasted-18.png)

### 死锁解决方案
从上面的分析中，我们可以得出造成死锁的关键点：
1. dispatch_sync函数阻塞了当前线程
2. 串行队列中两个任务形成队列阻塞。

#### 针对sync函数阻塞了当前线程
dispatch_sync函数”同步“的添加block任务到队列，阻塞线程。那么我们使用dispatch_async函数”非同步“的将block任务添加进队列，不阻塞线程就可以解决。  
如下图所示：

![upload successful](/images/pasted-19.png)
可以发现没有报错，打印顺序为”开始执行“->”结束执行“->"执行中"。
##### 解析
按照执行流程进行梳理：
1. 主线程从主队列中取出viewDidLoad任务开始执行。
2. 打印”开始执行--main thread“。
3. dispatch_async异步将block任务追加到主队列中，并且不会等待block任务执行结束。
4. 打印”结束执行--main thread“。
5. 此时追加进主队列中的block任务因为viewDidLoad任务结束，开始执行并打印”执行中--main thread“。

#### 针对串行队列中的队列阻塞
上述造成死锁的主队列中，viewDidLoad因为dispatch_sync函数处于等待状态而不能执行结束，而block又需要等待viewDidLoad执行结束。那么有几种方法可以解决：
1. 自行创建一个队列，将block追加新建的队列中，这样两个任务就不会造成队列阻塞了。  
2. 将多个任务添加进并发队列中，这样任务执行就不用等待上一个任务执行结束了。

##### 新建队列
如下图所示：
![upload successful](/images/pasted-14.png)
创建了一个自定义串行队列”com.wjsuumer.top.test"（以下简称test队列）。然后使用dispatch_sync函数将block任务添加进该队列。运行后控制台依次打印 “开始执行”->"执行中"->“结束执行”。
###### 解析
对于自定义的串行队列，添加block任务后，那么此时两个队列的情况如下图所示：

![upload successful](/images/pasted-15.png)
按照执行流程进行梳理：
- 主线程从主队列取出任务viewDidLoad开始执行。
- 执行第一行，打印”开始执行--main thread“。
- 执行第二行，创建了一个test队列。
- 执行第三行，使用dispatch_sync函数往test队列中添加一个block任务，此时dispatch_sync处于等待状态。
- 主线程从test队列中取出block任务执行，打印”执行中--main thread“。
- 此时block执行结束，dispatch_sync函数返回，继续往下执行。
- 执行最后一行代码，打印”结束执行--main thread“。

相比之前直接在主队列中添加block任务，从test队列中取出block任务时，因为block任务就在test队列的对头，不需要等待其他任务就可以立即调用。

**注意：这个例子中我们只需要追加一个block任务，所以创建的队列无论是串行队列还是并行队列都可以，因为都是将block添加进了一个新的队列中
**
##### 使用并发队列
对于上述的新建队列的方法，如果新建的是串行队列，那么针对下列情况结果会是怎样的呢？

![upload successful](/images/pasted-20.png)

运行会发现依然会发生崩溃，自建test队列中发生死锁，test队列中有block1任务和block2任务，block1任务执行结束需要block2任务指定，block2任务指定又因为test队列为串行队列，所以需要等待block1任务执行完毕，从而造成死锁。   
如下图所示：

![upload successful](/images/pasted-21.png)
那么此时可能会想到针对block2任务，再新建一个队列，然后将block2追加到新的队列中去，这样就不会造成死锁了。这样自然可以，但是使用并发队列更加的简单。   
使用并发队列如下图所示：


![upload successful](/images/pasted-23.png)
###### 解析
我们将block1和block2追加进了新建的全局并发队列中，由于并发队列中的任务并发执行，不需要等待上一个任务执行结束，也就不会出现死锁的情况。

### 练习案例分析
如下代码所示，是否会发生死锁的情况？为什么？如果死锁了应该如何解决？
```
- (void)viewDidLoad {
    
    NSLog(@"开始执行--%@",[NSThread currentThread]);
    //使用async往主队列中添加block1任务
    dispatch_async(dispatch_get_main_queue(), ^{
        NSLog(@"执行中--1--%@",[NSThread currentThread]);
        //使用sync函数往主队列中添加block2任务。
        dispatch_sync(dispatch_get_main_queue(), ^{
            NSLog(@"执行中--2--%@",[NSThread currentThread]);
        });
    });
    NSLog(@"结束执行--%@",[NSThread currentThread]);
}
```
运行这段代码会出现崩溃，其中发生了死锁。


![upload successful](/images/pasted-25.png)
按照执行顺序进行梳理：
1. 主线程从主队列中取出viewDidLoad任务开始执行，打印**“开始执行--main thread”**。
2. dispatch_async函数异步将block1任务添加进主队列，同时不会处于等待状态。
3. 打印**“结束执行--main thread”**。
4. 此时viewDidLoad执行结束，开始执行添加的block1任务，打印**“执行中--1--main thread”**。
5. block1中使用dispatch_sync函数添加block2任务到主队列中，并且处于等待状态，等待block2任务执行完毕。
6. block2在主队列中，那么需要等待它前面的block1任务执行完毕。此时形成了死锁，程序崩溃。

那么我们为了解决这样的死锁问题，可以使用dispatch_sync函数“非同步”来追加block2任务。
### 结语
这篇笔记中，主要是讲解了dispatch_sync函数和diapatch_async将任务追加到串行队列中引发的死锁问题以及如何去解决。   
其中没有讲述过使用dispatch_async+并行队列的形式，这种情况下会开启子线程对任务进行处理，关于GCD多线程编程在其他笔记中详述。   
对于GCD的使用，还是需要理解其原理机制，才能够更好的配合实现性能的优化，避免出现死锁之类的错误。