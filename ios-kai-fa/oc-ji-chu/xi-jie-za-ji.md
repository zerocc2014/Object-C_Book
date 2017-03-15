# 1. 类与对象

## 类方法

OC中类的方法只有实例方法和静态方法\(类方法\)两种：

```
@interface Controller : NSObject

+ (void)thisIsAStaticMethod; // 静态方法

– (void)thisIsAnInstanceMethod; // 实例方法

@end
```





