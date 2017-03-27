## 参考：

* \[雷纯峰技术博客之多线程\]\([http://blog.leichunfeng.com/blog/2015/07/29/ios-concurrency-programming-operation-queues/\](http://blog.leichunfeng.com/blog/2015/07/29/ios-concurrency-programming-operation-queues/%29\)
* \[iOS 多线程总概括\]\([http://www.cnblogs.com/GarveyCalvin/p/4206009.html\#NSThreed\](http://www.cnblogs.com/GarveyCalvin/p/4206009.html#NSThreed%29\)

# 线程与进程

**进程（process）：**指的是一个正在运行中的可执行文件。每一个进程都拥有独立的虚拟内存空间和系统资源，包括端口权限等，且至少包含一个主线程和任意数量的辅助线程。另外，当一个进程的主线程退出时，这个进程就结束了；

**线程（thread）**：指的是一个独立的代码执行路径，也就是说线程是代码执行路径的最小分支。在 iOS 中，线程的底层实现是基于 [POSIX threads API](https://developer.apple.com/library/content/documentation/Cocoa/Conceptual/Multithreading/Introduction/Introduction.html) 的。

**任务（task）：**指的是我们需要执行的工作，是一个抽象的概念，用通俗的话说，就是一段代码。

多线程缺点：当线程过多，会消耗大量的CPU资源，而且，每开一条线程也是需要耗费资源的（iOS主线程占用1M内存空间，子线程占用512KB）。

# 基本概念

#### 串行 vs. 并发：

从本质上来说，串行和并发的主要区别在于允许同时执行的任务数量。串行，指的是一次只能执行一个任务，必须等一个任务执行完成后才能执行下一个任务；并发，则指的是允许多个任务同时执行。

#### 同步 vs. 异步：

同样的，同步和异步操作的主要区别在于是否等待操作执行完成，亦即是否阻塞当前线程。同步操作会等待操作执行完成后再继续执行接下来的代码，而异步操作则恰好相反，它会在调用后立即返回，不会等待操作的执行结果。

#### 队列 vs. 线程：

有一些对 iOS 并发编程模型不太了解的同学可能会对队列和线程产生混淆，不清楚它们之间的区别与联系，因此，我觉得非常有必要在这里简单地介绍一下。在 iOS 中，有两种不同类型的队列，分别是串行队列和并发队列。正如我们上面所说的，串行队列一次只能执行一个任务，而并发队列则可以允许多个任务同时执行。iOS 系统就是使用这些队列来进行任务调度的，它会根据调度任务的需要和系统当前的负载情况动态地创建和销毁线程，而不需要我们手动地管理。

# iOS多线程

## NSThread

NSThread创建线程一般有三种方式：

```
// equivalent to the first method with kCFRunLoopCommonModes
- (void)performSelectorInBackground:(SEL)aSelector withObject:(nullable id)arg;
+ (void)detachNewThreadSelector:(SEL)selector toTarget:(id)target withObject:(nullable id)argument;
- (instancetype)initWithTarget:(id)target selector:(SEL)selector object:(nullable id)argument
```

* 前两种创建之后会自动执行，第三种方式创建后需要手动执行；
* 第一种

示例代码：

```
- (void)createThread{
    // 创建线程
    NSThread *thread = [[NSThread alloc] initWithTarget:self selector:@selector(run:) object:@"我是参数"];
    thread.name = @"我是线程名字啊";
    // 启动线程
    [thread start];

    // 或者 [NSThread detachNewThreadSelector:@selector(run:) toTarget:self withObject:@"我是参数"];
    // 或者 [self performSelectorInBackground:@selector(run:) withObject:@"我是参数"];
}
- (void)run:(NSString *)param{
    NSLog(@"-----run-----%@--%@", param, [NSThread currentThread]);
}
```

控制台输出：

```
-----run-----我是参数--<NSThread: 0x7ff8a2f0c940>{number = 2, name = 我是线程名字啊}
```

## NSOperation

使用NSOperation和NSOperationQueue进行多线程开发类似于C\#中的线程池，只要将一个NSOperation（实际开中需要使用其子类NSInvocationOperation、NSBlockOperation）放到NSOperationQueue这个队列中线程就会依次启动。NSOperationQueue负责管理、执行所有的NSOperation，在这个过程中可以更加容易的管理线程总数和控制线程之间的依赖关系。

NSOperation有两个常用子类用于创建线程操作：NSInvocationOperation和NSBlockOperation，两种方式本质没有区别，但是是后者使用Block形式进行代码组织，使用相对方便。

#### NSInvocationOperation

首先使用NSInvocationOperation进行一张图片的加载演示，整个过程就是：创建一个操作，在这个操作中指定调用方法和参数，然后加入到操作队列。其他代码基本不用修改，直接修加载图片方法如下：

示例代码：

```
-(void)loadImageWithMultiThread{
    /*创建一个调用操作
     object:调用方法参数
    */
    NSInvocationOperation *invocationOperation=[[NSInvocationOperation alloc]initWithTarget:self selector:@selector(loadImage) object:nil];
    //创建完NSInvocationOperation对象并不会调用，它由一个start方法启动操作，但是注意如果直接调用start方法，则此操作会在主线程中调用，一般不会这么操作,而是添加到NSOperationQueue中
//    [invocationOperation start];
    
    //创建操作队列
    NSOperationQueue *operationQueue=[[NSOperationQueue alloc]init];
    //注意添加到操作队后，队列会开启一个线程执行此操作
    [operationQueue addOperation:invocationOperation];
}
```

#### NSBlockOperation

下面采用NSBlockOperation创建多个线程加载图片。

示例代码：

```
#pragma mark 多线程下载图片
-(void)loadImageWithMultiThread{
    int count=ROW_COUNT*COLUMN_COUNT;
    //创建操作队列
    NSOperationQueue *operationQueue=[[NSOperationQueue alloc]init];
    operationQueue.maxConcurrentOperationCount=5;//设置最大并发线程数
    //创建多个线程用于填充图片
    for (int i=0; i<count; ++i) {
        //方法1：创建操作块添加到队列
//        //创建多线程操作
//        NSBlockOperation *blockOperation=[NSBlockOperation blockOperationWithBlock:^{
//            [self loadImage:[NSNumber numberWithInt:i]];
//        }];
//        //创建操作队列
//
//        [operationQueue addOperation:blockOperation];
        
        //方法2：直接使用操队列添加操作
        [operationQueue addOperationWithBlock:^{
            [self loadImage:[NSNumber numberWithInt:i]];
        }];
    }
}
```

之前提到过多线程并发队列可以设置最大并发数，以及队列的取消、暂停、恢复操

```
// 创建队列
NSOperationQueue *queue = [[NSOperationQueue alloc] init];

// 设置最大并发操作数
queue.maxConcurrentOperationCount = 2; // 并发队列
queue.maxConcurrentOperationCount = 1; // 串行队列

 // 恢复队列，继续执行
queue.suspended = NO;
// 暂停（挂起）队列，暂停执行
queue.suspended = YES;

// 取消队列
[queue cancelAllOperations];
```

## GCD

#### dispatch\_queue\_t   获取/创建队列

GCD 的队列有两种：

* Serial Dispatch Queue   等待现在执行中处理结束（串行队列）
* Concurrent Dispatch Queue   不等待现在执行中处理结束（并行队列）

GCD中的队列都是 **dispatch\_queue\_t **类型，获取/创建方法：

```
// 1. 手动创建队列
dispatch_queue_t dispatch_queue_create(const char *label, dispatch_queue_attr_t attr);
 // 1.1 创建串行队列
    dispatch_queue_t queue = dispatch_queue_create("com.sanyucz.queue", DISPATCH_QUEUE_SERIAL);
 // 1.2 创建并行队列
    dispatch_queue_t queue = dispatch_queue_create("com.sanyucz.queue", DISPATCH_QUEUE_CONCURRENT);

// 2. 获取系统标准提供的 Dispatch Queue
 // 2.1 获取主队列
    dispatch_queue_t queue = dispatch_get_main_queue();
 // 2.2 获取全局并发队列
    dispatch_queue_t queue = dispatch_get_global_queue(DISPATCH_QUEUE_PRIORITY_DEFAULT, 0);
```

需要说明的是，手动创建队列时候的两个关键参数，**const char \*label **指定队列名称，最好起一个有意义的名字，当然如果你想调试的时候刺激一下，也可以设置为 **NULL**，而 **dispatch\_queue\_attr\_t attr **参数文档有说明：

```
/*! 创建串行队列按顺序FIFO（First-In-First-On）先进先出；
 * @const DISPATCH_QUEUE_SERIAL
 * @discussion A dispatch queue that invokes blocks serially in FIFO order. 
 */ 
 #define DISPATCH_QUEUE_SERIAL NULL

/*! 则会创建并发队列
 * @const DISPATCH_QUEUE_CONCURRENT
 * @discussion A dispatch queue that may invoke blocks concurrently and supports
 * barrier blocks submitted with the dispatch barrier API.
 */
#define DISPATCH_QUEUE_CONCURRENT \
        DISPATCH_GLOBAL_OBJECT(dispatch_queue_attr_t, \
        _dispatch_queue_attr_concurrent)
```

#### dispatch\_async/dispatch\_sync 创建任务

创建完队列之后就是定义任务了，理解为任务在那个队列以何种方式（串行或者并行）执行，有两种方式：

```
// 创建一个同步执行任务
void dispatch_sync(dispatch_queue_t queue, dispatch_block_t block);
// 创建一个异步执行任务
void dispatch_async(dispatch_queue_t queue, dispatch_block_t block);
```

示例代码：

```
dispatch_queue_t queue = dispatch_queue_create("com.sanyucz.queue.asyncSerial", DISPATCH_QUEUE_SERIAL);
dispatch_async(queue, ^{
   NSLog(@"异步 + 串行 - %@",[NSThread currentThread]);
});
```

#### dispatch group 任务组

我们可能在实际开发中会遇到这样的需求：在两个任务完成后再执行某一任务。虽然这种情况可以用串行队列来解决，但是我们有更加高效的方法。

示例代码：

```
// 获取全局并发队列
dispatch_queue_t queue = dispatch_get_global_queue(DISPATCH_QUEUE_PRIORITY_DEFAULT, 0);
// 创建任务组
// dispatch_group_t ：A group of blocks submitted to queues for asynchronous invocation
dispatch_group_t group = dispatch_group_create();
// 在任务组中添加一个任务
dispatch_group_async(group, queue, ^{

});
// 在任务组中添加另一个任务
dispatch_group_async(group, queue, ^{

});
// 当任务组中的任务执行完毕之后再执行一下任务
dispatch_group_notify(group, queue, ^{

});
```



#### dispatch\_barrier\_async

从字面意思就可以看出来这个变量的用处，即阻碍任务执行，它并不是阻碍某一个任务的执行，而是在代码中，在它之前定义的任务会比它先执行，在它之后定义的任务则会在它执行完之后在开始执行。就像一个栏栅。

示例代码：

```
dispatch_queue_t queue = dispatch_queue_create("com.gcd.barrier", DISPATCH_QUEUE_CONCURRENT);
dispatch_async(queue, ^{
   NSLog(@"----1-----%@", [NSThread currentThread]);
});
dispatch_async(queue, ^{
   NSLog(@"----2-----%@", [NSThread currentThread]);
});
dispatch_barrier_async(queue, ^{
   NSLog(@"----barrier-----%@", [NSThread currentThread]);
}); 
dispatch_async(queue, ^{
   NSLog(@"----3-----%@", [NSThread currentThread]);
});
dispatch_async(queue, ^{
   NSLog(@"----4-----%@", [NSThread currentThread]);
});
```

控制台输出：

```
----1-----<NSThread: 0x7fdc60c0fd90>{number = 2, name = (null)}
----2-----<NSThread: 0x7fdc60c11500>{number = 3, name = (null)}
----barrier-----<NSThread: 0x7fdc60c11500>{number = 3, name = (null)}
----3-----<NSThread: 0x7fdc60c11500>{number = 3, name = (null)}
----4-----<NSThread: 0x7fdc60c0fd90>{number = 2, name = (null)}
```

#### dispatch\_apply 遍历执行任务

**dispatch\_apply **的用法类似于对数组元素进行 **for循环 **遍历，但是 **dispatch\_apply **的遍历是无序的。

```
dispatch_queue_t queue = dispatch_get_global_queue(DISPATCH_QUEUE_PRIORITY_DEFAULT, 0);

NSMutableArray *array = [NSMutableArray array];
for (int i = 0; i < 10; i++) {
   [array addObject:@(i)];
}
// array = [0, 1, 2, 3, 4, 5, 6, 7, 8, 9]
NSLog(@"--------apply begin--------");
dispatch_apply(array.count, queue, ^(size_t index) {
   NSLog(@"%@---%zu", [NSThread currentThread], index);
});
NSLog(@"--------apply done --------");
```

可以无序并发执行多个任务，但是有一点可以确定，就是 **NSLog\(@"--------apply done --------"\);**这段代码一定是在所有任务执行完之后才会去执行。

#### GCD 的其他用法

* dispatch\_after 延期执行任务
* dispatch\_suspend / dispatch\_resume 暂停/恢复某一任务
* dispatch\_once 保证代码只执行一次，而且线程安全
* Dispatch I/O 可以以更小的粒度读写文件



# 线程安全

能被多个线程同时执行的资源

#### 资源竞争\(race condition\)：

---

![](/assets/race condition.png)

如图，有一个变量存储着17，然后线程A读取变量得到17，线程B也读取了这个变量。两个线程都进行了加1运算得，并写入18都变量。从而导致了崩溃。这就是race condition,多线程使用共享资源，而没有确保其它线程是已经结束使用共享资源。

#### 互相排除\(Mutual Exclusion\)

---

![](/assets/Mutual.png)

---

如图，使用锁对线程正在使用的共享资源进行锁定，当共享资源使用完后。其它线程才能使用这个共享变量。这样就能避免race condition但会导致死锁。

#### 死锁\(Deadlock\)：

---

![](/assets/deadLock.png)

---

两个线程等待着彼此的完成而陷入的困境称为死锁。

示例代码：

```
void swap(A, B)
{
    lock(lockA);
    lock(lockB);
    int a = A;
    int b = B;
    A = b;
    B = a;
    unlock(lockB);
    unlock(lockA);
}

swap(X, Y); // 线程 thread 1
swap(Y, X); // 线程 thread 2
```

结果就会导致X被线程1锁住，Y被线程2锁住。而线程1又不能使用Y直到线程2解锁，同理，线程2也不能使用X。这就是死锁，互相等待。













































