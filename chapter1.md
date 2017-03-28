# First Chapter

#### 每日计划：20号：runtime  数据结构：5页；

---

gibook 使用：[http://blog.csdn.net/xiaocainiaoshangxiao/article/details/46882921](http://blog.csdn.net/xiaocainiaoshangxiao/article/details/46882921)

[https://github.com/ChenYilong](https://github.com/ChenYilong)

[cocopods](http://www.jianshu.com/p/3086df14ed08\)

[oc文件编译过程](http://ios.jobbole.com/92364/\)

[所有博客地址](http://www.jianshu.com/p/c47c24ab1e76)

[Xcode8继续使用插件](http://www.cnblogs.com/jys509/p/6290416.html\)

```
clang -rewrite-objc hello.m
```

```
clang -x objective-c -rewrite-objc -isysroot /Applications/Xcode.app/Contents/Developer/Platforms/iPhoneSimulator.platform/Developer/SDKs/iPhoneSimulator10.1.sdk XXXXXXX
```

video 插件

```
take a look at this video:
```

```
1. {% video %}https://www.youtube.com/watch?v=Oru-qw-Faac{% endvideo %}

2. {% video %}https://vimeo.com/128858567{% endvideo %}
```

```
"video-player"
{% videoplayerscripts %}{% endvideoplayerscripts %}
{% videoplayer id="myvideo" width="640" height="480" posterExt="png" %}https://s3.amazonaws.com/gitbooks/myvi
```

文字加底色

```
deo{% endvideoplayer %}
```

```
This text is {% em %}highlighted !{% endem %}

This text is {% em %}highlighted with **markdown**!{% endem %}

This text is {% em type="green" %}highlighted in green!{% endem %}

This text is {% em type="red" %}highlighted in red!{% endem %}

This text is {% em color="#ff0000" %}highlighted with a custom color!{% endem %}
```

语法高亮

```
{%ace edit=true, lang='c_cpp'%}
// This is a hello world program for C.
#include <stdio.h>

int main(){
  printf("Hello World!");
  return 1;
}
{%endace%}
```

"ace":{  
"languages": \[  
{  
"edit":"true",  
"lang": "mode-objectivec.js",  
"check":"true",  
"theme":"theme-xcode.js"  
}  
\]  
}

[https://plugins.gitbook.com/plugin/ace](https://plugins.gitbook.com/plugin/ace)

