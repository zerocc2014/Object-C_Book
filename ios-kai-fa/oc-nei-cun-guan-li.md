当前页面被pop出栈时，控制器没有被释放，dealloc方法不走，反复切换页面时，内存激增。到底哪些情况会导致这些问题出现。

## 控制器VC中代理的声明出错

代理的声明使用 assign 或 weak 关键字，如果您用了 retain、strong 强引用声明，有可能导致该问题的出现。

## **控制器VC中Block使用错误**

Block中直接使用成员变量（self.xxx）回造成循环引用，导致拥有该实例的对象不能释放。在ARC下要：

```
__weak Viewcontroller *weakSelf = self;
```

**注：**Block一般用copy声明，这样会把block从栈区移到堆区。这样，在block中进行回调或反向传值到上个页面时，不会出现对象被释放，内存泄露问题。

## **由自定义封装的控件使用错误**

在控制器VC中自定义的控件View的使用中传入了当前VC或self，造成循环引用，这种情况下pop返回时，当前页面也不会被释放，dealloc也不会走。只有在离开页面前，把该控件View先置为空nil。则可以。

## NSTimer中的循环引用

当控制器ViewController跳转进入控制器OneViewController中的时候开启定时器，让定时器每隔一段时间打印一次，当OneViewController dismiss的时候，控制器并没有被销毁。然而定时器的timer invalidate在dealloc中已经写了。

原因在图下：循环引用

![](/assets/NSTimer-retain-syscle.png)

控制器ViewController跳转进入OneViewController中开启定时器

```
#import "OneViewController.h"

@interface OneViewController ()
@property (nonatomic, strong) NSTimer *timer;

@end

@implementation OneViewController
-(void)touchesBegan:(NSSet<UITouch *> *)touches withEvent:(UIEvent *)event{
    [self dismissViewControllerAnimated:YES completion:nil];
}
-(void)viewDidLoad{
    [super viewDidLoad];
    self.view.backgroundColor = [UIColor orangeColor];
    /**
     1.__weak typeof(self) weakSelf = self; 不能解决
     */
    //开启定时器 
    self.timer = [NSTimer scheduledTimerWithTimeInterval:2.0 target:self selector:@selector(testTimerDeallo) userInfo:nil repeats:YES];
}
/** 方法一直执行 */
-(void)testTimerDeallo{
    NSLog(@"-----");
}
```

当开启定时器以后，testTimerDeallo方法一直执行，即使忽略此控制器以后，也是一直在打印，而且dealloc方法不执行。循环引用造成了内存泄露，控制器不会被释放。

```
/** 开启定时器以后控制器不能被销毁,此方法不会被调用 */
-(void)dealloc{
    NSLog(@"xiaohui");
    [self.timer invalidate];
}
@end
```

解决办法：由于循环引用的起因是target，则可以包装一个目标，让target是另一个对象，而不是ViewController即可。

创建一个集成NSObject的分类TimerWeakTarget，创建类方法---开启定时器的方法

```
#import <Foundation/Foundation.h>

@interface TimerWeakTarget : NSObject

@property (nonatomic, assign) SEL selector;
@property (nonatomic, weak) NSTimer *timer;
@property (nonatomic, weak) id target;
/** 
 1.重写开启定时器方法,在内部对target进行替换,换成本类(TimerWeakTarget)的对象即可
 2.不会造成循环引用了,原控制器OneViewController属性有timer对timer强应用,timer内部对self强引用,但是self在此方法内部被替换成了本类的对象(TimerWeakTarget *),而本类的对象不会对OneViewController强引用,则不会造成循环引用,也就不会造成内存泄露
 */
+ (NSTimer *) scheduledTimerWithTimeInterval:(NSTimeInterval)interval
                                      target:(id)aTarget
                                    selector:(SEL)aSelector
                                    userInfo:(id)userInfo
                                     repeats:(BOOL)repeats;

@end
```

TimerWeakTarget.m文件中

```
#import "TimerWeakTarget.h"

@implementation TimerWeakTarget

+ (NSTimer *) scheduledTimerWithTimeInterval:(NSTimeInterval)interval
                                      target:(id)aTarget
                                    selector:(SEL)aSelector
                                    userInfo:(id)userInfo
                                     repeats:(BOOL)repeats{
    TimerWeakTarget * timer = [TimerWeakTarget new];
    timer.target = aTarget;
    timer.selector = aSelector;
    //-------------------------------------------------------------此处的target已经被换掉了不是原来的VIewController而是TimerWeakTarget类的对象timer
    timer.timer = [NSTimer scheduledTimerWithTimeInterval:interval target:timer selector:@selector(fire:) userInfo:userInfo repeats:repeats];
    return timer.timer;
}

-(void)fire:(NSTimer *)timer{

    if (self.target) {
        [self.target performSelector:self.selector withObject:timer.userInfo];
    } else {

        [self.timer invalidate];
    }
}
@end
```

## **僵尸对象：内存已经被回收的对象。**

野指针：指向僵尸对象的指针，向野指针发送消息会导致崩溃。  
野指针错误形式在Xcode中通常表现为：`Thread 1：EXC_BAD_ACCESS`，因为你访问了一块已经不属于你的内存。  
对象已经被释放后，应将其指针置为空指针（没有指向任何对象的指针，给空指针发送消息不会报错）。  
然而在实际开发中实际遇到`EXC_BAD_ACCESS`错误时，往往很难定位到错误点，幸好Xcode提供方便的工具給我们来定位及分析错误。  
1） 在`product－scheme－edit scheme－diagnostics`中将`enable zombie objects`勾选上，下次再出现这样的错误就可以准确定位了。  
2） 在`Xcode－open developer tool－Instruments`打开工具集，选择Zombies工具可以对已安装的应用进行僵尸对象检测。

