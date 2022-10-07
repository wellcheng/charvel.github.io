---
layout: post
title: "AutoLayout 读书笔记(三)-可视化格式"
date: 2015-04-30 14:22:42 +0800
---

AutoLayout 目前有三种创建约束的方式，第一种为 IB 中手动创建、第二种为代码中使用 NSLayoutConstraint 类，现在我们学习第三种，使用 `格式化语言` 创建约束。

需要记住的是，不管何种方式创建的约束，都是 NSLayoutConstraint 类的实例，并且遵循 `y = mx + b` 关系

## 可视化约束的格式介绍

NSLayoutConstraint 类方法创建可视化的约束，尽管可视化约束的内容能够关联多个视图，但是它们会被 NSLayoutConstraint 翻译为每次只关联一个或多个视图。这种方式创建的约束为数组的形式。

<!--more-->



可视化格式，由一个描述布局方式的文本字符串组成。根据约束关联项在视图中出现的顺序一次列出它们。文本序列指定间隔、不等量和优先级。看代码比较直接：

    [self.view addConstraints:[NSLayoutConstraint constraintWithVisualFormat:@"V:[view1]-8-[view2]"]]
        options:NSLayoutFormatAlignAllLeading
        metrics:nil
        views:[NSDictionaryOfVariableBings(view1,view2)]
    ];
    
上面代码创建的约束为：在 View1 和 View2组成的左对齐一列，间距为 8 ，下面是几点说明：

- 坐标轴：作为前缀指定，H-水平，V-垂直，如果不指定，默认为 H，但是强烈建议显示指定
- 每个视图的变量名出现在方括号中，包起来
- 视图出现的顺序，与布局系统中请求的顺序匹配，一般为从左到右
- 两个视图的间隔以固定数字出现
- 选型参数指定对齐方式
- metrics 没用到
- views 表示真实对象与布局格式中 View 的绑定关系，一般使用语义化的词语来绑定真实的对象

为什么要使用这种方式呢，首先，简洁，使用这种方式创建的约束可能需要之前实例化 NSLayoutConstraint 方式好几个才行。第二是高度抽象，即当前表示的布局方式是基于一个比较高的角度去考虑。第三，很容易调整。

## 选项
可视化格式方法的选项，包括 对齐掩码、格式方向掩码

### 对齐
应当总是正交地使用掩码，比如在水平方向上创建了一行 view ，那么需要顶部对齐，基准线对齐，或者中心对齐

有些地方可以省略，例如给 options 传入 0，这个时候，只会创建可视化的约束，不建立对齐约束

### 变量绑定
将布局系统中表示视图的字符串与实际的对象绑定起来，使用 NSDictionaryOfVariableBindings（）宏来创建绑定。

你也可以自己来创建字典。但是字典对于视图数组很不友好，会在解析时出现问题。因为可视化格式对于一些步骤比较长的视图解析起来有些力不从心，例如 [self.view viewWithTag:100] 等。

为了解决上面的问题，可以使用代码创建字典和格式字符串，上代码：

    // 初始化格式字符串和绑定字典  
    NSMutableString *formatString = [NSMutableString string];  
    NSMutableDictionary *bindings = [NSMutableDictionary dictionary]; 
    
    // 视图数量 int i = 1; 
    // 创建一行视图，Build a format string that lays out the views in a row 
    // e.g. @"H:|-[view1] [view2]-[view3]..." 
    [formatString appendString:@"H:|-"]; 
    for (UIView *view in views) { 
        // 创建一个视图的名字 
        NSString *viewName = [NSString stringWithFormat:@"view%0d", i++]; 
        
        // 将这个视图添加到格式化字符串中 
        [formatString appendFormat:@"[%@]%@",viewName, (i <= views.count) ? @"-" : @""]; 
        
        // 将视图的名称与对象做绑定 
        bindings[viewName] = view; 
    }

### 度量
如果一开始不知道某个常量的值，可以使用度量字典提供的值。例如

    @"V:[view1]-spacing-[view2]"

以后知道了值时，创建一个字典对象将其关联起来，传递给 metrics：

    NSDictionary *metrics = @{@"spacing":@10};
    [NSLayoutConstraint constraintsWithVisualFormat:@"V:[view1]-spacing-[view2]"
        options: ...
        metrics:metrics
        views:bindings
    ]


在实际开发中，这个很有用，可以写一些可以复用的代码：

    - (void)constrainView:(UIView *)view toWidth:(CGFloat) width {
        NSString *formatString = @"H:[view(==width)]";    // 创建约束字符串
        NSDictionary *bindings = NSDictionaryOfVariableBindings(view);
        NSDictionary *metrics = @{@"width":@(width)};
        
        NSArray *constraints = [NSLayoutConstraint constraintsWithVisualFormat ...
        
        [view addConstraints:constraints];
    }

### 格式字符串结构
前面我们已经接触了很多次格式化字符串的结构，大概也能猜到一些，具体的语法为：

    (<orientation>:)? 
    (<superView><connection>)? 
    <view>(<conection><view>)* 
    (<connection><superview>)?
    
看起来很复杂，其实用到后面就会觉得简单了，`？`表示可选，`*` 表示0个或多个

### 方向
可视化格式，以一个方向开始，可以是 H：水平方向，V：垂直方向

### 视图名称
视图名称被方括号包围，使用变量绑定视图名时，视图名称指的是视图本身的变量名

使用竖线 `|` 来表示父视图，一般出现在格式字符串的开头或者结尾，开头时，一般出现在 `H:| ...`，结尾时，一般为最后一个引号之前

下面是几个典型的例子：

- 使当前视图按照父视图延伸（水平或者垂直）：`"H:|[view]|"`
- 某个视图偏移父视图的某条边：`"V:[view]-10-|"`,表示距离父视图底部 10 间隔
- 创建一行或者一列与父视图对齐的视图：`"V:|-[view1]-[view2]-[view3]-|"` 

### 连接
连接表示了视图之间的间隔

#### 空连接
一般是 `[view1][view2]` 这种形式，表示紧挨着

#### 标准间隔
这个时候使用连字符 `-` 代表固定的间隔，例如：

    "H:|-[view1]-[view2]"
    
表示 view1 和 view2 水平方向上标准的间隔，视图与视图之间为 8 ，子视图与父视图之间为 20

#### 数字间隔
可以在两个连字符中间插入数字表示准确的间隔值，如 `[view1]-30-[view2]` 。

#### 引用父视图
使用 `|` 表示父视图，例如 `"H:|[view1]-[view2]|"` 表示 view1 和 view2 之间有标准间隙，并且view1 的左边与父视图左边界相邻，view2 右边与父视图右边相邻，必须要制定 view1 或 view2 其中一个的宽度，不要会发生歧义。

#### 与父视图的间隔
加入连字符或者带数字的连字符即可，例如 `"H:|-[view1]-[view2]-30-|"`

#### 可变的间隔
有的时候，我们需要在两个视图之间加入一个可变的间隔。例如 `"H:|-[view1]-(>=0)-[view2]-|"` 表示，view1 和 view2 保持自己的尺寸，这个间隔最少为0 ，可以拉伸。

#### 圆括号
上面的例子中，大于等于关系包含在圆括号中，主要是有些数值可能带有负号，防止与连线符混淆。

对于同一种布局方式，有很多表示。例如表示两个视图相邻：

- [view1][view2]
- [view1]-0-[view2]
- [view1]-(0)-[view2]
- [view1]-(==0)-[view2]
- [view1]-(>=0,<=0)-[view2]
- [view1]-(==0@1000)-[view2]
- [view1]-(>=0,<=0,==0,<=30)-[view2]

关系后跟 `@` 表示优先级，多个关系可以用逗号隔开

#### 负数
为任何使用负数值的间隔加上括号，负数还有其他用途：`V:[view1]-(-10)-[view2]` 表示view1 下边缘往上10点为 view2 上边缘，这样子 view1 view2 垂直方向上有 10 点的重合。

#### 优先级
在数值或者关系上使用@添加优先级，一般为清晰起见，加上括号比较好。

#### 多视图
可以在格式字符串中插入多个视图

### 视图尺寸
方括号中除了表示视图名称，还可以指定尺寸，例如 `H:[view(100)]` 表示宽度 100，同样，也可以使用关系。

视图尺寸也可以加优先级，或者是某个视图的倍数

### 格式字符串部件
如下表总结，可以使用逗号将其组合起来：

![Visual Format Strings-1](http://7vzucb.com1.z0.glb.clouddn.com/blog.VisualFormatStrings-1.png)
![Visual Format Strings-2](http://7vzucb.com1.z0.glb.clouddn.com/blog.VisualFormatStrings-2.png)


> ios7 和 Xcode5 引入的 TopLayoutGuide 和 bottomLayoutGuide 需要先设置为本地变量然后再引用

### 错误
Xcode 无法提供编译器的检查，所以需要你注意控制台中的错误提示。

### NSLog 和可视化格式
有时候 Xcode 能够输出约束的可视化格式，有时候不行

## 约束到父视图
可以使用代码方便的为父视图添加可视化字符串式的约束。
可以在 ViewWillDidAppear 中打印出所有 view 的 frame

    void constrainToSuperview(UIView *view, float minimumSize, NSUInteger priority) {
        if (!view || !view.superview) { 
            return; 
        } 
        
        for (NSString *format in @[  
            @"H:|->=0@priority-[view(==minimumSize@priority)]",  
            @"H:[view]->=0@priority-|", 
            @"V:|->=0@priority-[view(==minimumSize@priority)]", 
            @"V:[view]->=0@priority-|"])  
        { 
            NSArray *constraints = [NSLayoutConstraint 
                constraintsWithVisualFormat:format options:0 
                metrics: @{@"priority":@(priority), 
                @"minimumSize":@(minimumSize)} 
                views:@{@"view": view}]; 
                [view.superview addConstraints:constraints];  
            } 
        }

## 视图拉伸

将视图向它们的父视图拉伸

    void stretchToSuperview(VIEW_CLASS *view, CGFloat indent, NSUInteger priority) {        
        for (NSString *format in @[
                @"H:|-indent-[view]-indent-|",
                @"V:|-indent-[view]-indent-|" ]) {
            NSArray *constraints = [NSLayoutConstraint
                    constraintsWithVisualFormat:format 
                    options:0 metrics:@{@"indent":@(indent)}
                    views:@{@"view": view}];
            for (NSLayoutConstraint *constraint in constraints) {
                constraint.priority = priority;
                [view.superview addConstraint:constraint];
            }
        }
    }

## 约束尺寸
将视图固定为制定的尺寸值

    void constrainViewSize(VIEW_CLASS *view, CGSize size, NSUInteger priority) {
        NSDictionary *bindings = NSDictionaryOfVariableBindings(view); 
        NSDictionary *metrics = @{
            @"width":@(size.width),
            @"height":@(size.height),
            @"priority":@(priority)
        };
        for (NSString *formatString in @[ 
            @"H:[view(==width@priority)]", 
            @"V:[view(==height@priority)]", ]) {
            NSArray *constraints = [NSLayoutConstraint
                constraintsWithVisualFormat:formatString
                options:0 metrics:metrics views:bindings];
            [view addConstraints:constraints];
        }
    }

## 创建一行或者一列
创建视图，排成一行或者一列：

    #define IS_HORIZONTAL_ALIGNMENT(ALIGNMENT) 
    \[@[@(NSLayoutFormatAlignAllLeft), @(NSLayoutFormatAlignAllRight),
    \@(NSLayoutFormatAlignAllLeading), @(NSLayoutFormatAlignAllTrailing),
    \@(NSLayoutFormatAlignAllCenterX)] containsObject:@(ALIGNMENT)]    
    
    void buildLineWithSpacing(NSArray *views, NSLayoutFormatOptions alignment, NSString *spacing, NSUInteger priority)
    {
        if (views.count == 0)
            return;
        
        VIEW_CLASS *view1, *view2;
        // Calculate the axis and its string representation. 
        // The axis is orthogonal to the requested alignment 
        // eg, centerX alignment creates a column, and
        // trailing builds a row.
        BOOL axisIsH = IS_HORIZONTAL_ALIGNMENT(alignment);
        NSString *axisString = (axisIsH) ? @"H:" : @"V:";
        // Build the format
        NSString *format = [NSString
            stringWithFormat:@"%@[view1]%@[view2]", axisString, spacing];
        // Apply the format to view pairs
        for (int i = 1; i < views.count; i++) {
            view1 = views[i-1];
            view2 = views[i];
            NSDictionary *bindings = NSDictionaryOfVariableBindings(view1, view2);
            NSArray *constraints = [NSLayoutConstraint 
            constraintsWithVisualFormat:format options:alignment
                metrics:nil views:bindings];
            for (NSLayoutConstraint *constraint in constraints)
                [constraint install:priority];
        }
    }





## 匹配尺寸
传入一个视图，然后将其他所有视图与第一个匹配

## 为什么不能够分布视图
可能很多时候，你希望沿着某个方向等间隔的分布视图，这个无法做到的原因就是，约束同时最多引用两个视图。
这个时候，有两种 hack 的方式，一种是先将总宽度分为 N 个区域，然后每个区域中心点放置一个视图。但是如果视图之间的尺寸不一致时，效果会大打折扣。还有一种方法就是用透明的 View 来充当间隔。



















