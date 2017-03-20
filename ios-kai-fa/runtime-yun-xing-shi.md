[http://www.cocoachina.com/ios/20151026/13908.html](http://www.cocoachina.com/ios/20151026/13908.html)

[http://blog.csdn.net/sike2008/article/details/50164961](http://blog.csdn.net/sike2008/article/details/50164961 "runtime 实现万能跳转")

1. 如何动态调用

runtime简称运行时。也就是系统运行的时候的一些机制，主要是消息机制。函数的调用在编译的时候就已经决定是具体哪个函数，编译完再按顺序执行。OC语言之所以称之为动态运行时语言，主要表现在对象类型在运行时才真正确定，并不是在编译阶段，Objective-C是基于C语言加入了面向对象特性和消息转发机制的动态语言，这意味着它不仅需要一个编译器，还需要Runtime系统来动态创建类和对象，进行消息发送和转发。。OC的函数调用称之为消息发送，是一个动态调用过程，编译的时候并不能决定真正调用哪个函数（只是一个方法名，在编译阶段可以调用任何函数，即使是这个函数并未实现，只要是声明过的就不会报错。而c语言在编译阶段就会报错），只要在真正运行的时候才会根据函数名找到对应函数来调用。





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

