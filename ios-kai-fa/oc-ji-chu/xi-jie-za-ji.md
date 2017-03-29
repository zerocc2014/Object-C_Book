# 1. 类与对象

## 类方法

OC中类的方法只有实例方法和静态方法\(类方法\)两种：

``` objectivec
@interface Controller : NSObject

+ (void)thisIsAStaticMethod; // 静态方法
– (void)thisIsAnInstanceMethod; // 实例方法

@end
```

OC 没有像 Java，C++ 中的那种绝对的私有及保护成员方法，仅仅可以对调用者隐藏某些方法（声明和实现都写在 @implementation 里）。

可以使用 Category 来实现私有方法，给myClass类添加一个分类\(Category\)，在分类.h中写上该私有方法声明，分类.m中不用写实现。这就是`私有方法的前向引用`：

``` objectivec
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

``` objectivec
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

## 类变量 {#类变量}

Objective-C 中使用`@property`来实现成员变量，在绑定属性时，如果我们直接把属性暴露出去，虽然写起来很简单，但是没办法检查参数，导致可以随便改，而`@property`可以通过一个`set`方法来设置参数，再通过一个`get`来获取参数值，这样在`set`方法里，就可以检查参数.

成员变量 的存取 （读写）

```
// Car.h
@interface Car : NSObject {
    // 实例变量
    NSString *carName;
    NSString *carType;
}
// setter
- (void)setCarName:(NSString *)newCarName;
// getter
- (NSString *)carName;
@end

#import "Car.h"

@implementation Car
// setter
- (void)setCarName:(NSString *)newCarName {
    carName = newCarName;
}
// getter
- (NSString *)carName {
    return carName;
}
@end
```

#### **@synthesize**

@synthesize 表示由编译器来自动实现属性的getter/setter方法，不需要你自己再手动去实现。默认情况下，不需要指定实例变量的名称，编译器会自动生成一个属性名前加“\_”的实例变量。当然也可以在实现代码 .m 文件里通过@synthesize语法来指定实例变量的名字。

#### **@dynamic**

简单来讲，通过@synthesize指令告诉编译器在编译期间产生getter和setter方法。如果自定义getter和setter方法则会覆盖编译器帮我们生成的方法。@dynamic指令告诉编译器在编译期间不自动生成getter和setter方法，避免编译期间产生警告。然后由自己实现存取方法或存取方法在运行时动态创建绑定。其主要作用就是用在NSManageObject对象的属性声明中，由于此类对象的属性一般是从Core Data的属性中生成的，Core Data框架会在程序运行的时候为此类属性生成getter和setter方法。

#### _**@property**_

其本质：@property = ivar + getter + setter;

“属性” \(property\)有两大概念：ivar（实例变量）、存取方法（access method ＝ getter + setter）。

“属性” \(property\)作为 Objective-C 的一项特性，主要的作用就在于封装对象中的数据。 Objective-C 对象通常会把其所需要的数据保存为各种实例变量。实例变量一般通过“存取方法”\(access method\)来访问。其中，“获取方法” \(getter\)用于读取变量值，而“设置方法” \(setter\)用于写入变量值。 而在正规的 Objective-C 编码风格中，存取方法有着严格的命名规范。 正因为有了这种严格的命名规范，所以 Objective-C 这门语言才能根据名称自动创建出存取方法。其实也可以把属性当做一种关键字，其表示:

1. @property有两个对应的词，一个是@synthesize，一个是@dynamic。如果@synthesize和@dynamic都没写，那么默认的就是@syntheszie var = \_var;
2. @synthesize的语义是如果你没有手动实现setter方法和getter方法，那么编译器会自动为你加上这两个方法。
3. @dynamic告诉编译器：属性的setter与getter方法由用户自己实现，不自动生成。（当然对于readonly的属性只需提供getter即可）。假如一个属性被声明为@dynamic var，然后你没有提供@setter方法和@getter方法，编译的时候没问题，但是当程序运行到instance.var = someVar，由于缺setter方法会导致程序崩溃；或者当运行到 someVar = var时，由于缺getter方法同样会导致崩溃。编译时没问题，运行时才执行相应的方法，这就是所谓的动态绑定。

4. [ ] 读写属性： （readwrite/readonly）

5. [ ] setter语意：（assign/retain/copy）

6. [ ] 原子性： （atomicity/nonatomic）

7. **assign**：默认关键字。非对象类型一般使用此关键字。assign用于值类型，如int、float、double和NSInteger，CGFloat等表示单纯的复制。还包括不存在所有权关系的对象，比如常见的delegate。在setter方法中，采用直接赋值来实现设值操作：

```
-(void)setRunning:(int)newRunning{  
    _running = newRunning;  
}
```

* **retain**：对象的引用计数+ 1.ARC下已经不再使用此关键字，用强代替。 简单来说，就是对传入的对象拥有所有权，只要对该对象拥有所有权，该对象就不会被释放。如下代码所示：

```
-(void)setName:(NSString*)_name{  
     //首先判断是否与旧对象一致，如果不一致进行赋值。  
     //因为如果是一个对象的话，进行if内的代码会造成一个极端的情况：当此name的retain为1时，使此次的set操作让实例name提前释放，而达不到赋值目的。  
     if ( name != _name){  
          [name release];  
          name = [_name retain];  
     }  
}
```

* **strong**：能够维持对象的生命。strong是在IOS引入ARC的时候引入的关键字，是retain的一个可选的替代。表示实例变量对传入的对象要有所有权关系，即强引用。strong跟retain的意思相同并产生相同的代码，但是语意上更好更能体现对象的关系。
* **copy**：拷贝一个新的对象，新对象的引用计数+1，原对象不变。与strong类似，但区别在于实例变量是对传入对象的副本拥有所有权，而非对象本身。

```
-(void)setThetest:(test *)newThetest {  
    if (thetest!= newThetest) {  
　　      thetest= [newThetest copy];  
    }  
}
```

* **weak**：在setter方法中，需要对传入的对象不进行引用计数加1的操作。简单来说，就是对传入的对象没有所有权，当该对象引用计数为0时，用weak声明的实例变量指向nil，即：所有指向该对象的weak属性指针会被自动设置为nil。

  注：weak关键字是IOS5引入的，iOS5之前是不能使用该关键字的。delegate 和 Outlet 一般用weak来声明。

##### **atomic与nonatomic**

* atomic:默认是有该属性的，这个属性是为了保证程序在多线程情况下，编译器会自动生成一些互斥加锁代码，避免该变量的读写不同步的问题，提供多线程安全。
* nonatomic:如果该对象无需考虑多线程的情况，请加入这个属性，这样会让编译器少生成一些互斥加锁代码，禁止多线程，变量保护，提高性能和效率。

**注：**atomic是Objc使用的一种线程保护技术，基本上来讲是防止在写未完成的时候另一个线程读取，造成的数据错误。而这种机制是非常耗费系统资源的，所以iPhone这种小型设备上，如果没有使用多线程间的通讯编程，那么nonatomic是一个非常好的选择。而iOS开发中，普遍使用nonatomic也是基于性能这一点。

#### 深拷贝\(mutableCopy\)和浅拷贝\(copy\)：

深拷贝就是内容拷贝，浅拷贝就是指针拷贝。

retain和strong都是指针拷贝。当有其他对象引用当前对象时，会拷贝一份当前对象的地址，这样它就也指向当前对象了。所以，还是同一个对象，只是retainCount+1；

copy:对于不可变对象copy采用的是浅复制，引用计数器加1（其实这是编译器进行了优化，既然原来的对象不可变，复制之后的对象也不可变那么就没有必要再重新创建一个对象了）；对于可变对象copy采用的是深复制，引用计数器不变（原来的对象是可变，现在要产生一个不可变的当然得重新产生一个对象）；

#### 强引用和弱引用

**强引用：**当前对象被其他对象引用时，会执行retain操作，引用计数器+1。当retainCount=0时，该对象才会被销毁。因为我们要进行对象的内存管理，所以这是默认的引用方式。（默认是强引用）

**弱引用：**当前对象的生命周期不被是否由其他对象引用限制，它本该什么时候销毁就什么时候被销毁。即使它的引用没断，但是当它的生存周期到了时就会被销毁。

在定义属性时，若声明为retain类型的，则就是强引用；若声明为assign类型的，则就是弱引用。后来内存管理都由ARC来完成后，若是强引用，则就声明为strong；若是弱引用，则就声明为weak。所以说，retain和strong是一致的（声明为强引用）；assign和weak是基本一致的（声明为弱引用）。之所以说它俩是基本一致是因为它俩还是有所不同的，weak严格的说应当叫“_**归零弱引用**_”，即当对象被销毁后，会自动的把它的指针置为nil，这样可以防止野指针错误。而assign销毁对象后不会把该对象的指针置nil，对象已经被销毁，但指针还在痴痴的指向它，这就成了野指针，这是比较危险的。

#### 类中添加全局变量 {#如何在类中添加全局变量}

有些时候我们需要在类中添加某个在类中全局可用的变量，为了避免污染作用域，一个比较好的做法是在 .m 文件中使用 static 变量：

```
static NSOperationQueue * _personOperationQueue = nil;

@implementation XYZPerson
...
@end
```

由于 static 变量在编译期就是确定的，因此对于 NSObject 对象来说，初始化的值只能是 nil。如何进行类似 init 的初始化呢？可以通过重载 initialize 方法来做：

```
@implementation XYZPerson
- (void)initialize {
    if (!_personOperationQueue) {
        _personOperationQueue = [[NSOperationQueue alloc] init];
    }
}
@end
```

为什么这里要判断是否为 nil 呢？因为`initialize`方法可能会调用多次，后面会提到。

如果是在 Category 中想声明全局变量呢？当然也可以通过 initialize，不过除非必须的情况下，并不推荐在 Category 当中进行方法重载。

有一种方法是声明 static 函数，下面的代码来自 [AFNetworking](https://github.com/AFNetworking/AFNetworking/blob/master/AFNetworking/AFURLSessionManager.m\) ，声明了一个当前文件范围可用的队列：

```
static dispatch_queue_t url_session_manager_creation_queue() {
    static dispatch_queue_t af_url_session_manager_creation_queue;
    static dispatch_once_t onceToken;
    dispatch_once(&onceToken, ^{
        af_url_session_manager_creation_queue = dispatch_queue_create("com.alamofire.networking.session.manager.creation", DISPATCH_QUEUE_SERIAL);
    });
    return af_url_session_manager_creation_queue;
}
```

下面介绍一个有点黑魔法的方法，除了上面两种方法之外，我们还可以通过编译器的`__attribute__`特性来实现初始化：

```
__attribute__((constructor))
static void initialize_Queue() {
    _personOperationQueue = [[NSOperationQueue alloc] init];
}

@implementation XYZPerson (Operation)

@end
```

## 类的导入

导入类可以使用`#include`,`#import`和`@class`三种方法，其区别如下：

* `#import`是Objective-C导入头文件的关键字，`#include`是C/C++导入头文件的关键字
* 使用`#import`头文件会自动只导入一次，不会重复导入，相当于`#include`和`#pragma once`；
* `@class`告诉编译器需要知道某个类的声明，可以解决头文件的相互包含问题；

`@class`是放在interface中的，只是在引用一个类，将这个被引用类作为一个类型使用。在实现文件中，如果需要引用到被引用类的实体变量或者方法时，还需要使用`#import`方式引入被引用类。

## 类的初始化 {#类的初始化}

Objective-C 是建立在 Runtime 基础上的语言，类也不例外。OC 中类是初始化也是动态的。在 OC 中绝大部分类都继承自`NSObject`，它有两个非常特殊的类方法`load`和`initilize`，用于类的初始化

#### +load

+load 方法是当类或分类被添加到 Objective-C runtime 时被调用的，实现这个方法可以让我们在类加载的时候执行一些类相关的行为。子类的 +load 方法会在它的所有父类的 +load 方法之后执行，而分类的 +load 方法会在它的主类的 +load 方法之后执行。但是不同的类之间的 +load 方法的调用顺序是不确定的。

load 方法不会被类自动继承, 每一个类中的 load 方法都不需要像 viewDidLoad 方法一样调用父类的方法。子类、父类和分类中的 +load 方法的实现是被区别对待的。也就是说如果子类没有实现 +load 方法，那么当它被加载时 runtime 是不会去调用父类的 +load 方法的[1](https://hit-alibaba.github.io/interview/iOS/ObjC-Basic/Class.html#fn:ref)。同理，当一个类和它的分类都实现了 +load 方法时，两个方法都会被调用。因此，我们常常可以利用这个特性做一些“邪恶”的事情，比如说方法混淆（Method Swizzling）。FDTemplateLayoutCell 中就使用了这个方法，见[这里](https://github.com/forkingdog/UITableView-FDTemplateLayoutCell/blob/2bead7b80e40e8689201e7c1d6f034e952c9a155/Classes/UITableView%2BFDIndexPathHeightCache.m#L147)。

#### +initialize {#initialize}

+initialize 方法是在类或它的子类收到第一条消息之前被调用的，这里所指的消息包括实例方法和类方法的调用。也就是说 +initialize 方法是以懒加载的方式被调用的，如果程序一直没有给某个类或它的子类发送消息，那么这个类的 +initialize 方法是永远不会被调用的。

+initialize 方法的调用与普通方法的调用是一样的，走的都是发送消息的流程。换言之，如果子类没有实现 +initialize 方法，那么继承自父类的实现会被调用；如果一个类的分类实现了 +initialize 方法，那么就会对这个类中的实现造成覆盖。



###  参考资料 {#参考资料}

[iOS开发基础面试题系列](http://blog.csdn.net/xunyn/article/details/8302787)

* [10个Objective-C基础面试题，iOS面试必备](http://www.oschina.net/news/42288/10-objective-c-interview)
* [Objective-C中“私有方法”的实现"](http://blog.sina.com.cn/s/blog_74e9d98d01013au8.html)
* [Objective-C中@property详解](http://www.cnblogs.com/andyque/archive/2011/08/03/2125728.html)
* [Objective-C中的protocol和delegate](http://www.cnblogs.com/whyandinside/archive/2013/02/28/2937217.html)
* [Objective-C——消息，Category 与 Protocol](http://www.cnblogs.com/chijianqiang/archive/2012/06/22/objc-category-protocol.html)
* [深入理解Objective-C中的@class](http://www.cnblogs.com/martin1009/archive/2012/06/24/2560218.html)
* [Objective-C +load vs +initialize](http://blog.leichunfeng.com/blog/2015/05/02/objective-c-plus-load-vs-plus-initialize/)
* [深入理解Objective-C：Category](http://tech.meituan.com/DiveIntoCategory.html)
* [https://stackoverflow.com/questions/19784454/when-should-i-use-synthesize-explicitly](https://stackoverflow.com/questions/19784454/when-should-i-use-synthesize-explicitly)
* [http://www.fantageek.com/blog/2014/07/13/property-in-protocol/](http://www.fantageek.com/blog/2014/07/13/property-in-protocol/)
* [http://www.friday.com/bbum/2009/09/06/iniailize-can-be-executed-multiple-times-load-not-so-much/](http://www.friday.com/bbum/2009/09/06/iniailize-can-be-executed-multiple-times-load-not-so-much/)

基础知识大全：[http://lib.csdn.net/qq\_31810357/286174/chart/%E3%80%90Objective-C%E3%80%91%E5%9F%BA%E7%A1%80%E7%9F%A5%E8%AF%86%E5%A4%A7%E5%85%A8](http://lib.csdn.net/qq_31810357/286174/chart/【Objective-C】基础知识大全)

