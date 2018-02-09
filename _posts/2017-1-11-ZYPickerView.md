---
title: "各种类型的PickerView选择器"
layout: post
date: 2018-1-11 16:57
image: /assets/images/markdown.jpg
headerImage: false
tag:
- UIPickerView
category: blog
author: LeaderBoy
description: 单列选择器/多列选择器/多列关联选择器/地址选择器/日期选择器.

---
#### 效果示例

![enter image description here](http://p13zzx9gf.bkt.clouddn.com/ZYPickerView03.png)

#### 使用方式
```
pod 'ZYPickerView'
```
调用
```
let associatedData: [[AssociatedData]] = [
    // 第一列数据 (key)
    [   AssociatedData(key: "swift"),
        AssociatedData(key: "objectivec"),
        AssociatedData(key: "html"),
        AssociatedData(key: "java")
    ],
    // 第二列数据 (valueArray)
    [    AssociatedData(key: "swift", valueArray: ["xcode"]),
         AssociatedData(key: "objectivec", valueArray: ["xcode"]),
         AssociatedData(key: "html", valueArray: ["vs", "monokai-sublime","foundation","dark","atelier-dune-dark","googlecode","color-brewer","atelier-dune-light"]),
         
         AssociatedData(key: "java", valueArray: ["androidstudio", "vs","pojoaque","googlecode"])
    ]
]

StringPickerView.show(dataSource: .associatedRowData(associatedData, defaultIndexs: nil)) { (indexPath) in
    print(indexPath)
}
```


**注**:本功能使用Swift开发,不适用于OC

#### 需求分析
选择器的使用大致分为单列选择器/多列选择器/日期选择器/地址选择器.而多列选择器一般是两列或者三列.

#### 实现
**初步分析:**

 1. 首先我们需要在原有Controller的上面加上一层遮罩
 2. 自己实现"取消"/"完成"按钮
 3. 然后再实现UIPickerView.
 4. 然后使用UIView的animationWithDuration方式移动y坐标实现show和hide方法.

首先在很多的ProgressHUD的源码都会找到在Controller上面添加灰色遮罩的方式,比如在SVProgressHUD源码中的获取最靠前的window
```
- (UIWindow *)frontWindow {
#if !defined(SV_APP_EXTENSIONS)
    NSEnumerator *frontToBackWindows = [UIApplication.sharedApplication.windows reverseObjectEnumerator];
    for (UIWindow *window in frontToBackWindows) {
        BOOL windowOnMainScreen = window.screen == UIScreen.mainScreen;
        BOOL windowIsVisible = !window.hidden && window.alpha > 0;
        BOOL windowLevelSupported = (window.windowLevel >= UIWindowLevelNormal && window.windowLevel <= self.maxSupportedWindowLevel);
        BOOL windowKeyWindow = window.isKeyWindow;
			
        if(windowOnMainScreen && windowIsVisible && windowLevelSupported && windowKeyWindow) {
            return window;
        }
    }
#endif
    return nil;
}
```
或者自己可以使用UIWindow添加到当前视图上面.而实现实现"取消"/"完成"按钮,然后再实现UIPickerView.则不必多说.

以上的实现方式存在很多问题,

 1. 去实现完成/取消按钮/PickerView然后autolayout布局,然后使用动画实现show和hide,实现起来繁琐.
 2. 如果引入UIWindow,那么只要你使用过多Window的情况(尤其是在加广告的App中),你自己就会明白多么不好调整视图.
 

**进一步分析:**

我们在使用键盘的时候经常会使用到UIToolBar,让键盘的inputAccessoryView = UIToolBar,既可以添加到键盘上面了.,那么我们的取消和完成按钮的那一栏就可以使用UIToolBar去实现.我们可以有如下的代码
```
fileprivate lazy var toolBar : UIToolbar = {
    let tool = UIToolbar(frame: CGRect(x: 0, y:0, width: ZYPickerView.screenWidth, height: ZYPickerView.toolBarH))
    let leftBarButtonItem = UIBarButtonItem(title: "取消", style: .plain, target: self, action: #selector(leftButtonClicked))
    let space = UIBarButtonItem(barButtonSystemItem: .flexibleSpace, target: nil, action: nil)
    let rightBarButtonItem = UIBarButtonItem(title: "完成", style: .plain, target: self, action: #selector(rightButtonClicked))
    tool.tintColor = UIColor.black
    tool.items = [leftBarButtonItem,space,rightBarButtonItem]
    return tool
}()
```
细心的你可能发现,除了取消/完成按钮我还添加了如下代码
```
    let space = UIBarButtonItem(barButtonSystemItem: .flexibleSpace, target: nil, action: nil)
```
由于toolBar上面的item的排列是从左到右,也就是说如果你不这么做的话,取消/完成按钮是紧挨着左对齐的排列,此处相当于是一个小技巧.

我们此时要做的就是把键盘换成UIPickerView,那么系统已经提供给了我们实现的方式,那就是重写视图的inputView

```
override public var inputView: UIView? {
    get {
        return self.picker
    }
}
```
同时还要实现
```
override public var canBecomeFirstResponder: Bool {
    return true
}
```
我们还需要知道另外一点,那就是只有UITextField和UITextView的inputAccessoryView是可赋值的,也就是说其他视图的inputAccessoryView是只读的.因此我们还需要重写视图的inputAccessoryView方法

```
override public var inputAccessoryView: UIView! {
    get {
        return self.toolBar
    }
}
```
**注**: "视图"一词指的是ZYPickerView

重写了这几个方法之后当ZYPickerView调用becomeFirstResponder方法之后,UIPickerView就会像键盘一样弹出.

**好处**:对比**初步分析**的方法很明显,重写系统提供的方法不需要进行视图的布局,不需要获取window实现遮罩,不需要实现视图弹出和隐藏的动画.一切从简.


----------


接下来是实现UIPickerView,以StringPickerView.show方法为例,首先UIPickerView的协议方法中有row和component,然而在类似的UITableView中有row和session,但是他们都被封装到了NSIndexPath中,因此我也将UIPickerView的row和component也封装为IndexPath如下,value则是对应位置的值.
```
public struct PickerIndexPath {
    var component : Int
    var row : Int
    var value : String
}
```
当点击完成按钮的时候我会返回对应的位置和对应的值那么我们定义点击的回调
```
public typealias DoneAction = ([PickerIndexPath]) -> Void

var doneAction : DoneAction?
// 当前选中的indexPath
var selectedValue : [PickerIndexPath]!
```
show方法的参数类型有三种单列/多列/多列关联,做如下的封装

```
public typealias AssociatedRowDataType = [[AssociatedData]]

public enum PickerDataSourceType {
    case singleRowData(_ : [String],defaultIndex: Int?)
    case multiRowData(_ : [[String]],defaultIndexs: [Int]?)
    case associatedRowData(_ : AssociatedRowDataType,defaultIndexs: [Int]?)
}
```
关联的数据类型
```
public struct AssociatedData {
    var key: String
    var valueArray: [String]?
    init (key: String, valueArray: [String]? = nil) {
        self.key = key
        self.valueArray = valueArray
    }
}
```
此处用到了枚举参数的特性.

以最简单的单列为例
```
func show(singleRowData:[String],default index : Int? = nil,doneAction : @escaping DoneAction) {
    assert(!singleRowData.isEmpty, "数组为空")
    let hasDefaultIndex : Bool = index != nil
    if hasDefaultIndex {
        assert(singleRowData.count > index!,"默认的index超出了dataSource数组的总个数")
    }
    self.doneAction = doneAction
    self.singleRowData = singleRowData
    self.selectedValue = hasDefaultIndex ?
        [PickerIndexPath(component: 0,row: index!,value: singleRowData[index!])]
        :
        [PickerIndexPath(component: 0, row: 0, value: singleRowData[0])]
    self.picker.selectRow(index ?? 0, inComponent: 0, animated: true)
}
```
实现协议方法
```
public func numberOfComponents(in pickerView: UIPickerView) -> Int {
    return 1
}

public func pickerView(_ pickerView: UIPickerView, numberOfRowsInComponent component: Int) -> Int {
    return singleRowData.count
}

public func pickerView(_ pickerView: UIPickerView, titleForRow row: Int, forComponent component: Int) -> String? {
    if singleRowData.count > row {
       return singleRowData[row]
    }else{
       return nil
    }
}

public func pickerView(_ pickerView: UIPickerView, didSelectRow row: Int, inComponent component: Int) {
    selectedValue[component].component = component
    selectedValue[component].row = row
    selectedValue[component].value = singleRowData[row]
}

```
更多的使用方式和演示请查看我的

[GitBook](https://leaderboy.gitbooks.io/zycomponents/zypickerviewswift.html)

[源码查看](https://github.com/LeaderBoy/ZYPickerView)






