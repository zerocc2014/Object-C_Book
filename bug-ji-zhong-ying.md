不要将拷贝用到NSMutableString，NSMutableArray，NSMutableDictionary等可变对象上，除非有特别的需求。

```
@interface ViewController ()

@property (nonatomic, copy) NSMutableArray *mutableArray_copy;

@end

@implementation ViewController

- (void)viewDidLoad {
    [super viewDidLoad];

    NSMutableArray *mutableArray = [NSMutableArray arrayWithObject:@"哈哈"];
    self.mutableArray_copy = mutableArray;

    [self.mutableArray_copy addObject:@"呵呵"];
}
```

![](/assets/bug.png)

仔细观察错误，`[__NSArrayI addObject:],`\_\_NSArrayI表示的是NSArray类型，NSArray是不可变数组，当然没有addObject：这个方法。明明self.mutableArray\_copy是可变类型，为什么变成了不可变类型。 执行这`self.mutableArray_copy = mutableArray;`一句的时候，会调用mutableArray\_copy的set方法，复制属性默认set方法是这样写的：

```
- (void)setMutableArray_copy:(NSMutableArray *)mutableArray_copy {
    _mutableArray_copy = [mutableArray_copy copy];
}
```

解决办法：如果确实有副本的需求，重写set方法，将copy改为mutableCopy即可;

---

* 


