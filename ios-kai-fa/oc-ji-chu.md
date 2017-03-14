> ```
> #import <UIKit/UIKit.h>
>
> @interface GXClientLabelView : UIView
>
> @property (nonatomic, strong) NSArray *LabelArry;
>
> @property (nonatomic, assign) float labelViewheight;
>
> /**
>  *  标签文本赋值
>  *
>  *  @param array 传入的数组（标签string）
>  */
> - (void)setClientLabelWithArray:(NSArray *)array setHight:(UIView *)view;
>
> - (CGFloat)clientLabelView:(GXClientLabelView *)clientLabelView viewHeightWithArray:(NSArray *)array;
>
> @end
> ```



