```
    typedef struct objc_class *Class;
    struct objc_class {
        Class isa;  //指向对象类型
        Class super_class; //指向父类的
        const char *name; //类名
        long version; //类的版本信息,默认为0
        long info; //供运行期使用的一些位标识。
        long instance_size; //类的实例变量大小
        struct objc_ivar_list *ivars; //成员变量的数组
        struct objc_method_list **methodList; //方法定义的数组，注意这里是“**”
        struct objc_cache; //指向最近使用的方法，用于方法调用的优化.
        struct objc_protocol_list *protocols; //协议的数组
    }
```

```
#import <UIKit/UIKit.h>

@interface GXClientLabelView : UIView

@property (nonatomic, strong) NSArray *LabelArry;

@property (nonatomic, assign) float labelViewheight;

/**
 *  标签文本赋值
 *
 *  @param array 传入的数组（标签string）
 */
- (void)setClientLabelWithArray:(NSArray *)array setHight:(UIView *)view;

- (CGFloat)clientLabelView:(GXClientLabelView *)clientLabelView viewHeightWithArray:(NSArray *)array;

@en
```

```
#pragma mark - View Lifecycle
- (void)viewDidLoad {
    [super viewDidLoad];
    
    [self initUI];
}

```



