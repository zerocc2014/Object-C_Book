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

## OC方法动态调用基础术语

OC只是在编译阶段确定了要向接收者发送message这条消息，而receive将要如何响应这条消息，那就要看运行时发生的情况来决定了。通过发送消息来达到动态调用：

> \[obj makeText\];   编译时runtime会将其转化为：objc\_msgSend\(obj,@selector\(makeText\)\);

也就是说 objc\_msgSend 函数相当于入口；对象调用某个方法都将被编译器转化为：

```
id objc_msgSend(id self, SEL op, ... );  // objc_msgSend(obj, selector, arg1, arg2, ...)
```

如果消息的接收者能够找到对应的selector，那么就相当于直接执行了接收者这个对象的特定方法；否则，消息要么被转发，或是临时向接收者动态添加这个 selector 对应的实现内容，要么就崩溃掉。

下面将会逐渐展开介绍一些术语，及它们各自对应的数据结构。

### SEL

objc\_msgSend函数第二个参数类型为SEL，它是selector在Objc中的表示类型（Swift中是Selector类）。selector是方法选择器，可以理解为区分方法的 ID，而这个 ID 的数据结构是SEL:

```
typedef struct objc_selector *SEL;
```

其实它就是个映射到方法的C字符串，你可以用 Objc 编译器命令@selector\(\)或者 Runtime 系统的 sel\_registerName函数来获得一个SEL类型的方法选择器。

不同类中相同名字的方法所对应的方法选择器是相同的，即使方法名字相同而变量类型不同也会导致它们具有相同的方法选择器，于是 Objc 中方法命名有时会带上参数类型\(NSNumber一堆抽象工厂方法\)，Cocoa 中有好多长长的方法....。

### id

objc\_msgSend第一个参数类型为id，objc.h中可以查看,它是一个指向类实例的指针：

```
typedef struct objc_object *id;
```

objc\_object又是：

```
struct objc_object {
    Class isa  OBJC_ISA_AVAILABILITY;
};
```

objc\_object结构体包含一个isa指针，指向它的类别Class，根据isa指针就可以一层层找到对象所属的类。

### Class

之所以说 isa 是指针是因为Class其实是一个指向 objc\_class 结构体的指针：

```
typedef struct objc_class *Class;
```

```
而objc_class，查看 <objc/runtime.h> 中 objc_class 结构体的定义如下：
```

```
struct objc_class {
    Class isa  OBJC_ISA_AVAILABILITY;

#if !__OBJC2__
    Class super_class                                        OBJC2_UNAVAILABLE;
    const char *name                                         OBJC2_UNAVAILABLE;
    long version                                             OBJC2_UNAVAILABLE;
    long info                                                OBJC2_UNAVAILABLE;
    long instance_size                                       OBJC2_UNAVAILABLE;
    struct objc_ivar_list *ivars                             OBJC2_UNAVAILABLE;
    struct objc_method_list **methodLists                    OBJC2_UNAVAILABLE;
    struct objc_cache *cache                                 OBJC2_UNAVAILABLE;
    struct objc_protocol_list *protocols                     OBJC2_UNAVAILABLE;
#endif

} OBJC2_UNAVAILABLE;
```

可以看到运行时一个类还关联了它的超类指针，类名，成员变量，方法，缓存，还有附属的协议。

\*\*methodLists//指针的指针，可以动态修改\*methodLists的值来添加成员方法,同样解释了Category不能添加属性的原因,二级指针

其中objc\_ivar\_list和objc\_method\_list分别是成员变量列表和方法列表：

```
struct objc_ivar_list {
            int ivar_count                                           OBJC2_UNAVAILABLE;
        #ifdef __LP64__
            int space                                                OBJC2_UNAVAILABLE;
        #endif
            /* variable length structure */
            struct objc_ivar ivar_list[1]                            OBJC2_UNAVAILABLE;
        }                                                            OBJC2_UNAVAILABLE;
        struct objc_method_list {
            struct objc_method_list *obsolete                        OBJC2_UNAVAILABLE;
            int method_count                                         OBJC2_UNAVAILABLE;
        #ifdef __LP64__
            int space                                                OBJC2_UNAVAILABLE;
        #endif
            /* variable length structure */
            struct objc_method method_list[1]                        OBJC2_UNAVAILABLE;
        }
```

### **Method**

Method是一种代表类中的某个方法的类型。

```
typedef struct objc_method *Method;
```

而objc\_method在上面的方法列表中提到过，它存储了方法名，方法类型和方法实现：

```
struct objc_method {
        SEL method_name                                          OBJC2_UNAVAILABLE;
        char *method_types                                       OBJC2_UNAVAILABLE;
        IMP method_imp                                           OBJC2_UNAVAILABLE;
    }
```

* 方法名类型为SEL，前面提到过相同名字的方法即使在不同类中定义，它们的方法选择器也相同。
* 方法类型method\_types是个char指针，其实存储着方法的参数类型和返回值类型。
* method\_imp指向了方法的实现，本质上是一个函数指针，后面会详细讲到

### IMP

函数指针 - IMP 在objc.h中的定义是：

```
typedef id (*IMP)(id, SEL, ...);
```

它就是一个函数指针，这是由编译器生成的。当你发起一个 ObjC 消息之后，最终它会执行的那段代码，就是由这个函数指针指定的。而IMP这个函数指针就指向了这个方法的实现。既然得到了执行某个实例某个方法的入口，我们就可以绕开消息传递阶段，直接执行方法，这在后面会提到。

你会发现IMP指向的方法与objc\_msgSend函数类型相同，参数都包含id和SEL类型。每个方法名都对应一个SEL类型的方法选择器，而每个实例对象中的SEL对应的方法实现肯定是唯一的，通过一组id和SEL参数就能确定唯一的方法实现地址；反之亦然。

每个类都有一个方法列表，存放着selector的名字和方法实现的映射关系。IMP有点类似函数指针，指向具体的Method实现，SEL与IMP之间的关系图：

![](/assets/SEL&IMP.png)

获取方法地址IMP避开消息绑定而直接获取方法的地址并调用方法。这种做法很少用，除非是需要持续大量重复调用某方法的极端情况，避开消息发送泛滥而直接调用该方法会更高效。NSObject类中有个methodForSelector:实例方法，你可以用它来获取某个方法选择器对应的IMP，举个栗子：

```
void (*setter)(id, SEL, BOOL);
        int i;
        setter = (void (*)(id, SEL, BOOL))[target
                                           methodForSelector:@selector(setFilled:)];
        for ( i = 0 ; i < 1000 ; i++ )
        setter(targetList[i], @selector(setFilled:), YES);
```

### Cache

Cache在 runtime.h 中的定义：

```
typedef struct objc_cache *Cache                             OBJC2_UNAVAILABLE;
```

在类 objc\_class 结构体中有一个struct objc\_cache \*cache，它到底是缓存啥的呢，先看看objc\_cache 的实现：

```
struct objc_cache {
        unsigned int mask /* total = mask + 1 */                 OBJC2_UNAVAILABLE;
        unsigned int occupied                                    OBJC2_UNAVAILABLE;
        Method buckets[1]                                        OBJC2_UNAVAILABLE;
    };
```

Cache为方法调用的性能进行优化,通俗地讲,每当实例对象接收到一个消息时,它不会直接在isa指向的类的方法列表中遍历查找能够响应消息的方法，因为这样效率太低了，而是优先在Cache中查找。Runtime系统会把被调用的方法存到Cache中（理论上讲一个方法如果被调用，那么它有可能今后还会被调用）method\_name作为key，method\_imp作为value给存起来，下次查找的时候效率更高。这根计算机组成原理中学过的CPU绕过主存先访问Cache的道理挺像，猜测苹果为提高Cache命中率应该也做了努力吧。高速缓存\(cache\) -&gt;内存-&gt;虚拟内存-&gt;磁盘

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

##### 完整消息转发

* * \(void\)forwardInvocation:\(NSInvocation \*\)anInvocation
* 对象需要创建一个NSInvocation对象，把消息调用的全部细节封装进去，包括selector, target, arguments 等参数，还能够对返回结果进行处理

* 为了使用完整转发，需要重写以下方法

  * -\(NSMethodSignature \*\)methodSignatureForSelector:\(SEL\)aSelector，如果2中return nil,执行methodSignatureForSelector：
  * 因为消息转发机制为了创建NSInvocation需要使用这个方法吗获取信息，重写它为了提供合适的方法签名

如果在上一步备用接收者还不能处理未知消息，则唯一能做的就是启用完整的消息转发机制了。此时会调用方法：

```
- (void)forwardInvocation:(NSInvocation *)anInvocation，
```

对象会创建一个表示消息的 NSInvocation 对象，把与尚未处理的消息有关的全部细节都封装在anInvocation中，包括selector，目标\(target\)和参数。我们可以在forwardInvocation方法中选择将消息转发给其它对象。forwardInvocation:方法的实现有两个任务：  
   &lt; 1&gt; 定位可以响应封装在anInvocation中的消息的对象。这个对象不需要能处理所有未知消息。  
  &lt; 2 &gt; 使用anInvocation作为参数，将消息发送到选中的对象。anInvocation将会保留调用结果，运行时系统会提取这一结果并将其发送 到消息的原始发送者。

还有一个很重要的问题，我们必须重写以下方法：- \(NSMethodSignature \*\)methodSignatureForSelector:\(SEL\)aSelector,消息转发机制使用从这个方法中获取的信息来创建NSInvocation对象。因此我们必须重写这个方法，为给定的selector提供一个合适的方法签名。

向OtherClass.m中添加如下方法：

```
/**
 *  逆置字符串
 *
 *  @param str 需逆置的字符串
 *
 *  @return 置换后的字符串
 */
- (NSString *)inverseWithString:(NSString *)str
{
    if (str && (str != NULL) && (![str isKindOfClass:[NSNull class]]) && str.length > 0) {
        NSMutableString *mStr = [NSMutableString stringWithCapacity:1];
        for (NSInteger index = str.length; index > 0; index--) {
            [mStr appendString:[str substringWithRange:NSMakeRange(index - 1, 1)]];
        }
        return mStr;
    }
    return nil;
}
```

向SomeClass.m中添加如下方法：

```
#pragma mark - 完整消息转发
//必须重写这个方法，为给定的selector提供一个合适的方法签名。
- (NSMethodSignature *)methodSignatureForSelector:(SEL)aSelector
{
    NSMethodSignature *signature = [super methodSignatureForSelector:aSelector];
    if (!signature) {
        if ([OtherClass instancesRespondToSelector:aSelector]) {
            //获取方法签名
            signature = [OtherClass instanceMethodSignatureForSelector:aSelector];
        }
    }
    return signature;
}

- (void)forwardInvocation:(NSInvocation *)anInvocation
{
    //anInvocation选择将消息转发给其它对象
    if ([OtherClass instancesRespondToSelector:anInvocation.selector]) {
        [anInvocation invokeWithTarget:[[OtherClass alloc] init]];
    }
}
```

发送消息的整体流程图：

![](/assets/methodForward.png)

## Runtime 运用

* 能获得某个类的所有成员变量
* 能获得某个类的所有属性
* 能获得某个类的所有方法

* 交换方法实现

* 能动态添加一个成员变量

* 能动态添加一个属性

* 字典转模型

* runtime归档/反归档

### 1. 交换方法

* 开发使用场景:系统自带的方法功能不够，给系统自带的方法扩展一些功能，并且保持原有的功能。
* 方式一:继承系统的类，重写方法
* 方式二:使用runtime,交换方法.

```
@implementation ViewController


- (void)viewDidLoad {
    [super viewDidLoad];
    // Do any additional setup after loading the view, typically from a nib.
    // 需求：给imageNamed方法提供功能，每次加载图片就判断下图片是否加载成功。
    // 步骤一：先搞个分类，定义一个能加载图片并且能打印的方法+ (instancetype)imageWithName:(NSString *)name;
    // 步骤二：交换imageNamed和imageWithName的实现，就能调用imageWithName，间接调用imageWithName的实现。
    UIImage *image = [UIImage imageNamed:@"123"];
}
@end

@implementation UIImage (Image)
// 加载分类到内存的时候调用
+ (void)load
{
    // 交换方法

    // 获取imageWithName方法地址
    Method imageWithName = class_getClassMethod(self, @selector(imageWithName:));

    // 获取imageWithName方法地址
    Method imageName = class_getClassMethod(self, @selector(imageNamed:));

    // 交换方法地址，相当于交换实现方式
    method_exchangeImplementations(imageWithName, imageName);
}

// 不能在分类中重写系统方法imageNamed，因为会把系统的功能给覆盖掉，而且分类中不能调用super.

// 既能加载图片又能打印
+ (instancetype)imageWithName:(NSString *)name
{
    // 这里调用imageWithName，相当于调用imageName
    UIImage *image = [self imageWithName:name];

    if (image == nil) {
        NSLog(@"加载空的图片");
    }
    return image;
}
@end
```

### 2. 动态添加方法

* 开发使用场景：如果一个类方法非常多，加载类到内存的时候也比较耗费资源，需要给每个方法生成映射表，可以使用动态给某个类，添加方法解决。
* 经典面试题：有没有使用performSelector，其实主要想问你有没有动态添加过方法。
* 简单使用

```
@implementation ViewController

- (void)viewDidLoad {
    [super viewDidLoad];
    // Do any additional setup after loading the view, typically from a nib.

    Person *p = [[Person alloc] init];

    // 默认person，没有实现eat方法，可以通过performSelector调用，但是会报错。
    // 动态添加方法就不会报错
    [p performSelector:@selector(eat)];

}
@end

@implementation Person
// void(*)()
// 默认方法都有两个隐式参数，
void eat(id self,SEL sel)
{
    NSLog(@"%@ %@",self,NSStringFromSelector(sel));
}

// 当一个对象调用未实现的方法，会调用这个方法处理,并且会把对应的方法列表传过来.
// 刚好可以用来判断，未实现的方法是不是我们想要动态添加的方法
+ (BOOL)resolveInstanceMethod:(SEL)sel
{
    if (sel == @selector(eat)) {
        // 动态添加eat方法

        // 第一个参数：给哪个类添加方法
        // 第二个参数：添加方法的方法编号
        // 第三个参数：添加方法的函数实现（函数地址）
        // 第四个参数：函数的类型，(返回值+参数类型) v:void @:对象->self :表示SEL->_cmd
        class_addMethod(self, @selector(eat), eat, "v@:");
    }
    return [super resolveInstanceMethod:sel];
}
@end
```

### 3. 给分类添加属性

原理：给一个类声明属性，其实本质就是给这个类添加关联，并不是直接把这个值的内存空间添加到类存空间。

```
@implementation ViewController

- (void)viewDidLoad {
    [super viewDidLoad];

    // 给系统NSObject类动态添加属性name
    NSObject *objc = [[NSObject alloc] init];
    objc.name = @"zerocc";
    NSLog(@"%@",objc.name);
}
@end

// 定义关联的key
static const char *key = "name";

@implementation NSObject (Property)

- (NSString *)name
{
    // 根据关联的key，获取关联的值。
    return objc_getAssociatedObject(self, key);
}

- (void)setName:(NSString *)name
{
    // 第一个参数：给哪个对象添加关联
    // 第二个参数：关联的key，通过这个key获取
    // 第三个参数：关联的value
    // 第四个参数:关联的策略
    objc_setAssociatedObject(self, key, name, OBJC_ASSOCIATION_RETAIN_NONATOMIC);
}
@end
```

### 4. 归档和解档 一键序列化

* 原理描述：用runtime提供的函数遍历Model自身所有属性，并对属性进行encode和decode操作。
* 核心方法：在Model的基类中重写方法：

如果需要实现一些基本数据的数据持久化\(data persistance\)或者数据共享\(data share\)。我们可以选择归档和解档。如果用一般的方法:

```
- (void)encodeWithCoder:(NSCoder *)aCoder {
    [aCoder encodeObject:self.name forKey:@"nameKey"];
    [aCoder encodeObject:self.gender forKey:@"genderKey"];
    [aCoder encodeObject:[NSNumber numberWithInteger:self.age] forKey:@"ageKey"];
}
```

也可以实现。但是如果实体类有很多的成员变量，这种方法很显然就无力了。这个时候，我们就可以利用`runtime`来实现快速归档、解档:

* 让实体类遵循`<NSCoding>`协议。并在.m文件导入头文件`<objc/runtime.h>`。
* 实现`- (instancetype)initWithCoder:(NSCoder *)aDecoder`和`- (void)encodeWithCoder:(NSCoder *)aCoder`方法。

```
- (instancetype)initWithCoder:(NSCoder *)aDecoder {
    self = [super init];
    if (self) {
        //
        unsigned int count = 0;
        objc_property_t *properties = class_copyPropertyList([self class], &count);
        for (int i = 0; i < count; i ++) {
            objc_property_t property = properties[i];
            const char *propertyChar = property_getName(property);
            NSString *propertyString = [NSString stringWithUTF8String:propertyChar];
            id value = [aDecoder decodeObjectForKey:propertyString];
            [self setValue:value forKey:propertyString];
        }
        free(properties);
    }
    return self;
}

- (void)encodeWithCoder:(NSCoder *)aCoder {
    unsigned int count = 0;
    objc_property_t *properties = class_copyPropertyList([self class], &count);
    for (int i = 0; i < count; i ++) {
        objc_property_t property = properties[i];
        const char *propertyChar = property_getName(property);
        NSString *propertyString = [NSString stringWithUTF8String:propertyChar];
        id object = [self valueForKey:propertyString];
        [aCoder encodeObject:object forKey:propertyString];
    }
    free(properties);
}
```

或者这种写法：

```
- (id)initWithCoder:(NSCoder *)aDecoder {
    if (self = [super init]) {
        unsigned int outCount;
        Ivar * ivars = class_copyIvarList([self class], &outCount);
        for (int i = 0; i < outCount; i ++) {
            Ivar ivar = ivars[i];
            NSString * key = [NSString stringWithUTF8String:ivar_getName(ivar)];
            [self setValue:[aDecoder decodeObjectForKey:key] forKey:key];
        }
    }
    return self;
}

- (void)encodeWithCoder:(NSCoder *)aCoder {
    unsigned int outCount;
    Ivar * ivars = class_copyIvarList([self class], &outCount);
    for (int i = 0; i < outCount; i ++) {
        Ivar ivar = ivars[i];
        NSString * key = [NSString stringWithUTF8String:ivar_getName(ivar)];
        [aCoder encodeObject:[self valueForKey:key] forKey:key];
    }
}
```

在main.m 函数中测试归档解档：

```
#import <Foundation/Foundation.h>
#import "Model.h"

int main(int argc, const char * argv[]) {
    @autoreleasepool {
        Model *model = [Model new];
        model.name = @"kxx";
        model.age = 24;
        model.gender = @"male";

        NSString *path = [NSHomeDirectory() stringByAppendingPathComponent:@"modelData.data"];

        // 归档
        BOOL flag = [NSKeyedArchiver archiveRootObject:model toFile:path];
        if (flag) {
            NSLog(@"archive object successfully!!!");
        }

        // 解档
        Model *unarchivedModel = [NSKeyedUnarchiver unarchiveObjectWithFile:path];

        NSLog(@"\n\n%@\n%@\n%ld", unarchivedModel.name, unarchivedModel.gender, unarchivedModel.age);
    }
    return 0;
}
```

### 5. 控制器的万能跳转

```
- (void)testRuntime
{
    NSDictionary *userInfo = @{@"class":@"CCRuntimePushVC",
                               @"property": @{
                                       @"ID":@"81198",
                                       @"type":@"2"
                                       }
                               };
    [self push:userInfo];
}

// 跳转
- (void)push:(NSDictionary *)params
{
    // 得到类名
    NSString *className = [NSString stringWithFormat:@"%@",params[@"class"]];
    
    // 通过名称转换成Class
    Class getClass = NSClassFromString([NSString stringWithFormat:@"%@",className]);
    
    // 判断得到的这个class 是否存在
    if (getClass) {
        // 创建 class 对象
        id creatClass = [[getClass alloc] init];
        
        NSDictionary *propertys = params[@"property"];
        [propertys enumerateKeysAndObjectsUsingBlock:^(id  _Nonnull key, id  _Nonnull obj, BOOL * _Nonnull stop) {
            
            if ([self checkIsExistPropertyWithInstance:creatClass verifyPropertyName:key]) {
                // 利用 kvc 赋值
                [creatClass setValue:obj forKey:key];
            }
        }];
        
        [self.navigationController pushViewController:creatClass animated:YES];
    }else{
        NSLog(@"not this class,can not push");
    }
}

// 检查对象是否存在该属性
- (BOOL)checkIsExistPropertyWithInstance:(id)instance verifyPropertyName:(NSString *)verifyPropertyName
{
    unsigned int outCount, i;
    // 获取对象的属性列表
    objc_property_t *properties = class_copyPropertyList([instance class], &outCount);
    
    for (i = 0; i < outCount; i++) {
        objc_property_t property = properties[i];
        // 属性名转换成字符串
        NSString *propertyName = [[NSString alloc] initWithCString:property_getName(property)  encoding:NSUTF8StringEncoding];
        // 判断该属性是否存在
        if ([propertyName isEqualToString:verifyPropertyName]) {
            free(properties);
            
            return YES;
        }
    }
    free(properties);
    
    return NO;
}

- (void)touchesBegan:(NSSet<UITouch *> *)touches withEvent:(UIEvent *)event {
    [self testRuntime];
}

```





































参考：[https://halfrost.com/objc\_runtime\_isa\_class/](https://halfrost.com/objc_runtime_isa_class/)

