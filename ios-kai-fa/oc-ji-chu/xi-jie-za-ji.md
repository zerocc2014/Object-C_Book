# 1. 类与对象

## 类方法

OC中类的方法只有实例方法和静态方法\(类方法\)两种：

```
@interface Controller : NSObject

+ (void)thisIsAStaticMethod; // 静态方法

– (void)thisIsAnInstanceMethod; // 实例方法

@end
```

OC 没有像 Java，C++ 中的那种绝对的私有及保护成员方法，仅仅可以对调用者隐藏某些方法（声明和实现都写在 @implementation 里）。

可以使用 Category 来实现私有方法，给myClass类添加一个分类\(Category\)，在分类.h中写上该私有方法声明，分类.m中不用写实现。这就是**私有方法的前向引用**：

```
// myClass.h文件
@interface myClass : NSObject

- (void)PublicMethod; //公开方法，可在其他类或子类进行访问

@end


// myClass.m文件
@interface myClass ()   //类延展
- (void)PrivateMethod;  //在类延展中定义的是私有方法

@end

@implementation myClass

- (void)PublicMethod { //.h中有申明，公开方法
    NSLog(@"这是公开方法，其他类可以调用");

}

- (void)PrivateMethod { //类延展中有申明，私有方法
    NSLog(@"这是私有方法，其他类无法调用");

}

- (void)noInterfacePrivateMethod { //不在任何地方申明，私有方法
    NSLog(@"这是私有方法，其他类无法调用");
}

@end
```

其分类private：

```
#import "myClass.h"

// myClass的分类 private
@interface myClass (private)

- (void)PrivateMethod;  //在分类中前向引用

@end


#import "myClass+private.h"

@implementation myClass (private)

@end
```

关于 Category 和 Extension 的一些区别，在[这里](https://hit-alibaba.github.io/interview/iOS/ObjC-Basic/Class.html#extension)。

































