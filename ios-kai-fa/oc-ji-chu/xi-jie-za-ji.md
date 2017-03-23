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

* [ ] 读写属性： （readwrite/readonly）
* [ ] setter语意：（assign/retain/copy）
* [ ] 原子性： （atomicity/nonatomic）

* **assign**：默认关键字。非对象类型一般使用此关键字。assign用于值类型，如int、float、double和NSInteger，CGFloat等表示单纯的复制。还包括不存在所有权关系的对象，比如常见的delegate。在setter方法中，采用直接赋值来实现设值操作：

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

  注：weak关键字是IOS5引入的，IOS5之前是不能使用该关键字的。delegate 和 Outlet 一般用weak来声明。

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





基础知识大全：[http://lib.csdn.net/qq\_31810357/286174/chart/%E3%80%90Objective-C%E3%80%91%E5%9F%BA%E7%A1%80%E7%9F%A5%E8%AF%86%E5%A4%A7%E5%85%A8](http://lib.csdn.net/qq_31810357/286174/chart/【Objective-C】基础知识大全)

