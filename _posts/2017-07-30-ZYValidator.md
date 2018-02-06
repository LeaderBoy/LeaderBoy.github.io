---
title: "设计一个输入验证的功能"
layout: post
date: 2017-10-30 16:57
image: /assets/images/markdown.jpg
headerImage: false
tag:
- oc
category: blog
author: LeaderBoy
description: 用于验证 用户名/密码/手机号/邮箱/身份证号码/日期/URL地址/车牌号 的格式
---

#### 使用方法
以验证手机号码为例
```
UITextField *phone = [[UITextField alloc]initWithValidatorType:ZYInputValidatorTypePhone];
// 在需要的地方使用
NSError *error;
BOOL result = [phone validate:&error];
if (error) {
	NSLog(@"%@",[error localizedDescription]);
}
```

---

#### 需求分析

多数时候我们需要验证的有以下几种类型 **用户名**/**密码**/**手机号**/**邮箱**/**身份证号码**/**日期**/**URL地址**/**车牌号**.

---

#### 设计模式
 1. 要对用户输入的内容进行验证 首先我们需要使用的是UITextField,那么我们需要给UITextField设计一个接口.以**分类**的方式给UITextField添加接口无疑是最好的选择.
 2. 由于要验证的类型有很多种,在20多种设计模式中,我选用了类族模式的思想.在iOS中UIButton采用的就是此种设计.
[UIButton buttonWithType:UIButtonTypeSystem];
UIButton作为抽象的父类根据buttonType而让UIButton子类实现不同类型的功能,同时隐藏了子类的实现.
我们只关系需要验证的类型而不关心到底是怎么实现的验证,因此隐藏底层的实现.

#### 实现

#### UITextField
首先给UITextField添加分类实现初始化的功能
在分类中暴漏的接口如下
```
-(instancetype)initWithValidatorType:(ZYInputValidatorType)type;

```
例如**使用方法**中的初始化方式
```
[[UITextField alloc]initWithValidatorType:ZYInputValidatorTypePhone];
```
**注**:ZYInputValidatorType为各种验证类型的枚举




暴漏的第二个接口如下用于让UITextField的实例调用验证
```
-(BOOL)validate:(NSError* _Nullable * _Nullable)error;
```
例如**使用方法**中的验证方式
```
[phone validate:&error]
```
初始化的实现
```
-(instancetype)initWithValidatorType:(ZYInputValidatorType)type {
    if (self = [super init]) {
        ZYInputValidator *validator = [ZYInputValidator validateWithType:type];
        self.validator = validator;
    }
    return self;
}
```
validate接口的实现
```
-(BOOL)validate:(NSError *__autoreleasing  _Nullable *)error {
    NSAssert(self.validator,@"请使用initWithValidatorType或者initWithPasswordType或者initWithPasswordOptions初始化UITextField" );
    return [self.validator validateInput:self error:error];
}
```

---


#### ZYInputValidator
调用的接口设计好了,接下来是接口的实现.上面说过我们需要设计一个抽象的父类命名为ZYInputValidator.ZYInputValidator需要根据UITextField传入的不同的枚举类型实现不同的验证方式.接口的设计可参考UIButton,因此在ZYInputValidator暴漏如下的接口
```
+(instancetype)validateWithType:(ZYInputValidatorType)type;
```
此接口的实现需要使用OC的**多态**的特性,即父类的指针指向子类的对象.
```
+(instancetype)validateWithType:(ZYInputValidatorType)type {
    switch (type) {
        case ZYInputValidatorTypeEmail:
            return [[ZYInputEmailValidator alloc] init];
            break;
        case ZYInputValidatorTypePhone:
            return [[ZYInputPhoneNumberValidator alloc] init];
            break;
        case ZYInputValidatorTypeUserName:
            return [[ZYInputUserNameValidator alloc] init];
            break;
        case ZYInputValidatorTypePassword:
            return [[ZYInputPasswordValidator alloc] init];
            break;
        case ZYInputValidatorTypeIDCard:
            return [[ZYInputIDCardValidator alloc] init];
            break;
        case ZYInputValidatorTypeCarNumber:
            return [[ZYInputCarNumberValidator alloc] init];
            break;
        case ZYInputValidatorTypeDate:
            return [[ZYInputDateValidator alloc] init];
            break;
        case ZYInputValidatorTypePostCode:
            return [[ZYInputPostCodeValidator alloc]init];
            break;
        case ZYInputValidatorTypeURL:
            return [[ZYInputURLValidator alloc]init];
            break;
    }
}
```
此类需要暴漏验证的接口,当初始化返回不同类型的子类,不同的子类在调用同一个接口的时候也就可以实现不同的功能,这也是**多态**的特性.
```
- (BOOL)validateInput:(UITextField *)input error:(NSError* _Nullable * _Nullable)error;

```

---


#### ZYInputEmailValidator
这样让不同的子类去实现不同的验证功能,以验证邮箱为例.

ZYInputEmailValidator继承于ZYInputValidator.实现验证的功能
```
-(BOOL)validateInput:(UITextField *)input error:(NSError **)error {
    
    if (input.text.length <= 0) {
        *error = [NSError errorWithDomain:ZYInputValidatorErrorDomain code:ZYInputValidatorErrorEmail userInfo:@{NSLocalizedDescriptionKey : kEmailValidatorErrorEmpty}];
        return NO;
    } else {
        BOOL isMatch = ZYRX(kEmailValidatorDefault,input.text);
        if (isMatch == NO) {
            *error = [NSError errorWithDomain:ZYInputValidatorErrorDomain code:ZYInputValidatorErrorEmail userInfo:@{NSLocalizedDescriptionKey : kEmailValidatorErrorFormat}];
            return NO;
        }
        return YES;
    }
}
```

ZYRX是我定义的一个宏 实际使用的就是NSPredicate这个类.使用的正则表达式进行验证,具体实现方式可以查看源码.

这样当我们调用UITextField的validate接口的时候实际上就是ZYInputEmailValidator调用了validateInput:(UITextField *)input error:(NSError **)error接口


---

#### 单元测试
一个库的功能想要放心的使用还需要单元测试

 1.比如URL验证的单元测试

![enter image description here](http://oikehvl7k.bkt.clouddn.com/blog_03_002.png)


2.所有的单元测试

![enter image description here](http://oikehvl7k.bkt.clouddn.com/blog_03_001.png)


---

#### 总结 
网络上的大部分实现方式都是给字符串添加分类既没有隐藏实现也没有暴漏统一的接口,并没有果断追求接口整洁,设计模式等,我还是想自己去实现这些功能.让自己更加舒心的调用.
更多的使用方法,更多的功能也可查看我的

[GitBook](https://leaderboy.gitbooks.io/zycomponents/content/chapter1.html)

[源码查看](https://github.com/LeaderBoy/ZYValidator)



