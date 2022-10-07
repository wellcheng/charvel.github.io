---
layout: post
title: "AutoLayout 读书笔记(二)"
date: 2015-04-23 09:00:52 +0800

tags: [AutoLayout, 学习笔记]
---


## 约束

AutoLayout 是一种约束满足系统，约束的本质就是`限制`,布局中，创建的每一个规则都提出一个要求，这些要求规定了视图中一个部分与另一个部分之间的关系，这些规定具有优先级，AutoLayout 系统根据所有的规则，确定唯一的布局方式。

<!--more-->


### 约束类型

AutoLayout 的核心约束类

- 布局约束（NSLayoutConstraint 类，Public）
布局约束规则用来指定视图的几何特征，这些规则将视图与其他视图关联起来确定视图的位置和尺寸，要么直接指定其位置和尺寸为一个常数。

- 内容大小约束（NSContentSizeLayoutConstraint，Private）
内容大小规则，指定其视图尺寸与内容之间的关系，例如，内容吸附规则尽量避免添加补白，内容压缩规则阻止视图被剪裁。

- 自动尺寸调整约束 （NSAutoresizingMaskLayoutConstraint，Private）
自动尺寸调整约束，将原来的 Autoresizing 转换为 AutoLayout 系统中对应的约束。

- 布局支持系统 （ _UILayoutSupportConstraint，Private）
布局支持约束，是 iOS 7 中新增的约束，用来建立视图控制器实例顶部和底部的实际边界，它防止视图的内容与类似 statusBar 之类的障碍物发生重叠

- 原型约束（NSIBProtypingLayoutConstraint，Private）
iOS7 新增，它是 IB 为你添加的约束，在迭代构造界面的过程中，仍然保留一个用于测试的工作界面，在发布应用时，不能引用原型约束。


在开发和调试 AutoLayout 时，Xcode 调试信息会显示出这些类的实例。

不管是什么类，所有约束类型：

1. 表达视图在屏幕上的布局方式
2. 都有一个内在的优先级，指定每个请求在 AutoLayout 系统中的强烈程度。

### 优先级

优先级表示规则请求强烈程度的数字，AutoLayout 使用优先级解决某些约束冲突。范围：`1~1000`，数字越大，优先级越高。

iOS UIKit 提供了常用的优先级枚举值：

    enum {
        UILayoutPriorityRequired = 1000,    // 必须满足
        UILayoutPriorityDefaulthigh = 750,    // 压缩抵抗的默认优先级
        UILayoutPriorityDefaultLow = 500,    // 
        UILayoutPriorityFittingSizeLevel = 50,    // 
    }
    
一般建议创建自己的优先级约束枚举类型。


### 内容大小约束

iOS 中，每一个 View 都是由其在父视图中的位置 position 和尺寸 size 来确定其位置的。虽然我们之前经常会指定其大小，但是现在在多屏幕尺寸的开发中，更迫切的希望能够跟随内容动态的调整其位置和大小。


之前提过，label、imageView 和 其他 UIControl 的大小取决于其需要呈现的内容。
而不包含自然内容的视图，其内在内容大小为（-1，-1），UIKit 将这种无内容的尺寸声明为 UIViewNoIntrinsicMetric。Apple 指出：

> 并非所有的视图都有 intrinsicContentSize。UIView 的默认实现是返回 （intrinsicContentSize，intrinsicContentSize）。内在内容大小仅考虑视图自身中的数据，而不考虑其他视图中的数据。
> 记住，仍然可以针对任何视图设置其高度、宽度约束。如果不希望这些尺寸随内容而改变，那么你不需要重写 instrinsicContentSize。


内容吸附约束限制视图允许自身伸展和填充视图的程度。如果这个优先级比较高，那么视图的 frame 将与自身内容相匹配。

压缩阻力防止视图剪切其内容。高优先级可以确保视图显示出完整的内在内容。

可以通过代码为每一个视图指定内容吸附和压缩阻力优先级。
内容吸附优先级为 250（所以指定尺寸时，内容吸附会被忽略）。压缩阻力优先级默认为 750：

    - (void)setContentHuggingPriority:(UILayoutPriority)priority forAxis:(UILayoutConstraintAxis)axis
    - (UILayoutPriority)contentHuggingPriorityForAxis: (UILayoutConstraintAxis)axis
    - (void)setContentCompressionResistancePriority:(UILayoutPriority)priority forAxis:(UILayoutConstraintAxis)axis
    - (UILayoutPriority)contentCompressionResistancePriorityForAxis: (UILayoutConstraintAxis)axis

当然也可以通过 IB 设置内容大小约束。


### 构建布局约束
布局约束（NSLayoutConstraint 的实例）定义了视图的物理几何特征的规则，指定了视图的布局方式，以及视图与同一层级结构中其他视图的关系。

![自动布局约束元素表](http://7vzucb.com1.z0.glb.clouddn.com/blog.AutoLayout-elements.png)

### 布局约束类
数学规则通过构建 NSLayoutConstraint 类的实例构建，然后将创建的规则应用到视图中，该类的实例提供以下几个基本约束：

- priority：优先级
- firstItem 与 secondItem ，约束关联的视图
- firstAttribute 与 secondAttribute ：约束系统中的名词，描述视图对齐矩阵的特征，如左边、右边、中心、高度，为上表中 12 个枚举类型中的任意一个，不存在第二项，使用 NSLayoutAttributeNotAnAttribute 占位符
- relation：动词。指出属性之间如何比较
- mutiplier 和 constant：提供代数元素，增强约束的功能和灵活性，通过这个属性，可以指出一个视图是另一个视图的一半，也可以指定一个视图是通过另一个视图偏移某个距离得到的。

约束本质山就是数学中的相等或者不等关系，用公式表示就是：

    y （关系）m * x + b
    
例如，视图 B 的左边 = 视图 A 的右边 + 15，这个是相等关系

或者，视图 B 的左边 >= 视图 A 的右边 + 15，这个是不等关系

其中 m 是缩放因子，用来方法缩小视图，b 是偏移常数

一般来说，构建的关系式都涉及两个视图，即

    firstItem.firstAttribute (R) secondItemAttribute * m + b
    
firstItem 和 secondItem 的顺序可以任意，也可以表示为 [视图A]-10-[视图B]，从理解上来讲，应该是视图 B 右边距离视图 A 的右边 10pt，因为我们总是将左上的视图作为 firstItem，将右下的视图作为secondItem。

### 创建布局约束

构建布局约束有 3 种方法：

- IB 设计界面
- 可视化约束语言 ，允许 NSLayoutConstraint 根据请求生成实例
- 为每一个组件提供一个基本关系，从而构建 NSLayoutConstraint 的实例

从简易程度上讲，是从上到下的，并且从上到下也是 Apple 推荐的顺序，但是作为开发者，我认为还是从下往上比较好，因为在 IB 中创建约束后，你过后经常会忘记你当时想要的是什么布局，而且对于边界条件不能够很好的处理。

NSlayoutConstraint 的类方法：

    [NSLayoutConstraint constraintWithItem:
                                 attribute:
                                 relatedBy:
                                    toItem:
                                 attribute:
                                multiplier:
                                  constant:
     ]
     
每次只创建一个约束，每个布局约束定义一个规则，可能涉及一个或者两个视图。

如果涉及到两个视图，那么上述约束将会创建一个严格的 `视图.属性 R 视图.属性 * 因数 + 常数` 的关系，其中关系 R 可以是等于或不等关系。

举个🌰
    
    [self.view addConstraint:
    [NSLayoutConstraint constraintWithItem:self.view
                                 attribute:NSLayoutAttributeCenterX
                                 relatedBy:NSLayoutRelationEqual
                                    toItem:textField
                                 attribute:NSLayoutAttributeCenterX
                                multiplier:1
                                  constant:0
     ]];

该代码向视图控制器的视图（self.view）添加了一个约束，要求与 textField 输入框水平居中对齐。表达式如下：

`[self.view]的 centerX = （[文本字段]的 centerX * 1） + 0`

addConstraint 方法是将某个约束实例添加到视图中，类似与 IB 中 某个 View 下的 constraints

有一部分约束是只需一个视图的。例如指定宽度等。

约束至少需要与一个 View 有关系。

### 视图项

约束用 firstItem 和 secondItem 属性引用其所影响的视图，这些属性是只读的，只能在创建时期设置。可以拓展 NSLayoutConstraint 类（类别）来获取 NSLayoutConstraint 类实例的 firstItem 和 secondItem

### 约束、层次结构与边界系统

> 约束引用两个视图时，这两个视图必须在同一个视图层次结构，这个特别重要。
>
> 对于两个涉及视图的约束，只有两种情况，一种是父视图与子视图，另一种就是视图是平级的，即它们拥有相同的祖先视图。

### 安装约束

约束创建完成之后，还只是 NSLayoutConstraint 类的实例，如果想要约束生效，还需要将约束添加到 AutoLayout 系统，例如：


    [myView addConstraint: aConstraintInstance];
    
因为可视化格式系统返回的是约束实例数组，所以还提供另一种方式：

    [myView addConstraints: myArrayConstraints];
    
对于单个 item 的约束，安装到这个 item 对应 View 的本身，对于两个 Item，需要安装到它们最近的公共祖先视图上。

需要删除约束时，有一种办法是删除约束涉及的某个 view，然后 AutoLayout 系统会自动将所有涉及 view 的约束删除掉，即调用 [view removeFromSuperView];

还有一种就是 view 调用删除约束的方法，不过需要注意的是，这个时候传入的 constraint 实例必须是之前安装的实例，即两者的指针要相同。这个时候可能需要将之前安装的 constraint 实例保存后，才能正确的删除，不过也有一种办法就是搜索当前 View 中所有与希望删除约束等价的约束，然后将其删除。

因为，视图将约束按照对象存储，如果两个约束不在内存中的同一个位置，即使它们的规则一样，视图仍然认为它们是不同的。

### 比较约束

前面提到过，所有的约束都是一种关系：

`视图1.属性 （关系R）视图2.属性 * 因数 + 常数`

这些属性有 priority、firstItem、firstAttribute、relation、secondItem、secondAttribute、multiplier 和 constant。

对两个约束，可以分别比较它们的这些属性来判断是否为等价的约束。

约束安装后，可以删除，也可以用新的规则来替换。

AutoLayout 使用约束创建动画时，很大部分是修改约束部分的常数来实现。

### 布局约束法则

- 布局约束具有优先级
- 布局约束没有天然顺序，所有具有相同优先级的约束都被同时考虑
- 布局约束是关系，没有方向
- 布局约束可以取近似值
- 布局约束可以循环
- 布局约束可以冗余（只要不发生冲突）
- 布局约束可以引用兄弟视图
- AutoLayout 对变形的处理可能不是很好
- AutoLayout 不支持 iOS7 新增的视图动态功能
- AutoLayout 支持动画效果
- 布局约束不应在不同的边界系统之间交叉使用
- 布局约束会在运行时失败
- 格式不正确的约束会 crash
- 约束至少引用一个视图
- 小心无效的属性结对






