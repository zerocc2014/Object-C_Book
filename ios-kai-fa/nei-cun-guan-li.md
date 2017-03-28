## Objective-C 中的内存分配 {#objective-c-中的内存分配}

在 Objective-C 中，对象通常是使用`alloc`方法在堆上创建的。`[NSObject alloc]`方法会在对堆上分配一块内存，按照`NSObject`的内部结构填充这块儿内存区域。

一旦对象创建完成，就不可能再移动它了。因为很可能有很多指针都指向这个对象，这些指针并没有被追踪。因此没有办法在移动对象的位置之后更新全部的这些指针。

## OC内存管理的三种方式

1. 自动垃圾收集 GC \(Automatic Garbage Collection\)；
2. 手动引用计数器 MRC \(Manual Reference Counting\)和自动释放池；
3. 自动引用计数器 ARC \(Automatic Reference Counting\)。

## Autorelease Pool

**Autorelase Pool** 提供了一种可以允许你向一个对象**延迟发送**`release`**消息的机制**。当你想放弃一个对象的所有权，同时又不希望这个对象立即被释放掉（例如在一个方法中返回一个对象时），Autorelease Pool 的作用就显现出来了。

所谓的延迟发送`release`消息指的是，当我们把一个对象标记为`autorelease`时:

```
NSString *str = [[[NSString alloc] initWithString:@"hello"] autorelease];
```

这个对象的 retainCount 会+1，但是并不会发生 release。当这段语句所处的 autoreleasepool 进行 drain 操作时，所有标记了`autorelease`的对象的 retainCount 会被 -1。即`release`消息的发送被延迟到 pool 释放的时候了。

在 ARC 环境下，苹果引入了`@autoreleasepool`语法，不再需要手动调用`autorelease`和`drain`等方法。

#### Autorelease Pool 的用处 {#autorelease-pool-的用处}

在 ARC 下，我们并不需要手动调用 autorelease 有关的方法，甚至可以完全不知道 autorelease 的存在，就可以正确管理好内存。因为 Cocoa Touch 的 Runloop 中，每个 runloop circle 中系统都自动加入了 Autorelease Pool 的创建和释放。

当我们需要创建和销毁大量的对象时，使用手动创建的 autoreleasepool 可以有效的避免内存峰值的出现。因为如果不手动创建的话，外层系统创建的 pool 会在整个 runloop circle 结束之后才进行 drain，手动创建的话，会在 block 结束之后就进行 drain 操作。详情请参考[苹果官方文档](https://developer.apple.com/library/ios/documentation/Cocoa/Conceptual/MemoryMgmt/Articles/mmAutoreleasePools.html#//apple_ref/doc/uid/20000047-CJBFBEDI)。一个普遍被使用的例子如下：

```
for (int i = 0; i < 100000000; i++)
{
    @autoreleasepool {
        NSString* string = @"ab c";
        NSArray* array = [string componentsSeparatedByString:string];
    }
}
```

如果不使用 autoreleasepool ，需要在循环结束之后释放 100000000 个字符串，如果 使用的话，则会在每次循环结束的时候都进行 release 操作。

#### Autorelease Pool 进行 Drain 的时机

如上面所说，系统在 runloop 中创建的 autoreleaspool 会在 runloop 一个 event 结束时进行释放操作。我们手动创建的 autoreleasepool 会在 block 执行完成之后进行 drain 操作。需要注意的是：

* 当 block 以异常（exception）结束时，pool 不会被 drain
* Pool 的 drain 操作会把所有标记为 autorelease 的对象的引用计数减一，但是并不意味着这个对象一定会被释放掉，我们可以在 autorelease pool 中手动 retain 对象，以延长它的生命周期（在 MRC 中）。



