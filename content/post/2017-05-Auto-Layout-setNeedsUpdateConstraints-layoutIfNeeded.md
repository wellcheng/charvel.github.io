---
title: Auto Layout 中的 setNeedsUpdateConstraints 和 layoutIfNeeded
date: 2017-05-14 23:03:01
---

先抛出结论：

`setNeedsUpdateConstraints` 保证之后肯定会调用 `updateConstraintsIfNeeded` .

`SetNeedsLayout` 保证之后肯定会调用 `layoutIfNeeded` .



## AutoLayout 的本质

>   AutoLayout 是指，用一套规则（约束）来定义视图之间的位置。
>
>   AutoLayout 能够让每个 view 有唯一的 frame。

其实，这样子解释，还是让人很难理解，所以接下来会简单介绍下 AutoLayout ，在对其有所了解和深入后，再解释下面这几个问题：

-   `保证之后调用` 的之后是在什么时候？
-   这些方法的调用时序大概是怎么样的？
-   为什么要先 set 一下，而不是直接 updateConstraint 和 layout
<!--more-->

## UIView 生命周期



```flow

init=>start: InitWithFrame

layout=>operation: setNeedsDisplay: 
setNeedsUpdateConstraint:  
setNeedsDisplay:

ifUpdateCons=>condition: update constraints ?
updateCons=>operation: updateConstraints
ifLayout=>condition: update layout ?
layoutSubview=>operation: layoutSubviews
ifDisplay=>condition: needs display ?
draw=>operation: Draw in Rect
e=>end: Event Loop

init->layout->ifUpdateCons
ifUpdateCons(yes)->updateCons->ifLayout
ifUpdateCons(no)->ifLayout
ifLayout(yes)->layoutSubview->ifDisplay
ifLayout(no)->ifDisplay
ifDisplay(yes)->draw(right)->e(right)
ifDisplay(no)->e(right)
e->ifUpdateCons
```

在 iOS 中，AutoLayout Engine 是一个迭代机。

为什么这样说呢？先回到手写布局时代，我们通过计算 view 与屏幕之间的相对距离，得出 view 实际的 frame，然后赋值给 view，这样子每个 view 都会按照我们设置的位置正确地显示。

接着我们来到当下的 iOS 开发，先列出几个问题：

-   需要适配多种屏幕
-   既可以在 iPhone 上使用，又可以在 iPad 上使用
-   可以在横屏下使用
-   在不同的屏幕尺寸下，有不同的布局方式（比如屏幕小，就一行放一个，屏幕大一行放两个）

如果仍然在原始的手写布局下去完成上述工作，势必累死，也不一定能够很好的完成。

### 搞个机器人

人类能够发展到现在的文明，就是因为 `善假于物也`。

我希望现在有一套东西，我只需要告诉它，我希望视图表现成什么样子，然后它就会按照我的期望去计算出每个 view 的 frame，我只需要把 frame 拿来用就行了。

AutoLayout 就是这样的一套东西，它接收视图与视图之间的规则，生成最终的 frame。

>   这里有个误区，AutoLayout 的确是最终生成了 frame，不过生成之后自动给 view 赋值上去了，所以我们没有看到 setFrame 这个过程。

这个规则就是约束 constraint。

### 为什么需要 update constraint

在 UIView 显示之前，先判断 view 是否能根据当前约束计算出唯一的 Frame。如果可以，那么就根据这个 Frame 去布局。同时，在这里我们认为 view 是满足约束的。

严谨一些，是指在 AutoLayout Engine 中，该 view 的 constraint 是否为最新。

AutoLayout Engine 是一个单独的约束处理系统，在绝大多数操作中，比如：

*   Activating或Deactivating 启用和停用
*   设置constant或priority
*   添加和删除视图


AutoLayout Engine 本身都会标记 view 的约束不是最新，即调用 `setNeedsUpdateConstraint` 。

但是也有例外，AutoLayout Engine 并不知道你又修改了约束。因此在这种 case 下，需要手动调用  `setNeedsUpdateConstraint` 来标记约束需要更新。

#### setNeedsUpdateConstraint

`setNeedsUpdateConstraint` 控制 view 的约束是否需要更新。当一个自定义view的某个属性发生改变，并且可能影响到constraint时，需要调用此方法去标记constraints需要在未来的某个点更新，系统然后会调用 updateConstraints，. 以解决这个由属性改变带来的影响。

#### updateConstraintsIfNeeded

`updateConstraintsIfNeeded`立即触发约束更新，自动更新布局。

#### updateConstraints

当 Custom View 发现属性或者其他的改变导致它的所有约束中有一个失效时，首先应该删除这个失效的约束，然后调用 setNeedsUpdateConstraints 表示当前的约束需要更新，然后在 updateConstraints 中恰当地地方检查当前 content 所需的必要约束。

>   注意：要在实现在最后调用[super updateConstraints]

#### layoutSubviews

在确定了 view 的约束后，AutoLayout 通过计算可以得出 view 的 frame，计算的过程就是 layout 的过程。

Layout 的顺序，是由最外层向里递进，所以子视图只需要相对于父视图做好布局就可以。

在调用 layoutSubviews 的同时，也会调用 setNeedsUpdateConstraint。



## Auto Layout Process 自动布局过程（引用自Objccn.io）

与使用springs and struts(autoresizingMask)比较，Auto layout在view显示之前，多引入了两个步骤：updating constraints 和laying out views。

每一个步骤都依赖于上一个。display依赖layout，而layout依赖updating constraints。显示之前首先得知道布局，想要完整的布局就得更新约束（约束才能得出布局啊）。

updating constraints->layout->display

第一步：updating constraints，被称为测量阶段，其从下向上(from subview to super view),为下一步layout准备信息。

可以通过调用方法setNeedUpdateConstraints去触发此步。constraints的改变也会自动的触发此步。但是，当你自定义view的时候，如果一些改变可能会影响到布局的时候，通常需要自己去通知Auto layout，updateConstraintsIfNeeded。

自定义view的话，通常可以重写updateConstraints方法，在其中可以添加view需要的局部的contraints。

第二步：layout，其从上向下(from super view to subview)，此步主要应用上一步的信息去设置view的center和bounds。可以通过调用setNeedsLayout去触发此步骤，此方法不会立即应用layout。如果想要系统立即的更新layout，可以调用layoutIfNeeded。另外，自定义view可以重写方法layoutSubViews来在layout的工程中得到更多的定制化效果。

第三步：display，此步时把 view 渲染到屏幕上，它与你是否使用Auto layout无关，其操作是从上向下(from super view to subview)，通过调用setNeedsDisplay触发，

因为每一步都依赖前一步，因此一个display可能会触发layout，当有任何layout没有被处理的时候，同理，layout可能会触发updating constraints，当constraint system更新改变的时候。

需要注意的是，这三步不是单向的，constraint-based layout是一个迭代的过程，layout过程中，可能去改变constraints，有一次触发updating constraints，进行一轮layout过程。

注意：如果你每一次调用自定义layoutSubviews都会导致另一个布局传递，那么你将会陷入一个无限循环中。

就是说，layout 和 updateConstraints 不断迭代最终确立了整个布局和显示，然后交给屏幕去显示 

## 实践出真知

### layoutIfNeeded 调用导致 Crash

在调用 layoutIfNeeded 时，view 必须要被 setNeedsLayout 后，才会理解执行 layoutSubviews。

>   一个视图缺少高宽约束，在设置完了约束后执行layoutIfNeeded，然后设置宽高，这种情况在低配机器上可能会出现崩问题。原因在于layoutIfNeeded需要有标记才会立刻调用layoutSubview得到宽高，不然是不会马上调用的。页面第一次显示是会自动标记上需要刷新这个标记的，所以第一次看显示都是看不出问题的，但页面再次调用layoutIfNeeded时是不会立刻执行layoutSubview的（但之前加上setNeedsLayout就会立刻执行），这时改变的宽高值会在上文生命周期中提到的Auto Layout Cycle中的Engine里的Deferred Layout Pass里执行layoutSubview，手动设置的layoutIfNeeded也会执行一遍layoutSubview，但是这个如果发生在Deferred Layout Pass之后就会出现崩的问题，因为当视图设置为setTranslatesAutoresizingMaskIntoConstraints:NO时会严格按照约束->Engine->显示这种流程，如在Deferred Layout Pass之前设置好是没有问题的，之后强制执行LayoutSubview会产生一个权重和先前一样的约束在类似动画block里更新布局让Engine执行导致Ambiguous Layouts这种权重相同冲突崩溃的情况发生。

### RemoveFromSuperView

将多个有相互约束关系视图removeFromSuperView后更新布局在低配机器上出现崩的问题。这个原因主要是根据不含视图项的约束不合法这个原则来的，同时会抛出野指针的错误。在内存吃紧机器上，当应用占内存较多系统会抓住任何可以释放heap区内存的机会视图被移除后会立刻被清空，这时约束如果还没有被释就满足不含视图项的约束会崩的情况了。

remove 之前最好能够 clearConstraint。



## 我的一些疑问

>   第一步：updating constraints，被称为测量阶段，其从下向上(from subview to super view),为下一步layout准备信息。

始终不明白为什么要从下往上更新约束，我的理解是，先是父视图确定自己位置，子视图才确认自己视图。当然这个是 layout 的过程。

我当前的理解是这样，子视图的约束先更新，再逐步向上触发父视图更新约束，猜测的原因是子视图的约束可能会导致父视图约束更改。

## 参考引用

-   https://objccn.io/issue-3-5/

    ​


