title: 简单自定义选择按钮(switchDemo)
author: Jian Wang
date: 2019-04-04 16:41:23
tags: [Objective-C,UI]
categories: iOS
---
#### 简介
虽然系统的UISwitch效果已经很好了,附带的动画效果也是很好的,但是在实际开发中UI和程序员对头(产品经理)经常会要求按照项目的整体效果使用其他的图片或者背景来代替,这里仅在项目中做了一个简单的自定义switch.

#### demo图样
![switchDemo.png](http://upload-images.jianshu.io/upload_images/2203462-95c34978f9e5af4e.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

主要是使用自定义UIView,利用背景图片的切换,和按钮图片的x的位置开实现.

这里把demo放上,方便以后查看 : [demo地址](https://github.com/w467364316/WJSwitchButton.git)

#### 代码解析
##### WJSwitch.h
```
#import <UIKit/UIKit.h>

@protocol WJSwitchViewDelegate <NSObject>

/**
 *  开关是否开启
 *
 *  @param isopen YES 开启  NO 关闭
 *  @param tag   tag值
 */
-(void)WJSwitchViewDelegateWithSwitchStates:(BOOL)isopen withTag:(NSInteger)tag;

@end

@interface WJSwitchView : UIView

@property(nonatomic,weak) id <WJSwitchViewDelegate> delegate;//代理


-(instancetype)initWithFrame:(CGRect)frame withTag:(NSInteger)tag delegate:(id)delegate;

@end
```

WJSwitch.m
```
#import "WJSwitchView.h"

@interface WJSwitchView (){

    BOOL isOpen;//是否开启(右边为开启)

}

@property(nonatomic,strong) UIImageView  * bgView;//背景图片

@property(nonatomic,strong) UIImageView  * btnView;//按钮图片

@property(nonatomic,assign) CGFloat beginpoint;//开始的位置

@end

@implementation WJSwitchView


-(instancetype)initWithFrame:(CGRect)frame withTag:(NSInteger)tag delegate:(id)delegate{

    
    if (self = [super initWithFrame:frame]) {
        
        [self setFrame:frame];
        [self setBtnImageView];
        self.tag = tag;
        self.delegate = delegate;
    }
    return self;
}

/**
 *  设置按钮背景图片
 */
-(void)setBtnImageView{
    
    //设置背景图片
    self.bgView = [[UIImageView alloc] initWithFrame:self.bounds];
    [self.bgView setImage:[UIImage imageNamed:@"bgViewOff"]];
    [self addSubview:self.bgView];
    
    isOpen = NO;
    
    //设置按钮图片
    self.btnView = [[UIImageView alloc] initWithFrame:CGRectMake(2, 2, CGRectGetWidth(self.frame)/2, CGRectGetHeight(self.frame)-4)];
    [self.btnView setImage:[UIImage imageNamed:@"switchBtn"]];
    [self addSubview:self.btnView];
}
```
##### 触摸方法
```
-(void)touchesBegan:(NSSet<UITouch *> *)touches withEvent:(UIEvent *)event{

    
    //    UITouch * touch = [touches anyObject];
    //    self.beginpoint = [touch locationInView:self].x;
    
    if (!isOpen) {
        [self switchOpen];
        isOpen = YES;
  
    }else{
        
        [self switchClose];
        isOpen = NO;
        
    }
    
    //实现协议方法 传递按钮tag和是否开启
    if (self.delegate  && [self.delegate respondsToSelector:@selector(WJSwitchViewDelegateWithSwitchStates: withTag:)]) {
        [self.delegate WJSwitchViewDelegateWithSwitchStates:isOpen withTag:self.tag];
    }
    
}
```
```
/**
 *  开启
 */
-(void)switchOpen{
    
    dispatch_after(dispatch_time(DISPATCH_TIME_NOW, (int64_t)(0.005 * NSEC_PER_SEC)), dispatch_get_main_queue(), ^{
        [self.bgView setImage:[UIImage imageNamed:@"bgViewOn"]];
    });
    CGFloat X = CGRectGetWidth(self.frame)/2;
    CGRect frame = self.btnView.frame;
     /*创建弹性动画
     damping:阻尼，范围0-1，阻尼越接近于0，弹性效果越明显
     velocity:弹性复位的速度
     */
    [UIView animateWithDuration:0.2 delay:0 usingSpringWithDamping:0.8 initialSpringVelocity:1.0 options:UIViewAnimationOptionCurveLinear animations:^{
        [self.btnView setFrame:CGRectMake(X, frame.origin.y, frame.size.width, frame.size.height)];
    } completion:nil];
}

/**
 *  关闭
 */
-(void)switchClose{
    //替换图片 这里仅做简单的图片替换
    dispatch_after(dispatch_time(DISPATCH_TIME_NOW, (int64_t)(0.005 * NSEC_PER_SEC)), dispatch_get_main_queue(), ^{
        [self.bgView setImage:[UIImage imageNamed:@"bgViewOff"]];
    });
    CGFloat X = 2;
    CGRect frame = self.btnView.frame;
    //修改按钮图片的位置
 [UIView animateWithDuration:0.2 delay:0 usingSpringWithDamping:0.9 initialSpringVelocity:1.0 options:UIViewAnimationOptionCurveLinear animations:^{
         [self.btnView setFrame:CGRectMake(X, frame.origin.y, frame.size.width, frame.size.height)];
    } completion:nil];
}
```
##### viewController.m
```
- (void)viewDidLoad {
    [super viewDidLoad];
    // Do any additional setup after loading the view, typically from a nib.
    
    //创建view
    WJSwitchView * firstView = [[WJSwitchView alloc] initWithFrame:self.firstSwitchView.bounds withTag:1 delegate:self];
    [self.firstSwitchView addSubview:firstView];
    
    WJSwitchView * secondView = [[WJSwitchView alloc] initWithFrame:self.firstSwitchView.bounds withTag:2 delegate:self];
    [self.secondSwitchView addSubview:secondView];
    
    WJSwitchView * thirdView = [[WJSwitchView alloc] initWithFrame:self.firstSwitchView.bounds withTag:3 delegate:self];
    [self.thirdSwitchView addSubview:thirdView];
    
}
#pragma mark - WJSwitchViewDelegate
//协议方法
-(void)WJSwitchViewDelegateWithSwitchStates:(BOOL)isopen withTag:(NSInteger)tag{

    NSLog(@"%ld",tag);
}
```
#### 结语
按钮的位置移动可以使用CAAnimation做一些简单的动画过度,或者直接使用UIView的封装方法实现一些效果来实现,关于动画的一些基本应用,在文集中有转载一篇写的特好的文章讲动画的!!建议自己可以跟着敲一次!!
PS:能用自带插件最好劝说需求用吧,毕竟系统的很强大!  ~~不过我和我们需求谈判失败了...