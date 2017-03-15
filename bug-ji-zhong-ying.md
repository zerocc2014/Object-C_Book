![](/assets/bug.png)

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



