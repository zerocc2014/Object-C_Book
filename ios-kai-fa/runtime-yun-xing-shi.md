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

## OC如何实现动态调用

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

