title: ScrollView 中使用 autoLayout 的坑与填坑
date: 2015-11-13 14:00:25
categories: 
- iOS
tags: [ScrollView, autoLayout]
---

>   ScrollView 是 UIKit 中很重要的一个组件，TableView 和 UIWebView 等很多涉及到滑动的视图都有它的影子，在之前使用代码布局的时代，只要简单的进行配置，就 OK 了。但是在 AutoLayout 的时代，ScrollView 就存在了很多坑，今天就是分析坑和填坑的。


<!--more-->


## 矛盾从哪儿来

先上地址是一个好习惯：

[Github 地址](https://github.com/wellcheng/UIScrollView-AutoLayout)

​

可以 git rest 到对应的阶段去看约束的设置。

好比下面举个例子，先在最上面放置了一个 imageView ，然后设置其约束:

``` objective-c
// 设置其水平居中
imageView.centerX = superView.center.X

// 然后确定其顶部间距
imageView.top = Top Layout Guide.bottom

// 设置其宽度和高度
imageView.height = 155 
imageView.width = 115

// 在这里说明下，auto layout 中，对于 label、imageView 等其中有内容的视图，有一个叫做固有内容的概念。对于通常的 UIView ，如果你想要使其约束达到完整。首先需要确定它的位置，也就是根据约束能够推断出 x 和 y 坐标，还得确定其 height 和 width

// 但如果这个视图是有固定内容的，那么就可以不主动的去设置其 height 或 width，因为 autoLayout 系统能够通过视图中的内容推断出视图当前的 height 和 width

// 比如，Label，我们不去主动设置 label 的高度，autoLayout 可以根据 label 中字体的大小推断得到其高度。比如 imageView ，autoLayout 可以根据其中 image 的尺寸得到其 height 和 width

// 关于固有内容还有一些压缩和扩张的优先级，这里就不做介绍了。
```





![给图片添加约束](http://7vzucb.com1.z0.glb.clouddn.com/blogscrollview-1.png)



然后拖一个 ScrollView 进去，设置 scrollView 的约束：

``` objective-c
// 顶部距离图片 20 
scrollView.top = image.top + 20

// 左边、右边以及下边都是 20 的距离
scrollView.leading = superView.leading + 20
scrollView.trailing + 20 = superView.trailing + 20
scrollView.bottom + 20 = Botton Layout Guide.top
```

看一下 scrollView 的约束：

![scrollView 的约束](http://7vzucb.com1.z0.glb.clouddn.com/blog.scrollview-2.png)



这样子 scrollView 的约束就确定了。build 一把，也没啥问题。

然后往 scrollView 中拖几个 label 、button 等进去，按照平时的习惯设置下约束，哎，好像出问题了。不对啊，我平常不都是这么弄的么，难道 scrollView 还不太一样吗？



![设置 textField 的约束](http://7vzucb.com1.z0.glb.clouddn.com/blog.scrollview-3.png)



看到约束变红了，表示约束当前出现了问题。对于调试约束有经验的同学，知道这个时候点击左上角的红色箭头，去看一下，然后发现出了个叫 `ScrollView content size ambiguity` 的问题。具体说的是 scrollView 的 content height 和 content width 有歧义。



### ScrollView content size ambiguity

字面理解，就是 scrollView 的 contentSize 布局有歧义。

contentSize 是 scrollView 中的内容尺寸，scrollView 需要知道自己的内容尺寸从而决定该怎么滑动。

将鼠标移动到黑色粗体的说明上，右边会显示出一个详细按钮，点击就能看到具体的说明，可以帮助我们快速了解 autoLayout 布局中的问题。



![scroll view content size 布局错误](http://7vzucb.com1.z0.glb.clouddn.com/blog.scrollview-4.png)

点击之后得到具体信息如下：

>   the scrollable size of a uiscrollview is computed automatically based upon the constraints of its subviews. in order to fully define the scrollable content width and height , there needs to be constraints touching all edges (leading ,trailing ,top, and bottom) of the scroll view.
>  
>   to fix ,make sure you can trace a line of continuously connected constraints within the scroll view's subviews from the leading (or top) edge of scroll view to the trailing (or bottom) edge



随便翻一下，大概意思就是说：scrollView 的滚动区域是根据子视图的约束来确定的，即根据所有子视图的约束，确定 scrollView 需要多大的区域才能全部容纳这些子视图，然后将这个区域大小设置为 scrollView 的 content size。这样 scrollView 就可以安全的滚来滚去了~



为了能够充分的定义滚动的区域，那么这些约束必须能够接触到所有的边缘（顶部、底部、左边、右边）。通俗的讲，只有你的约束设置时关联到了四个边缘，才能确定 content size 的大小。因为上下左右的边距都被设置了，整体的宽高也就确定了。



要修复这种问题的话也很简单，只要确保在 leading 到 trailing 方向（或 Top 到 bottom）上，子视图之间有一条不间断的约束线连接到 scrollView 的边缘。也是通俗的讲，即水平方向和垂直方向上，所有子视图的约束最终都能推导得出这个方向上的长度。举个例子，在我们刚才的视图中，scrollView 中从 Top 到 bottom 放置了两个 textField。在水平方向上，我们连接从 scrollView.leading - textField1 - scrollView.trailing ；（不管是 1 还是 2，这里只需要一个 textfield 就可以）垂直方向上，连接 scrollView.top - textField1 - textField2 - scrollView.bottom；



是不是真的这样呢，我们添加了一堆约束，发现仍然是一段红线，怎么办，这里给大家推荐一个办法，看一下系统是怎么干的。我们使用 add missing constraints ，看系统给我们添加了哪些约束，然后进行分析。

![系统自动添加约束](http://7vzucb.com1.z0.glb.clouddn.com/blog.scrollview-5.png)





#### 约束分析

-  imageView 和 scrollView 本身的约束并没有变化，符合预期
-  textField 被设置了约束，并且所有约束成功的「撑满了」scrollView 的 content

接下来就分析下 UITextField 上的几个约束是如何「撑满」content 的：

>   需要注意的是，textField 本身已经存在了 height 和 width 上的约束，但是你可能在截图看不出来，是因为我在 Xcode 中 开启了 Intrinsic size 为 placeholder ，就是说 textField 现在已经有了固定内容。

首先分析第一个 textField：

``` objective-c
// 以下的 superView 即 scrollView

// 水平居中
textField-1.centerX= superView.centerX

// 距离右边缘，设定一个绝对值 
textField-1.trailing + 131 = superView.trailing

// 距离上边缘，设定绝对值
textFeild-1.top = superView.top + 163

// 距离下边缘，设定绝对值
textField-1.bottom + 20 = textField-2.bottom // note

/**
*	可以看出，由上面这些约束可以完整的推断出 scrollView 的 content size：
*	由 水平居中、距离右边缘 131、本身宽度 100，得出 content.size.width = 50 + 131
*	由 距离上边缘 163、距离下面的 textField-2 20、本身高度 30， 得出 content.size.height = 163 + 30 + 20 + textField-2距离下边缘值
*	可以看出 content size 的 width 已经确定了，但是 height 还依赖于 textField-2 与下边缘的距离
*/
```



接下来分析第二个 textField-2：

``` objective-c
// top 
textField-2.top = textField-1.bottom + 20

// leading
// textField 1 和 2 跟左边缘的距离相等，左侧对齐
textField-2.leading = textField-2.leading

// trailing
// 因为 textField-1 的约束已经确定了 contentSizeS.width ，且 textField-2 的 width 固定。因此 trailing 方向上可以不需要约束

// bottom
textField-2.bottom + 20 = superView.bottom
```



#### 结论

-  scrollView 出现 ambiguity 约束警告的原因是，在 scrollView 中的 subView 不能确定 content size 的大小。
-  只要 scrollView 中的 subViews 的约束能够唯一的确定 content size ，那么就不会出现问题
-  当 content size 的 width 超出了 scrollView 本身的 width，即可水平方向滚动
-  当 content size 的 height 超出了 scrollView 本身的 height ，即可垂直方向滚动



### 如果我想要滚动呢？

由以上我们知道，如果我们想要让 scrollView 听话，需要解决掉约束的歧义，即确定 scrollView 的 content size。

确定 content size 的必要条件是，分别在水平方向和垂直方向都能有一些约束组合起来确定当前 content size 的 height 和 width。

也就是说，content size 其实由约束来决定的，你可以认为 scrollView 中的那块画布是无限伸展的。不信你试试。一般情况下，下图中，当 scrollView 是普通的 UIView 时，确定左边距以及顶部边距之后，本身又有固定的高度和宽度，最右边的红色箭头指向的那条约束是不必要的，想想为什么？

>   因为在普通的 view 中，view 的宽度已经固定了，subView 只要确定了左边距和本身的宽度，我用普通 view 自己的宽度减去前面两者，就自己能推断出来，subView 距离右边的距离了。
>  
>   但是 scrollView 为什么还得继续加一个右边距呢，特殊的原因之前已经提到了。scrollView 的 content 不是自己决定的，而是由 subView 决定，因此你得加个右边距，scrollView 才会知道自己的 content size 的宽度，向右的边距大于 scrollView 的 frame 的宽度时，就可以向右边滚动了。你看，我在这里设置了右边距为 400。



![伸展的 scrollView](http://7vzucb.com1.z0.glb.clouddn.com/blog.scrollview-7.png)





### scrollView origin 在 y  轴上 64 个像素的下移

也许你有时候会发现，scrollView 中的 content 顶部并是不严格的贴紧 scrollView 的顶部。使用 Xcode 的 Debug View hierarchy 功能，可以看到有大概 64 个像素的下移。如下图右上角：

![scroll view  content 下移 64 像素](http://7vzucb.com1.z0.glb.clouddn.com/blog.scrollview-6.png)





这其实应该是一个历史遗留问题，只要 scrollView 是 ViewController 的第一个 subView，且导航栏不隐藏时，UIViewController 在 iOS 7 下，会自动将视图中的 scrollView 进行调整，API 是这样的：

``` objective-c
// bool 值，指明是否自动去调整 scrollView 的 contentInsets

@property(nonatomic,assign) BOOL automaticallyAdjustsScrollViewInsets NS_AVAILABLE_IOS(7_0); // Defaults to YES

```



#### 解决办法

-  在 iOS7 之下设置上面的 API 为 NO
-  不要让 scrollView 作为 viewController 的第一个子视图



>   关于 scrollView 64 point 偏移可以去了解这些：
>  
>   http://corsarus.com/2014/uiscrollview-basics/
>  
>   http://segmentfault.com/a/1190000002936987



## 更加优雅的解决方案

如果在添加 subView 的约束时，需要不断的考虑其 scrollView 的 content ，是一件很麻烦的事情，所以得想个办法弄的优雅一些。目前比较常用的是给 scrollView 添加一个覆盖全部范围的 UIView 或者 containerView，然后在这个视图种添加真正的 subView，这样子就可以让 subView 针对这个 view 来布局。

### 添加 UIView

1、给 UIScrollView 添加 UIView ，命名为 contentView 设置约束上下左右边距都为0 ，铺满 scrollView 窗口。

2、然后在 UIScrollView 中进行布局真正的 subView。

然后你会发现，仍然报红线约束歧义，这是因为虽然我们给 scrollView 添加了一层，但是并没有解决基本问题：`scrollView 仍然不能确定自己的 content size 是多少？`

为了解决上面的问题，我们给 contentView 添加两个约束：

``` objective-c
conentView.width = scrollView.width
conentView.height = scrollView.height
```

这样子，UIScrollView 就知道自己的 content size 了，即 contentView 的大小。现在你在 contentView 中的布局都能够刚好铺满 scrollView。

>   问题解决的不彻底怎么能叫填坑呢，有同学会问，如果我的内容超过了一屏，想要滑动怎么办。修改约束的 multiplier 或者 constant 即可。想要水平滑动，修改 `conentView.width = scrollView.width`这条约束的 multiplier 或者 constant 即可。因为你修改了这个值后，contentView 的宽度也就变化了，即 scrollView 的可滑动区域 content size 也就修改了。
>  
>   其实，`conentView.width = scrollView.width` 这条约束写完整是，`conentView.width = scrollView.width * multiplier + constant`，如果对 autoLayout 不了解的同学还需再次查漏补缺。我们还可以将这个约束拖到代码中作为 IBOutlet。



### 添加 ContainerView

目前还没有遇到过需要添加 container view 来解决的问题，并且我自己对于 container view 本身也不是很了解。所以这里就省略了。

也有可能就是大家所说的 container view 其实就是命名为 `container view `的 UIView

## 键盘遮挡事件

如果 scrollView 中存在 textField ，且可能会被遮挡键盘时，可以这样解决：

1、创建对象保存当前被用户点击的 textField

2、监听键盘的调起和放下事件

3、在调起事件中，首先判断当前的 textField 会不会被遮挡，被遮挡时，修改 scrollView 的 contentInset ，并且判断修改之后，当前 textField 是否还在可视区域内，如果不可视还需调用 `- scrollRectToVisible:animated:` 滑动到可视区域内。

4、在键盘消失事件中，恢复 scrollView 的 contentInset 值


