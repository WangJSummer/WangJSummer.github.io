---
title: UITextView-placeholder的实现和解析
author: Jian Wang
tags: [Objective-C,UI]
categories: iOS
date: 2019-04-04 16:18:40
---
#### 前言
##### 概要
项目中UITextfield使用的比较频繁，对于placeholder可以直接设置，文字，颜色，字体等等，但是UITextView继承自UIScrollView,并没有placeholder属性。项目中以前就有使用到UITextView的placeholder,当时只是外加了一个UILabel,但是每次都需要重新定制label，所以想着能够写一个类别，添加一个label，实现字体，颜色，位置的可调节。

这里主要是在GitHub上面发现一个很好的类别，该类别的编写者也是我所喜欢的一位iOS开发者。自己尝试写了一次，有了一定的了解，所以记录下来，方便以后查阅。
Demo地址：[码云demo下载地址](http://git.oschina.net/W-Jian/UITextView_placeholder)
##### 效果图
首先先上图看看效果：
默认效果图：
![defalut_placeholder.png](http://upload-images.jianshu.io/upload_images/2203462-7760a2edcc736f86.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
设置颜色和location图：
![color_location_placeholder.png](http://upload-images.jianshu.io/upload_images/2203462-e23a9c34f5d03ec3.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

![placeholder.gif](http://upload-images.jianshu.io/upload_images/2203462-9668d7416ecf48b4.gif?imageMogr2/auto-orient/strip)
下面开始解析代码
根据意图，也是在UITextView中添加一个UILabel,然后添加文字，颜色，富文本，位置等属性，实现placeholder的可定制。

#### 代码解析
##### .h 解析

* (readOnly)UILabel *placeholdLabel
* NSString *placeholder placeholder                   文字
* UIColor *placeholderColor                                  颜色
* NSAttributedString *attributePlaceholder        富文本
* CGPoint location                                                   位置

```
#import <UIKit/UIKit.h>

@interface UITextView (WJPlaceholder)

@property(nonatomic,readonly)  UILabel *placeholdLabel;
@property(nonatomic,strong) IBInspectable NSString *placeholder;
@property(nonatomic,strong) IBInspectable UIColor *placeholderColor;
@property(nonnull,strong) NSAttributedString *attributePlaceholder;
@property(nonatomic,assign) CGPoint location;
+ (UIColor *)defaultColor; //获取默认颜色（选取UITextFiled的placeholder颜色）
@end
```

使用IBInspectable修饰，可以使该属性在interface builder中进行编辑。placeholder和placeholder color都可以在user defined runtime attributes中进行配置，如下图：

![UITextViewKeyPath.png](http://upload-images.jianshu.io/upload_images/2203462-d5be40ffdc4df595.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)


![kvc_placeholder.png](http://upload-images.jianshu.io/upload_images/2203462-ca38c48721b234ae.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

##### .m 解析
首先设置三个key值，用于关联属性的创建和获取
```
static char *labelKey = "placeholderKey";
static char *needAdjust = "needAdjust";
static char *changeLocation = "location";
```
创建的类别是不支持添加属性的，要实现属性的可设置，可以使用runtime来动态设置和获取属性值。
runtime添加关联属性
```
void objc_setAssociatedObject(id object, const void *key, id value, objc_AssociationPolicy policy)
```
第一个参数(object)表示要添加属性的类，第二参数(const void *key)为一个key值，用来获取设置的属性值，第三个参数(value)为属性的值，第四个参数(objc_AssociationPolicy policy)为属性的修饰类型：一共有5个可选项：
```
typedef OBJC_ENUM(uintptr_t, objc_AssociationPolicy) {
    OBJC_ASSOCIATION_ASSIGN = 0,           /**< Specifies a weak reference to the associated object. */
    OBJC_ASSOCIATION_RETAIN_NONATOMIC = 1, /**< Specifies a strong reference to the associated object. 
                                            *   The association is not made atomically. */
    OBJC_ASSOCIATION_COPY_NONATOMIC = 3,   /**< Specifies that the associated object is copied. 
                                            *   The association is not made atomically. */
    OBJC_ASSOCIATION_RETAIN = 01401,       /**< Specifies a strong reference to the associated object.
                                            *   The association is made atomically. */
    OBJC_ASSOCIATION_COPY = 01403          /**< Specifies that the associated object is copied.
                                            *   The association is made atomically. */
};
```
分别对应assign,retain_nonatomic,copy_nonatomic,retain,copy。

runtime获取成员属性方法为：
```
id objc_getAssociatedObject(id object, const void *key)
```
第一个参数（object）为属性所属的类，一般使用self，第二个参数（key）为key值，和设置关联属性时使用的key一样。

placeholderLabel为readOnly，重写其get方法。
```
/**
 *  设置placeholderLabel
 *
 *  @return label
 */
- (UILabel *)placeholdLabel
{
    UILabel *label = objc_getAssociatedObject(self, labelKey);
    if (!label) {
        label = [[UILabel alloc] init];
        label.textAlignment = NSTextAlignmentLeft;
        label.numberOfLines = 0;
        label.textColor = [self.class defaultColor];
        objc_setAssociatedObject(self, labelKey, label, OBJC_ASSOCIATION_RETAIN_NONATOMIC);
        //添加通知
        [[NSNotificationCenter defaultCenter] addObserver:self selector:@selector(updateLabel) name:UITextViewTextDidChangeNotification object:nil];
        //监听font的变化
        [self addObserver:self forKeyPath:@"font" options:NSKeyValueObservingOptionNew context:nil];
    }
    return label;
}

/**
 *  设置默认颜色
 *
 *  @return color
 */
+ (UIColor *)defaultColor
{
    static UIColor *color = nil;
    static dispatch_once_t once_t;
    dispatch_once(&once_t, ^{
       UITextField *textField = [[UITextField alloc] init];
       textField.placeholder = @" ";
       color = [textField valueForKeyPath:@"_placeholderLabel.textColor"];
    });
    return color;
}
```
创建UILabel时，设置基本属性，这里采用UITextfiled的placeholder的颜色作为默认颜色。
同时添加对UITextView的状态通知，UITextView主要有以下几个通知可供使用
```
UIKIT_EXTERN NSNotificationName const UITextViewTextDidBeginEditingNotification;
UIKIT_EXTERN NSNotificationName const UITextViewTextDidChangeNotification;
UIKIT_EXTERN NSNotificationName const UITextViewTextDidEndEditingNotification;
```
这里使用UITextViewTextDidChangeNotification，在text改变的时候执行updateLabel方法，进行label的显示和隐藏以及其他设置。
对于UITextView的字体，使用KVO监听UITextView的font属性，在设置时改变placeholderLabel的font。

以下是一些成员属性的set get方法，通过set方法，将text,color,attributedText赋给placeholderLabel。
该类别主要使用runtime创建了3个关联属性，分别为
* placeholderLabel  
* location             CGPoint 位置
* needAdJust       BOOL 判断是否需要调整

```
- (void)setPlaceholder:(NSString *)placeholder
{
    self.placeholdLabel.text = placeholder;
    [self updateLabel];
}

- (NSString *)placeholder
{
    return self.placeholdLabel.text;
}

- (void)setPlaceholderColor:(UIColor *)placeholderColor
{
    self.placeholdLabel.textColor = placeholderColor;
    [self updateLabel];
}

- (UIColor *)placeholderColor
{
    return self.placeholdLabel.textColor;
}

- (void)setAttributePlaceholder:(NSAttributedString *)attributePlaceholder
{
    self.placeholdLabel.attributedText = attributePlaceholder;
    [self updateLabel];
}

- (NSAttributedString *)attributePlaceholder
{
    return self.placeholdLabel.attributedText;
}

- (void)setLocation:(CGPoint)location
{
    objc_setAssociatedObject(self, changeLocation,NSStringFromCGPoint(location), OBJC_ASSOCIATION_RETAIN_NONATOMIC);
    [self updateLabel];
}

-(CGPoint)location
{
    return CGPointFromString(objc_getAssociatedObject(self, changeLocation));
}

//是否需要调整字体
- (BOOL)needAdjustFont
{
    return [objc_getAssociatedObject(self, needAdjust) boolValue ];
}

- (void)setNeedAdjustFont:(BOOL)needAdjustFont
{
    objc_setAssociatedObject(self, needAdjust, @(needAdjustFont), OBJC_ASSOCIATION_ASSIGN);
}
```

KVO 监听方法，判断是否是font，执行updataLabel方法。
```
- (void)observeValueForKeyPath:(NSString *)keyPath ofObject:(id)object change:(NSDictionary<NSKeyValueChangeKey,id> *)change context:(void *)context
{
    if ([keyPath isEqualToString:@"font"])
    {
        self.needAdjustFont = YES;
         [self updateLabel];
    }
}
```
下面则是更新placeholderLabel的主要方法，调整位置，字体等。
```
/**
 *  更新label信息
 */
- (void)updateLabel
{
    if (self.text.length) {
        [self.placeholdLabel removeFromSuperview];
        return;
    }
    //显示label
    [self insertSubview:self.placeholdLabel atIndex:0];
    
    //是否需要更新字体（NO 采用默认字体大小）
    if (self.needAdjustFont) {
        self.placeholdLabel.font = self.font;
        self.needAdjustFont = NO;
    }

   CGFloat  lineFragmentPadding =  self.textContainer.lineFragmentPadding;  //边距
   UIEdgeInsets contentInset = self.textContainerInset;

    //设置label frame
    CGFloat labelX = lineFragmentPadding + contentInset.left;
    CGFloat labelY = contentInset.top;
    if (self.location.x != 0 || self.location.y != 0) {
        if (self.location.x < 0 || self.location.x > CGRectGetWidth(self.bounds) - lineFragmentPadding - contentInset.right || self.location.y< 0) {
            // 不做处理
        }else{
            labelX += self.location.x;
            labelY += self.location.y;
        }
    }
    CGFloat labelW = CGRectGetWidth(self.bounds)  - contentInset.right - labelX;
    CGFloat labelH = [self.placeholdLabel sizeThatFits:CGSizeMake(labelW, MAXFLOAT)].height;
    self.placeholdLabel.frame = CGRectMake(labelX, labelY, labelW, labelH);
}
```
 对于placeholderLabel的frame设置，需要考虑几个方面：
* lineFragmentPadding  UITextView textContainer边距
* contentInset  输入区域偏移 UIEdgeInsets
* location    设置的位置
* labeltextHeight  placeholder的text高度

> placeholder的frame.orgin.x = 边距lineFragmentPadding + contentInset.left + location.x;
frame.orgin.y = contentInset.top + location.y;
frame.size.width = UITextView的宽度 - placeholder的frame.orgin.x - contentInset.right;
frame.height = placeholder.text的高度

#### 结语
以上就是关于整个UITextView_placeholder的学习解剖，学到了一些新的知识，如果大家有什么意见和建议可以给我留言，指出我的不对的地方，大家一起学习交流。