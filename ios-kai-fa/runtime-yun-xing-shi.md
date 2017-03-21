[http://www.cocoachina.com/ios/20151026/13908.html](http://www.cocoachina.com/ios/20151026/13908.html)

[http://blog.csdn.net/sike2008/article/details/50164961](http://blog.csdn.net/sike2008/article/details/50164961 "runtime 实现万能跳转")

ding [https://halfrost.com/objc\_runtime\_isa\_class/](https://halfrost.com/objc_runtime_isa_class/)

1. 如何动态调用

runtime简称运行时。也就是系统运行的时候的一些机制，主要是消息机制。函数的调用在编译的时候就已经决定是具体哪个函数，编译完再按顺序执行。OC语言之所以称之为动态运行时语言，主要表现在对象类型在运行时才真正确定，并不是在编译阶段，Objective-C是基于C语言加入了面向对象特性和消息转发机制的动态语言，这意味着它不仅需要一个编译器，还需要Runtime系统来动态创建类和对象，进行消息发送和转发。。OC的函数调用称之为消息发送，是一个动态调用过程，编译的时候并不能决定真正调用哪个函数（只是一个方法名，在编译阶段可以调用任何函数，即使是这个函数并未实现，只要是声明过的就不会报错。而c语言在编译阶段就会报错），只要在真正运行的时候才会根据函数名找到对应函数来调用。

## OC 中类与对象

Objective-C类是由Class类型来表示的，它实际上是一个指向objc\_class结构体的指针，它的定义如下：

```
typedef struct objc_class *Class;   // An opaque type that represents an Objective-C class. 不透明类型代表类
```

查看 &lt;objc/runtime.h&gt; 中 objc\_class 结构体的定义如下：

```
struct objc_class {
    Class isa  OBJC_ISA_AVAILABILITY;     //指向对象类型 是指向元类的指针

#if !__OBJC2__
    Class super_class                       OBJC2_UNAVAILABLE;  // 父类 是指向父类的
    const char *name                        OBJC2_UNAVAILABLE;  // 类名
    long version                            OBJC2_UNAVAILABLE;  // 类的版本信息，默认为0
    long info                               OBJC2_UNAVAILABLE;  // 类信息，供运行期使用的一些位标识
    long instance_size                      OBJC2_UNAVAILABLE;  // 该类的实例变量大小
    struct objc_ivar_list *ivars            OBJC2_UNAVAILABLE;  // 该类的成员变量链表
    struct objc_method_list **methodLists   OBJC2_UNAVAILABLE;  // 方法定义的链表
    struct objc_cache *cache                OBJC2_UNAVAILABLE;  // 方法缓存
    struct objc_protocol_list *protocols    OBJC2_UNAVAILABLE;  // 协议链表
#endif

} OBJC2_UNAVAILABLE;
```

* methodList：这是方法的定义列表，是指针的指针，所以可以通过修改该指针指向的指针的地址，来动态增加方法，这也是Category的实现原理。同理存储对象成员变量的指针是\*vars，所以无法动态增加成员变量。所以Category只能添加方法，却不可添加属性的原因就在于此了。那苹果什么要这么设计呢？简单说Category设计的目的就是用来扩展类功能，而非封装数据，所有的属性和成员变量应该放在主接口（main interface），才能使得类的设计更加清晰。

* objc\_cache：在讲解Effective Objective-C Notes：理解消息传递机制，会发现Objective-C要调用一个方法似乎要经过很多步骤。为了优化方法调用速度，就将调用过的一些方法缓存到到objc\_cache中，后续如果再次调用时，会先从缓存列表里面查找，如此速度便可加快。

此结构体存放类的元数据（metadata），例如类的实例实现了几个方法，具备了多少个实例变量等信息。结构体的首个变量也是isa指针，说明Class本身也是Objective-C对象。对象所属的类型（即isa指针所指向的对象类型）是另一个类，叫做元（metaclass），用来表述对象本身所具备的元数据。类法就定义在此处，因为这些方法可以理解成类对象的实例方法。每个类仅有一个类对象，每个类对象也仅有一个与之相关的元类。

![](/assets/Class.png)

NSObject 基类定义 :

```
typedef struct objc_class *Class;

@interface NSObject <NSObject> {
    Class isa  OBJC_ISA_AVAILABILITY;
}
```

objc\_class 2.0 之后的定义：

```
typedef struct objc_class *Class;  
typedef struct objc_object *id;

@interface Object { 
    Class isa; 
}

@interface NSObject <NSObject> {
    Class isa  OBJC_ISA_AVAILABILITY;
}

struct objc_object {  
private:  
    isa_t isa;
}

struct objc_class : objc_object {  
    // Class ISA;
    Class superclass;
    cache_t cache;             // formerly cache pointer and vtable
    class_data_bits_t bits;    // class_rw_t * plus custom rr/alloc flags
}

union isa_t  
{
    isa_t() { }
    isa_t(uintptr_t value) : bits(value) { }
    Class cls;
    uintptr_t bits;
}
```

代表一个类的实例，对象的定义如下：

```
// Represents an instance of a class.
struct objc_object {
    Class isa  OBJC_ISA_AVAILABILITY;
};
```

一个类实例的指针，通用对象类型： id

```
// A pointer to an instance of a class.
typedef struct objc_object *id;
```

##### 备注

in tvar是定义一个int型数var；

int \*p是一个指针变量也就是说指针所指向p地址等价于取值操作

int &var变量var的地址取地址操作p = &var

typedef可以给指针、结构体起别名，当然也可以给指向结构体的指针起别名

```
#include <stdio.h>
 // 定义一个结构体并起别名
typedef struct {
    float x;     
    float y;
} Point; 
 // 起别名
  typedef Point *PP;

 int main(int argc, const char * argv[]) {
       // 定义结构体变量
       Point point = {10, 20};   
       // 定义指针变量
       PP p = &point;

  // 利用指针变量访问结构体成员
   printf("x=%f，y=%f", p->x, p->y);
      return 0;
}
```

## OC方法动态调用

OC只是在编译阶段确定了要向接收者发送message这条消息，而receive将要如何响应这条消息，那就要看运行时发生的情况来决定了。通过发送消息来达到动态调用：

> \[obj makeText\];   编译时runtime会将其转化为：objc\_msgSend\(obj,@selector\(makeText\)\);

也就是说 objc\_msgSend 函数相当于入口；对象调用某个方法都将被编译器转化为：

```
id objc_msgSend(id self, SEL op, ... );  // objc_msgSend(obj, selector, arg1, arg2, ...)
```

如果消息的接收者能够找到对应的selector，那么就相当于直接执行了接收者这个对象的特定方法；否则，消息要么被转发，或是临时向接收者动态添加这个 selector 对应的实现内容，要么就干脆玩完崩溃掉。

#### `id objc_msgSend (id self,SEL op, ... )  //对应的数据结构：`

SEL：方法编号, objc\_msgSend函数的第二个参数，它是selector在Objc中的表示类型（Swift中是Selector类）。selector是方法选择器，可以理解为区分方法的ID，而这个ID的数据结构是SEL:

```
             typedef struct objc_selector *SEL;
```

其实它就是个映射到方法的C字符串，你可以用Objc编译器命令@selector\(\)或者Runtime系统的sel\_registerName函数来获得一个          SEL类型的方法选择器。

不同类中相同名字的方法所对应的方法选择器是相同的，即使方法名字相同而变量类型不同也会导致它们具有相同的方法选择器，于是Objc中方法命名有时会带上参数类型\(NSNumber一堆抽象工厂方法拿走不谢\)。

## OC 消息发送流程

objc\_msgSend\(receiver, selector, arg1, arg2,...\) 这个函数完成了动态绑定的所有事情：

* 检测这个selector是不是要忽略。比如Mac OS X开发，有了垃圾回收就不理会retain, release这些函数了。

* 检测这个target是不是nil对象。ObjC的特性是允许对一个nil对象执行任何一个方法不会Crash，因为会被忽略掉。

* 上面检测都通过则开始查找这个类的IMP,先从cache里面找,完了找得到就跳到对应的函数去执行。

* 如果cache找不到，通过对象的isa指针获取到类的结构体，然后在方法分发表里面查找方法的selector \(方法分发表既是：class中的方法列表method\_list，它将方法选择器和方法实现联系起来\)。

* 如果分发表找不到，objc\_msgSend 结构体中指向父类的指针找到其父类，并在父类的分发表去找方法的selector，会一直沿着类的继承体系到达NSObject类。一旦定位到selector，函数会就获取到了实现的入口点，并传入相应的参数来执行方法的具体实现,并将该方法添加进入缓存中。如果最后也没有定位到selector，则会走消息转发流程。

### 消息转发

调用方法的方式有两种：

1. \[object message\] 的方式调用方法，如果一个对象无法按上述正常流程接受某一消息时，就会启动所谓“消息转发\(message forwarding\)”机制，通过这一机制，我们可以告诉对象如何处理未知的消息。默认情况下，对象接收到未知的消息，会导致程序崩溃，通过控制台，我们可以看到以下异常信息：这段异常信息实际上是由NSObject的“doesNotRecognizeSelector”方法抛出的。不过，我们可以采取一些措施，让我们的程序执行特定的逻辑，而避免程序的崩溃。

2. 以perform…的形式来调用，则需要等到运行时才能确定object是否能接收message消息。如果不能，则程序崩溃。通常，当我们不能确定一个对象是否能接收某个消息时，会先调用respondsToSelector:来判断一下。如下代码所示：

```
    if([self respondsToSelector:@selector(method)]){
        [self performSelector:@selector(method)];
    }
```

此处讨论第一种方式的方法调用情况下的消息转发机制，在异常抛出前，Objective-C的运行时会给你三次拯救程序的机会：

* 动态方法解析
* 备用接受者
* 完整转发流程

##### 动态方法解析

* resolveInstanceMethod:解析实例方法 
* resolveClassMethod:解析类方法
* 通过class\_addMethod的方式将缺少的selector动态创建出来，前提是有提前实现好的IMP（method\_types一致\)
* 这种方案更多的是为@dynamic属性准备的

对象在接收到未知的消息时，首先会调用所属类的类方法 +resolveInstanceMethod:\(实例方法\)或者 +resolveClassMethod:\(类方法\)。在这个方法中，我们有机会为该未知消息新增一个“处理方法”，通过运行时class\_addMethod函数动态添加到类里面就可以了。

```
@interface SomeClass : NSObject
- (void)foo;
- (void)crash;
@end

@implementation SomeClass

-(void)foo {
   NSLog(@"method foo was called on %@", [self class]);
}
@end
```

分别调用这两个方法：

```
SomeClass *someClass = [[SomeClass alloc] init];
[someClass foo];
[someClass crash];
```

foo 方法正常打印，crash 方法崩溃；运用动态方法解析，Objective-C运行时会调用+ resolveInstanceMethod：或者+ resolveClassMethod :,让你有机会提供一个函数实现。如果你添加了函数并返回YES，那运行时系统就会重新启动一次消息发送的过程。还是以crash为例，你可以这么实现：

```
@implementation SomeClass
- (void)foo{
    NSLog(@"method foo was called on %@",[self class]);
}

void crashMethod(id obj, SEL _cmd) {
    NSLog(@"crash Method");
}

+ (BOOL)resolveInstanceMethod:(SEL)sel {
    if(sel == @selector(crash)){
        class_addMethod([self class], sel, (IMP)crashMethod, "v@:");
        return YES;
    }
    return [super resolveInstanceMethod:sel];
}
@end
```

`Core Data`有效到这个方法，NSManagedObjects中的属性的getter和setter就是在运行时动态添加的。

如果resolveInstanceMethod：方法返回NO，运行时就会进行下一步：消息转发（Message Forwarding）。

##### 备用接受者

* 如果上一步没有处理，runtime会调用以下方法

* * -\(id\)forwardingTargetForSelector:\(SEL\)aSelector
* 如果该方法返回非nil的对象，则使用该对象作为新的消息接收者
  * 不能返回self，会出现无限循环
  * 如果不知道该返回什么，应该使用\[super forwardingTargetForSelector:aSelector\]
* 这种方法属于单纯的转发，无法对消息的参数和返回值进行处理

这一步合适于我们只想将消息转发到另一个能处理该消息的对象上。但这一步无法对消息进行处理，如操作消息的参数和返回值。

```
@interface SomeClass : NSObject
- (void)foo;
- (void)crash;
/**
 *  把字符串转换为数组
 *
 *  @param str 需转换的字符串
 *
 *  @return 转换好的数组
 */
- (NSArray *)arrayWithString:(NSString *)str;
@end

@implementation SomeClass
- (void)foo{
    NSLog(@"method foo was called on %@",[self class]);
}

void crashMethod(id obj, SEL _cmd) {
    NSLog(@"crash Method");
}

+ (BOOL)resolveInstanceMethod:(SEL)sel {
    if(sel == @selector(crash)){
        class_addMethod([self class], sel, (IMP)crashMethod, "v@:");
        return YES;
    }
    return [super resolveInstanceMethod:sel];
}

#pragma mark - 备用接收者
- (id)forwardingTargetForSelector:(SEL)aSelector
{
    //获取方法名
    NSString *selectorString = NSStringFromSelector(aSelector);
    //根据方法名添加方法
    if ([selectorString isEqualToString:@"arrayWithString:"]) {
        OtherClass *otherClass = [[OtherClass alloc] init];
        
        return otherClass;
    }
    
    return [super forwardingTargetForSelector:aSelector];
}
@end
```

```
#import "OtherClass.h"

@implementation OtherClass
/**
 *  把字符串转换为数组
 *
 *  @param str 需转换的字符串
 *
 *  @return 转换好的数组
 */
- (NSArray *)arrayWithString:(NSString *)str
{
    if (str && (str != NULL) && (![str isKindOfClass:[NSNull class]]) && str.length > 0) {
        NSMutableArray *mArr = [NSMutableArray arrayWithCapacity:1];
        for (NSInteger index = 0; index < str.length; index++) {
            [mArr addObject:[str substringWithRange:NSMakeRange(index, 1)]];
        }
        
        return mArr;
    }
    
    return nil;
}
@end
```



##### 完整转发

* - \(void\)forwardInvocation:\(NSInvocation \*\)anInvocation

* 对象需要创建一个NSInvocation对象，把消息调用的全部细节封装进去，包括selector, target, arguments 等参数，还能够对返回结果进行处理
* 为了使用完整转发，需要重写以下方法
  * -\(NSMethodSignature \*\)methodSignatureForSelector:\(SEL\)aSelector，如果2中return nil,执行methodSignatureForSelector：
  * 因为消息转发机制为了创建NSInvocation需要使用这个方法吗获取信息，重写它为了提供合适的方法签名

























