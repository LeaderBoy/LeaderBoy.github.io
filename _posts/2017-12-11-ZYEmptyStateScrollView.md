---
title: "空视图加载状态的切换"
layout: post
date: 2017-12-11 16:57
image: /assets/images/markdown.jpg
headerImage: false
tag:
- oc
category: blog
author: LeaderBoy
description: 让UIScrollView/UITableView/UICollectionView 在加载不同的时期显示不同的视图,比如加载成功,加载中,加载无数据,无网络连接,加载失败几种情况显示不同的视图,避免空白页面体验差,只需三行代码.

---
#### 效果
![enter image description here](http://p13zzx9gf.bkt.clouddn.com/ZYEmptyStateScrollView.gif)

#### 使用方法
```
pod 'ZYEmptyStateScrollView'
```


----------
加载plist数据源[Plist下载地址](http://p13zzx9gf.bkt.clouddn.com/ZYEmptyScrollViewDataSource.plist)

```
[self.tableView loadEmptyDataFromPlist:@"ZYEmptyScrollViewDataSource"];
```
假如当前的状态为加载中
```
self.tableView.emptyDataState = ZYScrollViewStateLoading;
[self.tableView reloadEmptyDataSet];
//reloadData: 一般为success的时候使用 其余情况一般无需使用
[self.tableView reloadData];
```

#### 需求分析
在tableView的加载中我们需要更好的用户体验而不是只提供空白的视图.为此,我们需要在**加载中**/**无网络**/**无数据**/**加载失败**/**加载成功**几种情况来显示不同的视图


#### 实现分析
已经实现的比较好的库为EmptyDataSet,但是在视图的切换并不能满足需求,即使使用其去改变也会造成代码量增大,但是根据对UITableView的理解本质是对数据源的改变,视图的切换需要首先变更数据源,然后reloadData,因此,需要对**加载中**/**无网络**/**无数据**/**加载失败**/**加载成功**提供不同的数据源,因此我在EmptyDataSet的基础上去封装数据源.

#### 实现
首先加载不同状态下的数据源,加载有两种方式一种是使用plist的方式加载,一种是使用数组的方式加载.使用方式中演示了plist加载方式,Plist格式如下

![plist格式](http://ozim7ibb7.bkt.clouddn.com/blog_04_01.png)

得到数据源之后我们需要赋值给EmptyDataSet的协议,首先对协议进行封装.命名为ZYEmptyDataSource.以空视图的时候的标题为例
遵循DZNEmptyDataSetSource协议
```
@interface ZYEmptyDataSource : NSObject<DZNEmptyDataSetSource>
```

在.h中声明数据源
```
@property(nonatomic,strong)NSDictionary *dataSource;

```
在.m中定义emptyTitle
```
@property (nonatomic,copy)NSString  *emptyTitle;

```
```
-(NSString *)emptyTitle {
    return self.dataSource[ZYEmptyDataSourceTitleKey] ?: @"";
}
```
实现DZNEmptyDataSetSource协议的方法

```
- (NSAttributedString *)titleForEmptyDataSet:(UIScrollView *)scrollView {
    NSDictionary *attributesDic = @{NSForegroundColorAttributeName:self.emptyTitleColor,NSFontAttributeName:self.emptyTitleFont};
    NSAttributedString *attributesStr = [[NSAttributedString alloc]initWithString:self.emptyTitle attributes:attributesDic];
    return attributesStr;
}
```
逆向思考,当给ZYEmptyDataSource的dataSource数据源赋值的时候,在调用EmptyDataSet库的
```
[self.tableView reloadEmptyDataSet];
```
此时会更新数据源,刷新界面.

因此,此时需要给dataSource赋值.不同的加载状态赋予不同的值.
给UIScrollView添加一个分类UIScrollView+ZYEmptyState
声明加载状态的枚举
```
typedef NS_ENUM(NSUInteger, ZYScrollViewState) {
    ZYScrollViewStateNetworkUnReachable = 0,
    ZYScrollViewStateLoading,
    ZYScrollViewStateNoData,
    ZYScrollViewStateLoadFailed,
    ZYScrollViewStateLoadSuccess
};
```

给分类添加属性
```
// 视图加载状态
@property(nonatomic,assign)ZYScrollViewState  emptyDataState;
// 数据源
@property(nonatomic,strong)ZYEmptyDataSource *emptyDataSource;

```
使用运行时实现添加属性的Getter和Setter方法
```
-(ZYScrollViewState )emptyDataState {
    return [objc_getAssociatedObject(self, kEmptyDataStateKey) unsignedIntegerValue];
}
```

```
-(void)setEmptyDataState:(ZYScrollViewState)emptyDataState {
    
    if (self.emptyDataState && self.emptyDataState == emptyDataState) return;
    [self.emptyDataSource resetDataSource];
    if (self.emptyDataArray.count > emptyDataState) {
        
        self.emptyDataSource.dataSource = self.emptyDataArray[emptyDataState];
    }
    objc_setAssociatedObject(self, kEmptyDataStateKey, @(emptyDataState), OBJC_ASSOCIATION_ASSIGN);
}
```

emptyDataArray是**给分类添加的私有属性** 实现的方式为
```
@interface UIScrollView(ZYEmptyStatePrivate)
@property(nonatomic,strong)NSArray <NSDictionary *>*emptyDataArray;
@end

@implementation UIScrollView (ZYEmptyStatePrivate)
#pragma mark - Private Property
-(NSArray<NSDictionary *>*)emptyDataArray {
    return objc_getAssociatedObject(self, kEmptyDataArrayKey);
}

-(void)setEmptyDataArray:(NSArray<NSDictionary *>*)emptyDataArray {
    objc_setAssociatedObject(self, kEmptyDataArrayKey, emptyDataArray, OBJC_ASSOCIATION_RETAIN_NONATOMIC);
}

@end
```
也就是再次给UIScrollView添加一个分类

我们在加载plist的时候内部的实现方式实际上就是在给emptyDataArray赋值

```
-(void)loadEmptyDataFromPlist:(NSString *)plistName {
    NSAssert(plistName, @"plistName不能为空");
    [self configueEmptyDataSetSource];
    NSString *path = [[NSBundle mainBundle] pathForResource:plistName ofType:@"plist"];
    NSArray * dataSourceArray = [[NSArray alloc] initWithContentsOfFile:path];
    self.emptyDataArray = dataSourceArray;
}

-(void)configueEmptyDataSetSource {
    self.emptyDataSetSource = self.emptyDataSource = [ZYEmptyDataSource dataSourceInitial];
}
```

整体的流程就是:当我们加载Plist的时候赋值所有加载状态的数据源给emptyDataArray,然后emptyDataArray会根据当前不同的加载状态赋值数据源给ZYEmptyDataSource的dataSource
此时,再调用EmptyDataSet库的以下方法刷新视图
```
[self.tableView reloadEmptyDataSet];
```

更多的使用方式请查看我的

[GitBook](https://leaderboy.gitbooks.io/zycomponents/content/zyemptystatescrollview.html)

[源码查看](https://github.com/LeaderBoy/ZYEmptyStateScrollView)




