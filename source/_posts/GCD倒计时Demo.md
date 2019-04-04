title: GCD倒计时Demo
author: Jian Wang
tags:
  - Objective-C
  - GCD
categories: iOS
date: 2019-04-04 16:06:12
---
#### 简介 
这次的项目里面有涉及到商品的倒计时功能，UIcollectionview中罗列商品进行倒计时，以前顶多就是密码找回，获取验证码，或者轮播图简单的使用了一下NSTimer。之前还因为timer使用不恰当造成了内存持续飙高，直接crash的情况。。。销毁timer需要使用invalidate，直接设置nil并不能销毁。

这次使用GCD定时器进行功能简单实现，其中也是找了一些资料，看了其他大神的代码，然后自己写了一个demo，仅作为笔记方便以后自己查看，完善。
[demo地址]( https://github.com/w467364316/GoodTimerDemo.git)

**示例图**：
![demo图.png](http://upload-images.jianshu.io/upload_images/2203462-b82a6a996dd07c05.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

##### 代码解析
---
放代码：
工具类.h

```
@interface CutDown : NSObject

-(void)creatTimerWit:(void(^)())completeBlock;

-(void)destroyTimer;
@end
```
创建定时器，设定间隔时间为0.1秒
```

@interface CutDown ()

@property(nonatomic,strong) dispatch_source_t timer;

@end

@implementation CutDown
-(void)creatTimerWit:(void(^)())completeBlock{

    if (_timer == nil) {
        dispatch_queue_t queue = dispatch_get_global_queue(DISPATCH_QUEUE_PRIORITY_DEFAULT, 0);
        _timer = dispatch_source_create(DISPATCH_SOURCE_TYPE_TIMER, 0, 0, queue);
        dispatch_source_set_timer(_timer, dispatch_walltime(NULL, 0), 0.1*NSEC_PER_SEC, 0);
        dispatch_source_set_event_handler(_timer, ^{
           dispatch_async(dispatch_get_main_queue(), ^{
               completeBlock();
           });
        });
        dispatch_resume(_timer);
    }
}

-(void)destroyTimer{
    if (_timer) {
        dispatch_source_cancel(_timer);
        _timer = nil;
    }
    
}
@end
```
viewController内部，在viewDidLoad方法中调用cutDown 创建定时器
```
- (void)viewDidLoad {
    
    [super viewDidLoad];
    // Do any additional setup after loading the view from its nib.
    
    self.myCollection.backgroundColor = [UIColor whiteColor];
    self.dataArray = [NSMutableArray array];
    [self.myCollection registerNib:[UINib nibWithNibName:@"TimerCollectionViewCell" bundle:nil] forCellWithReuseIdentifier:@"timeCell"];
    
    NSString *path = [[NSBundle mainBundle] pathForResource:@"GoodsTimes" ofType:@".plist"];
    NSArray *array = [NSArray arrayWithContentsOfFile:path];
    
    for (NSDictionary *dic in array) {
        GoodTimeModel *model = [GoodTimeModel initWithDic:dic];
        [self.dataArray addObject:model];
    }
    //开始进行计时
    self.cutDown = [[CutDown alloc] init];
    __weak typeof(self) weakSelf = self;
    [self.cutDown creatTimerWit:^{
        [weakSelf updateVisialCells];
    }];
}
```
![plist.png](http://upload-images.jianshu.io/upload_images/2203462-7a479d21dd209a6a.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
GoodTimeModel参数
```
@interface GoodTimeModel : NSObject
@property(nonatomic,copy) NSString *number;//序号
@property(nonatomic,copy) NSString *beginTime;//开始时间
@property(nonatomic,copy) NSString *time;//时间
@property(nonatomic,copy) NSString *minSecond;//毫秒

+(instancetype)initWithDic:(NSDictionary*)dic;
-(instancetype)initWithDic:(NSDictionary*)dic;

@end
```
更新当前视图内的cell的显示信息
```
/**
 *  更新cell
 */
-(void)updateVisialCells{
    
    NSArray *cells = self.myCollection.visibleCells;
    for (TimerCollectionViewCell *cell in cells) {
        GoodTimeModel *model = self.dataArray[cell.tag];
        int minSec = [model.minSecond intValue];
        if (minSec <=0) {
            minSec = 9;
        }else{
            minSec --;
        }
        model.minSecond = [NSString stringWithFormat:@"%d",minSec];
        cell.model = model;
    }
}
```
设置cell部分
```
-(UICollectionViewCell*)collectionView:(UICollectionView *)collectionView cellForItemAtIndexPath:(NSIndexPath *)indexPath{
    
    TimerCollectionViewCell *cell = [collectionView dequeueReusableCellWithReuseIdentifier:@"timeCell" forIndexPath:indexPath];
    GoodTimeModel *model = self.dataArray[indexPath.row];
    cell.model = model;
    cell.tag = indexPath.row;
    return cell;
}
```
TimerCollectionViewCell中进行计算显示剩余时间
```
-(void)setModel:(GoodTimeModel *)model{
    
    _model = model;
    self.numerLabel.text = model.number;
    self.minSecondLabel.text = model.minSecond;
    [self setTimeWithLastTime:model.time beginTime:model.beginTime];
}

/**
 *  开始进行倒计时
 *
 *  @param time 结束时间
 */
-(void)setTimeWithLastTime:(NSString*)time beginTime:(NSString*)beginTime{

    NSDateFormatter *formatter = [[NSDateFormatter alloc] init];
    [formatter setDateFormat:@"yyyy-MM-dd HH:mm:ss"];
    
    NSDate *endDate = [formatter dateFromString:time];
    NSDate *nowDate = [NSDate date];
    
    NSDate *beginDate = [formatter dateFromString:beginTime];
    NSTimeInterval beginTimeInterval = [beginDate timeIntervalSinceDate:nowDate];
    
    //剩余时间
    NSTimeInterval timeInterval = [endDate timeIntervalSinceDate:nowDate];
    self.minSecondLabel.hidden = YES;
    if (timeInterval <=0) {
        //活动结束
        self.timerLabel.text = @"活动结束";
    }else if (beginTimeInterval > 0){
         //活动未开始
          self.timerLabel.text = @"活动未开始";
    }else{
        self.minSecondLabel.hidden = NO;
        int day = (int)timeInterval/(3600*24);
        int hours = (int)(timeInterval - day*3600*24)/3600;
        int minus = (timeInterval - day*24*3600 - hours*3600)/60;
        int second = (timeInterval - day*3600*24 - hours*3600- minus*60);
        
        //小时
        NSString *finalHours = [NSString stringWithFormat:@"%d",day*24 + hours];
        if ([finalHours intValue] <10) {
            finalHours = [NSString stringWithFormat:@"0%@",finalHours];
        }
        //分钟
        NSString *finalMinutes = [NSString stringWithFormat:@"%d",minus];
        if ([finalMinutes intValue] <10) {
            finalMinutes = [NSString stringWithFormat:@"0%@",finalMinutes];
        }
        //秒
        NSString *finalSeconds = [NSString stringWithFormat:@"%d",second];
        if ([finalSeconds intValue] < 10) {
            finalSeconds = [NSString stringWithFormat:@"0%@",finalSeconds];
        }
        self.timerLabel.text = [NSString stringWithFormat:@"%@:%@:%@",finalHours,finalMinutes,finalSeconds];
    }
}
```
```
-(void)dealloc{
    
    NSLog(@"class = %s",object_getClassName(self));
    [_cutDown destroyTimer];
}
```