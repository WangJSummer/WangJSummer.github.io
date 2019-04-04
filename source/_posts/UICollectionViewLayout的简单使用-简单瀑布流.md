---
title: UICollectionViewLayout的简单使用(简单瀑布流)
author: Jian Wang
date: 2019-04-04 16:41:49
tags: [Objective-C,UI]
categories: iOS
---
对于需要使用到列表的页面,一般是使用UITableView或者是UICollectionView来实现。一直以来都是直接使用UICollectionViewFlowLayout,基本都能实现需求功能，但是对于直接利用UICollectionViewLayout来自定义view的layout没怎么使用过，这里查了蛮多资料自己写了demo，仅供日后参考了。

###UICollectionViewLayoutAttrubutes
一个很重要的类主要记录了cells，supplementaryViews，decorationviews的位置，size，透明度，层级等
- @property (nonatomic) CGRect frame; frame信息
- @property (nonatomic) CGPoint center; 中心点
- @property (nonatomic) CGSize size;大小
- @property (nonatomic) CATransform3D transform3D;
- @property (nonatomic) CGRect bounds NS_AVAILABLE_IOS(7_0);
- @property (nonatomic) CGAffineTransform transform NS_AVAILABLE_IOS(7_0);
- @property (nonatomic) CGFloat alpha;透明度
- @property (nonatomic) NSInteger zIndex; // default is 0  层级
- @property (nonatomic, getter=isHidden) BOOL hidden; // As an optimization, UICollectionView might not create a view for items whose hidden attribute is YES
- @property (nonatomic, strong) NSIndexPath *indexPath;位置
- @property (nonatomic, readonly) UICollectionElementCategory representedElementCategory;
- @property (nonatomic, readonly, nullable) NSString *representedElementKind; 响应类型(区分cell,supple,decaration)
那么当UICollectionView获取布局的时候会通过访问每个位置的部件通过其attribute来询问其布局信息

###自定义一个UICollectionViewLayout
继承自UICollectionViewLayout类，然后一般需要重载下列方法：
- -(void)prepareLayout；
    每次请求布局时候都会自动调用，可以在这里修改一些必要的layout的结构和初始需要的参数等
- -(CGSize)collectionViewContentSize；
 定义UIcollectionView的ContentSize,内容大小
- -(NSArray<UICollectionViewLayoutAttributes *> *)layoutAttributesForElementsInRect:(CGRect)rect；
  返回可见区域rect内的所有元素的布局信息
 返回包含attributes的数组
- -(UICollectionViewLayoutAttributes *)layoutAttributesForItemAtIndexPath:(NSIndexPath *)indexPath；
  定义每个indexpath的item的布局信息
  相应的还有定义supplement以及decoration的方法 这里就不在进行列举
- -(BOOL)shouldInvalidateLayoutForBoundsChange:(CGRect)newBounds；
  当边界发生改变时，是否应该刷新布局。如果YES则在边界变化（一般是scroll到其他地方）时，将重新计算需要的布局信息。
  
   ##demo 
 [demo地址](git@github.com:w467364316/waterfallDemo.git)
![Simulator Screen Shot 2016年10月13日 上午10.52.12.png](http://upload-images.jianshu.io/upload_images/2203462-702a9fe47adaab48.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
使用了一组图片和一个json文件(如果添加过后发现解析为空，在target ->Build Phase - >copy Bundle Resource添加需要的json文件) 正在减肥用的都是减肥励志的哈哈

####PickModel
使用到的model
.h
```
#import <Foundation/Foundation.h>
#import <UIKit/UIKit.h>

@interface PicModel : NSObject
@property(nonatomic,strong) NSString *picPath;//图像
@property(nonatomic,strong) NSString *picDetail;//详细内容
@property(nonatomic,assign) CGFloat rowHeight;//行高

+(instancetype)initWithDic:(NSDictionary*)dic;
-(instancetype)initWithDic:(NSDictionary*)dic;

@end
```
.m 通过detail计算高度，计算所在item的高度，这里图片的高度直接约束限制死了，可以改为按比例计算等。
```
#import "PicModel.h"

@implementation PicModel

+(instancetype)initWithDic:(NSDictionary*)dic{
    
    PicModel *model = [self alloc];
    return [model initWithDic:dic];
}

-(instancetype)initWithDic:(NSDictionary*)dic{

    if (self = [super init]) {
        self.picPath = dic[@"path"];
        self.picDetail = dic[@"detail"];
        
        CGFloat width = ([UIScreen mainScreen].bounds.size.width -(MAXCOUNT - 1) * 10)/MAXCOUNT;
        CGSize size = [dic[@"detail"] boundingRectWithSize:CGSizeMake(([UIScreen mainScreen].bounds.size.width -(MAXCOUNT - 1) * 10)/MAXCOUNT, MAXFLOAT) options:NSStringDrawingTruncatesLastVisibleLine | NSStringDrawingUsesLineFragmentOrigin | NSStringDrawingUsesFontLeading attributes:@{NSFontAttributeName:[UIFont systemFontOfSize:15]} context:nil].size;
        self.rowHeight = size.height + 59 *width / 100 ;
    }
    return self;
}
@end

```
通过修改定义的列数
![Pch文件.png](http://upload-images.jianshu.io/upload_images/2203462-21b4b613e52fd128.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

####ViewController
.m文件
```
#import "BaseViewController.h"
#import "PicModel.h"
#import "PureLayout.h"
#import "PureCollectionViewCell.h"

@interface BaseViewController ()<UICollectionViewDelegate,UICollectionViewDataSource>
@property(nonatomic,strong) UICollectionView *mainCollectionView;
@property(nonatomic,strong) NSMutableArray *dataArray;//数据源数组
@end

@implementation BaseViewController

- (void)viewDidLoad {
    [super viewDidLoad];
    // Do any additional setup after loading the view from its nib.
    
    self.dataArray = [NSMutableArray array];
    //解析geojson文件
    NSString *jsonPath = [[NSBundle mainBundle] pathForResource:@"Detail" ofType:@"geojson"];
    NSArray *detailArr = [[NSJSONSerialization JSONObjectWithData:[NSData dataWithContentsOfFile:jsonPath] options:NSJSONReadingMutableContainers error:nil] objectForKey:@"data"
    ];
    
    //处理model数据
    for (int i =0; i<12; i++) {
        NSString *path = [NSString stringWithFormat:@"%d.jpg",i];
        PicModel *model = [PicModel initWithDic:[NSDictionary dictionaryWithObjectsAndKeys:path,@"path",detailArr[i],@"detail", nil]];
        [self.dataArray addObject:model];
    }
    
    [self definationMainCollectionView];
    
    //添加MJRefreshFooter 添加数据
    __weak typeof(self)weakSelf = self;
    self.mainCollectionView.mj_footer = [MJRefreshAutoNormalFooter footerWithRefreshingBlock:^{
        
        for (int i =0; i<12; i++) {
            NSString *path = [NSString stringWithFormat:@"%d.jpg",i];
            PicModel *model = [PicModel initWithDic:[NSDictionary dictionaryWithObjectsAndKeys:path,@"path",detailArr[i],@"detail", nil]];
            [weakSelf.dataArray addObject:model];
        }
        [weakSelf.mainCollectionView reloadData];
        [weakSelf.mainCollectionView.mj_footer endRefreshing];
    }];
}

/**
 *  设置相关UICollectionView信息
 */
-(void)definationMainCollectionView{
    
    PureLayout *layout = [[PureLayout alloc] init];
    layout.dataArray = self.dataArray;
    
    self.mainCollectionView = [[UICollectionView alloc] initWithFrame:[UIScreen mainScreen].bounds collectionViewLayout:layout];
    [self.mainCollectionView registerNib:[UINib nibWithNibName:@"PureCollectionViewCell" bundle:nil] forCellWithReuseIdentifier:@"pureCell"];
    self.mainCollectionView.delegate = self;
    self.mainCollectionView.dataSource = self;
    self.mainCollectionView.backgroundColor = [UIColor whiteColor];
    [self.view addSubview:self.mainCollectionView];
}
```
```
#pragma mark - UICollectionViewDataSource
-(NSInteger)numberOfSectionsInCollectionView:(UICollectionView *)collectionView{
    
    return 1;
}

-(NSInteger)collectionView:(UICollectionView *)collectionView numberOfItemsInSection:(NSInteger)section{
    
   return  self.dataArray.count;
}

-(UICollectionViewCell*)collectionView:(UICollectionView *)collectionView cellForItemAtIndexPath:(NSIndexPath *)indexPath{
    
    PureCollectionViewCell *cell = [collectionView dequeueReusableCellWithReuseIdentifier:@"pureCell" forIndexPath:indexPath];
    PicModel *model = self.dataArray[indexPath.row];
    cell.model = model;
    return cell;
}
```
####PureLayout
.h文件，这里简单的直接传递了数据源
```
@interface PureLayout : UICollectionViewLayout

@property(nonatomic,strong)NSArray *dataArray;//数据源
@property(nonatomic,assign) NSInteger maxCount;//列数
@end
```
.m文件 定义需要展示的列数，水平 垂直间隔等基本信息
```
#import "PureLayout.h"
#import "PicModel.h"

static CGFloat horizonalSpace = 10;//水平间隔
static CGFloat verticalSpace = 15;//垂直间隔

@interface PureLayout ()
@property(nonatomic,strong) NSMutableArray *offSets;//用于存储不同列的MAXY信息
@end

@implementation PureLayout

-(void)prepareLayout{

    _maxCount = MAXCOUNT;
}
```
根据需要展示的列数，使用数组分别记录每行所在列item的frame.origin.y，进行对比，设置UICollectionView的contentsize
```
-(CGSize)collectionViewContentSize{
    
    CGFloat width = [UIScreen mainScreen].bounds.size.width;
    CGFloat height = 0.0;

   _offSets = [NSMutableArray array];
  
    for (int i =0; i<_maxCount; i++) {
        [_offSets addObject:@0];
    }
    
    for (int i = 0; i<self.dataArray.count; i++) {
        NSInteger col = i % _maxCount;
        PicModel *model = self.dataArray[i];
        
        CGFloat offSetY ;
        offSetY = [_offSets[col] floatValue] + model.rowHeight + verticalSpace;
        _offSets[col] = @(offSetY);
        height = MAX(height, offSetY);
    }
    
    if (height < [UIScreen mainScreen].bounds.size.height) {
        height = [UIScreen mainScreen].bounds.size.height;
    }
    return CGSizeMake(width, height);
}
```
返回可见Rect内的元素的attributes信息
```
-(NSArray<UICollectionViewLayoutAttributes *> *)layoutAttributesForElementsInRect:(CGRect)rect{
    
    NSMutableArray * attributes = [NSMutableArray array];
    for (int i =0 ; i<self.dataArray.count; i++) {
        UICollectionViewLayoutAttributes *attribute = [self layoutAttributesForItemAtIndexPath:[NSIndexPath indexPathForRow:i inSection:0]];
        [attributes addObject:attribute];
    }
    return attributes;
}
```
对不同的indexpath的items设置attributes信息
```
-(UICollectionViewLayoutAttributes *)layoutAttributesForItemAtIndexPath:(NSIndexPath *)indexPath{
    
    UICollectionViewLayoutAttributes *attribute = [UICollectionViewLayoutAttributes layoutAttributesForCellWithIndexPath:indexPath];
    PicModel *model = self.dataArray[indexPath.row];
    CGFloat itemWidth = ([UIScreen mainScreen].bounds.size.width - (MAXCOUNT - 1) * 10)/_maxCount;
    attribute.size = CGSizeMake(itemWidth, model.rowHeight);
    CGFloat itemY = 0.0;
    CGFloat itemX = 0.0;
    
    NSInteger col = indexPath.row % _maxCount;
    itemX = (([UIScreen mainScreen].bounds.size.width - (MAXCOUNT - 1) * 10)/_maxCount + horizonalSpace)* col;
    if (indexPath.row <_maxCount) {
        itemY = 0.0;
    }else{
        UICollectionViewLayoutAttributes *otherAttribute = [self layoutAttributesForItemAtIndexPath:[NSIndexPath indexPathForRow:indexPath.row - _maxCount inSection:0]];
        itemY = CGRectGetMaxY(otherAttribute.frame) + verticalSpace;
    }
    
    attribute.frame = CGRectMake(itemX, itemY, itemWidth,  model.rowHeight);
    return attribute;
}
```
最后添加上该方法，边界改变是重新布局
```
-(BOOL)shouldInvalidateLayoutForBoundsChange:(CGRect)newBounds{
    
    return YES;
}
```
只是一个简单的额瀑布流demo，还有蛮多地方需要优化，这里仅仅写下一些基本思路。

官方给出的两个demo很有学习价值，CircleLayout以及LinLayout，在我之前的给出的参考链接里面都是可以直接下载的,对于与文章中的CircleLayout用法，insert和delete方法已经被appearing和disappearing取代了，参考的githubdemo被我fork了一份，可以进行下载学习 https://github.com/w467364316/CircleLayout.git

参考资料地址：
http://blog.csdn.net/majiakun1/article/details/17204921
 http://www.cnblogs.com/wangyingblock/p/5627448.html  