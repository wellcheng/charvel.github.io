---
title: 在 AutoLayout 和 Masonry 中使用动画
date: 2017-06-14 23:04:39
---

动画是 iOS 中非常重要的一部分，它给用户展现出应用灵气的一面。

## 在动画块中修改 Frame

在原来使用 frame 布局时，在 UIView 的 animate block 中对 view 的布局进行修改，动画即可生效。

```
[UIView animte ... {
    view.frame = ...
}];
```
## AutoLayout 没有 Frame 时如何做动画

在 AutoLayout 中，view 的布局等属性都是由约束 constraint 决定，Apple 也不建议在使用 AutoLayout 直接修改 view 的 frame。

在介绍之前，先看一下 UIView 的 `layoutIfNeeded` 方法：

<!--more-->

```objc
/** 
Allows you to perform layout before the drawing cycle happens. 
-layoutIfNeeded forces layout early
*/
- (void)layoutIfNeeded;

/**
Lays out the subviews immediately.
Use this method to force the layout of subviews before drawing. 
Using the view that receives the message as the root view, 
this method lays out the view subtree starting at the root.
*/
```
换而言之，向 view 发送该方法，将 view 作为 root view，开始向下重新布局（即计算 subView 的 frame）。

可以想象一下，在计算布局的过程中，subView 的 frame 发生了变化。

而我们之前使用动画时，是将这些变化的代码放在 UIView Animation Block 中。
所以，在 AutoLayout 中使用 Animation 的正确姿势可以是：

```
UIView *animationView = [[UIView alloc] init];
// 给 animationView 设置约束
[self.view addSubview:animationView];

[UIView animateWithDuration:10.0f delay:0 options:UIViewAnimationOptionCurveLinear | UIViewAnimationOptionRepeat animations:^{
    // 修改 animationView 的宽度约束
    animationView.widthConstrint.constant = 100.0f;
    // 注意这里要调用 animationView 父视图的 layoutIfNeeded 方法
    [self.view layoutIfNeeded];
} completion:^(BOOL finished) {

}];
```

> [self.view layoutIfNeeded]; 这行代码，其实就是让系统帮我们去做 frame 相关的修改。

## 约束还有约束

现在，我们学会了在 Animation Block 中修改约束来达到动画的目的。
但是这需要我们持有对该 constraint 的引用才行。如果页面中有大量的动画需求，那么就会出现如下例子：

```
@interface SSAnchorRankView ()
@property (nonatomic, strong) NSLayoutConstraint *labelWidthCons;
@property (nonatomic, strong) NSLayoutConstraint *labelHeightCons;
@property (nonatomic, strong) NSLayoutConstraint *labelTopCons;

...

@end
```

也就是说，实现动画时需要改变的约束，都需要在类中增加属性来引用它。

虽然可以将这些属性单独放在一个 category 中，但是仍然避免不了要大量引用约束的尴尬。

### 复杂的动画会导致什么

复杂的动画，会让你在设计约束时，考虑的内容太多，也容易出现一些约束的 Bug。

### 优雅的姿势

其实在布局的过程中，是可以将界面的显示部分做一个抽象的划分。例如在编写一个音乐播放器的界面时，将相似或者耦合的元素包装起来，作为一块单独的内容区域：
 - 歌曲专辑图片、歌曲名、歌手名
 - 歌词、歌词控制
 - 播放暂停、下一首、上一首
 - 喜欢歌曲、不喜欢歌曲

这样，将元素分别装在不同的 container view 中，抽象起来，在之后不管是做约束，还是做动画，都会更加方便。

## Masonry 呢？

Masonry 的 make 方法会生成约束，其实，它本身是有返回值，类型为 MASConstraint *，所以也是可以持有 Masonry 生成的约束并改变其值。

```
[label mas_remakeConstraints:^(MASConstraintMaker *make) {
    self.labelWidthCons = make.width.equalTo(self.view).offset(200);
}
```


## 进阶

Constraint 本身可以修改的属性并不多，有时候需要转换一种思路，比如 install 两条约束，在不同的 case 下激活其中一条即可。

Constraint 本身有很多方法还可以继续研究，当然这个就是后话了。


