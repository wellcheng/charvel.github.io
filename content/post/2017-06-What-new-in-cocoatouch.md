---
title: WWDC 2016 Session 205 - What's new in CocoaTouch 总结
date: 2016-06-25 17:03:22
---

今天的 Session 将主要将四件事情：

*   你可能已经在 App 中使用的一些核心技巧，我们会讨论如何使用它们构建更好的 App
*   讨论如何通过 UIKit 和其他一些 API 构建更好的用户界面
*   展示你可以将哪些新功能集成到你的应用中
*   应用如何通过新的系统 extension 来拓展自己

<!-- more -->

## Core Technologies

### Adaptivity

自从 Apple 推出了更大屏幕的 iPad 之后，开发者在屏幕的适配上，将会花费更多的时间。

Apple Pencil 、Smart Keyboard 等新的硬件，在你的 App 中使用后，也能够增加更多的亮点。

在之前，可以通过 Size Class 来进行屏幕的自适应，比如 iPhone 是 compact，iPad 是 regular。现在有了新的 iPad Pro，当然不会是新增一种类型，因为在 Framework 中有很多的工具来表示这种屏幕尺寸。Apple 在 SizeClass 中增加了新的特性来支持各种设备的适配。

另外，iPad Air 2 和 Pro 能够每秒扫描屏幕 120 次，这远远大于屏幕上内容的刷新频率。开发者可以使用有关 Apple Pencil 、3D Touch 等 API 来做出更棒的 App。

### Swift 3

在 Swift 3 中，很多 API 都进行了重写，因此在使用 Swift Coding 时将会有更好的体验。比如新的字体，WhiteColor 和 blackColor 简化为 white 和 black，以及与 Core Graphics 相关的一些 API，都做了简化。

GCD，现在是一个完整的对象，之前使用 GCD 创建自定义队列并加入 task 是这样子：

```swift
let queue = dispatch_queue_create("com.example.queue, nil")
dispatch_async(queue) {
  // ...
}
```

Swift 3 中是酱紫：

```swift
let queue = DispatchQueue(label: "com.example.queue")
queue.async {
  // ...
}
```

这种写法简直爽的不行，特别简洁明了。

另外，增加了一个新的特性特别棒，在创建自定义 queue 时，可以使用 autoReleasePool 将 work item 包装起来，写法很简单：

```swift
let q = DispatchQueue(label: "com.example.queue", attributes: [.autoreleaseWorkItem])
```

有关 GCD 的其他内容可以去看单独的 GCD session。

### 单位和计量制

在 Foundation 框架中，也有很多新的提升，比如 Units 和 measurements ，以及新的 DateFormatter：NSISO8601.

增加了 NSDateInterval ，这些计量和单元的内容可以观看另一个 Session 来详细了解。

### 剪贴板

在 WWDC 中提到过，现在剪贴板可以跨设备使用，但是需要注意的是，如果剪贴板的内容很大时，那么在粘贴时可能会触发一个 loading 动画。

为了避免产生这个动画，可以在粘贴前先做一次检查，Apple  提供了这些 API：

```swift
public class UIPasteboard : NSObject {
   public var hasStrings: Bool { get }
   public var hasURLs: Bool { get }
   public var hasImages: Bool { get }
   public var hasColors: Bool { get }
```

在粘贴之前，可以先检查下粘贴板中是否有自己需要的内容，避免 loading 动画。

同样，你也可以控制向剪贴板发布哪些内容，在将内容发布到剪贴板前，可以设置过期时间，也可以限制只能用在当前设备。



### Color 色域

在 iMac 5K 和 iPad Pro 9.7 上，可表示的颜色将更多。

在 iOS 和 MacOS 上，已经暴露了一些颜色的 API，可以使用更加宽广的色域。

这里的提升，主要会体现在生产创造上。

Apple 并没有因为 iMac 5K 和 iPad Pro 9.7 支持 pRGB 来增加新的类，只需要使用这两个 API 就好了：



```swift
public class UIColor : NSObject {
   public init(red: CGFloat, green: CGFloat, blue: CGFloat, alpha: CGFloat)
   public init(displayP3Red: CGFloat, green: CGFloat, blue: CGFloat, alpha: CGFloat)
```



### Image Render 图像渲染

在之前，我们一般是创建一个 image 上下文，然后做一些自定义的绘图操作，接着获取上下文中的图像内容，然后结束这个上下文，这样子就能获取到图片了，其实这是一种常见的错误方法，因为这么做的话，图片将只有 32 bit 的 sRGB 。

这不是一个很好的 API，既没有 block ，也不能进行拓展。

之前是酱紫：

```swift
 func createDrawing(size: CGSize) -> UIImage {
   let renderer = UIGraphicsBeginImageContext(size)
   // Do your drawing here
   let image = UIGraphicsGetImageFromCurrentImageContext()
   UIGraphicsEndImageContext()
   return image
}
```

但是现在，Apple 新增了类 `UIGraphicsRenderer`，它是 block-based、full-color-managed，并且是可以拓展的，可以做位 image 和 PDF 的子类。并且管理 context 的生命周期，因此可以做一些内存优化。

下面给出正确的使用方式：

```swift
func createDrawing(size: CGSize) -> UIImage {
   let renderer = UIGraphicsImageRenderer(size: size)
   return renderer.image { rendererContext in
  } }
```

### Asset Management

因为我对图像色域这块不是很了解，这里大概讲的就是可以使用更宽色域的图片，并且在支持显示的设备上选择高色域的显示，不支持的设备上仍然使用普通的进行显示，这与 App Thing 没有冲突，是完美结合的。

并且这种选择也是自动的，好比 @2x @3x 图片的选择。

### Accessibility Inspector

Accessibility是Apple很早之前构建的一个框架，它能帮助一些行动不便的用户来更好地使用你的应用。它为你的UI提供了丰富的语义数据，这能让不同的Accessibility功能给行动不便的用户展现你的应用。

iOS 10 在这里也提升了体验。

### Speech Recognition 朗读识别

语音朗读推出了更加简洁的 API，可以实时朗读，也可以自定义音频文件或者在线的音频流。

## UIKit

### Smart Text Input

文本输入是 App 中最常见的场景，现在可以为文本输入增加一个类型信息，比如位置、电话、邮件、信用卡和数字等，这样系统就能够提供更加智能的输入建议。

在 iOS 7 中增加了动态类型，可以使得文本的大小更随系统设置而改变，现在这个特性在 iOS 10 有了提升。

### UITabBar 自定义

iOS 10 对于 UITabBar 的自定义给出了官方的 API，在之前，如果我们需要一个个性的 tabBar，那么就需要做很多工作，现在简单多了：

-   自定义的 badge 颜色和文本属性 text attributes
-   自定义未选中状态时的 tintColor

```swift
tabBarItem.badgeColor = UIColor.white()
badgeTextAttributes = [NSForegroundColorAttributeName: UIColor.blue(), NSFontAttributeName :UIFont.italicSystemFont(ofSize: 12) ] 
tabBarItem.setBadgeTextAttributes(textAttributes: badgeTextAttributes, 
                                    forState: UIControlStateNormal)
tabBar.unselectedTintColor = UIColor.brown()
```

### 3D Touch 支持

WKWebView 新增了 delegate 方法，作为 3D touch 的支持。

新增类 `UIPreviewInteraction` ，这意味着，3D touch 的支持可以使用自定义的动画。

另外还有一个小亮点就是 UIRefreshControl 支持自定义的 UIViewController 了。

### CollectionView

在 iOS 9，为 collectionView 的流式布局（Flow Layout）增加了 automatic self-sizing cell，但是这需要你计算一个大概的 size。

现在为 Flow Layout 增加了新的模式，你将不在需要返回一个估算的 size。

CollectView 并还将支持分页，之前这个功能是 ScrollView 才有的。

Apple 还特别的重新设计了底层，使得 collectionView 能够更加平滑的滑动。假如你现在的 collectionView 每行有三个 cell，那么在快滑动到下一行时，将会一次性创建 3 个 cell，再假如每一个 cell 都比较复杂并且耗费比较多的时间时，那么将会有卡顿。

Apple 这里提供了一个叫做 （单元格预读）cell prefetching 的技术，也就是说，你的 cell 还不需要显示在屏幕上时，可能系统会调用 data source 方法来跟你要这个 cell。

cell prefetching 技术是 iOS 10 底层实现的，所以我们无需关注和修改，将会自动获得这项技术的支持，可能还有一些需要注意的地方就是，你需要保证你的 `cellForxxx` 方法，不管系统在什么时候调用，它都能返回正确的 cell。

Apple 不但为 cell 做了预处理，还对数据的获取也增加了预处理的 delegate 方法，这样子，我们就能够在 cell 显示之前，做一些网络加载、硬盘数据读取等工作，大大提高了性能。并且 Data PreFetching 并不是 collectionView 特有，TableView 也加入了支持。

### New Animator

动画这块也加入了更新，新的动画 API 支持动画的中断、取消、反向等等，动画时间函数也增加了更多的类型。

新的动画将好比电影，你可以快进一段时间、倒退一段时间、暂停、取消、反向播放等。

用法也很简单：

```swift
// 创建动画时间函数
let timing = UICubicTimingParameters(animationCurve: .easeInOut) 
// 创建动画
let animator = UIViewPropertyAnimator(duration: duration, timingParameters: timing)
// 添加动画块
animator.addAnimations { 
	self.squareView.center = CGPoint(x: point.x, y: point.y) 
}
// 执行动画
animator.startAnimation()
```



可以看到，新的 API 特别简洁明了。

通过这种新的 API，我们可以将动画和手势控制合成在一起，创建出更加 excited 的视觉交互！！！

### Open URL - URL 跳转

在之前，假如我们通过一个 url 需要跳转到微博，那么首先得通过 url 跳转到 Safari，接着如果安装了 weibo，那么就可以跳转过去，如果没有，就停留在 Safari。

我们都是很不希望，一个操作需要在 App 外面操作或者迫使用户离开 App。

新的 API 解决了这个问题，在跳转前可以判断系统是否安装微博，如果有就直接跳过去，没有的话就在 App 内部打开网页。

```swift
UIApplication.shared().
   open(url, options: [UIApplicationOpenURLOptionUniversalLinksOnly: true]) {
      (didOpen: Bool) in
         if !didOpen {
            // 没有安装 App，做自己的操作
} }
```

## System Kit

### CoreData

对 query generation 做了优化，简化了 CoreData 代码（再怎么简化也很难用 Orz…）。

另一个优化的地方是连接池（Connection Pool），现在 CoreData 可以提供多个 Reader，一个 writer。这将会带来更多的性能提升。

### CloudKit

支持存储公共文档：Public databases。

支持每个人一个库（per user database），这将使得应用能够支持多用户。

数据共享：通过新的 UICloudSharingController 类进行管理。

### User Activity

我们知道，Handoff 是通过 `User Activity` 这个独立的信息集合单位，不依赖于其他进行传输的，iOS 10 增加了用户地理位置。

这样子，我们就可以在 handoff 中增加更多更棒的特性了。

### App Search

iOS 9 加入了应用内搜索，但是在 iOS 10 对此功能做了提升，当用户在系统的 search 中找到了查询结果后，可以点击 App 图标中的 search in app，接着进入 App 持续查找。

这个新功能可以通过很简单的方式就能够集成到 App 中。

### ApplePay

Apple Pay 在 Web 和 Extensions 中也支持啦。

这意味着 SFSafariViewController 也支持了。

在 iMessage 中也能轻松集成。

## System Extension

### widgets

widgets 现在有了两种形式，compact 和 expanded 。

可以很方便的在这两种形式上显示不同的内容。

### Notificaiton

media attachment：通知能够附加媒体信息。

end to end encryption：通知数据端到端的编码，这样子可以增加安全性。

 embedded UI views：在通知中嵌入 UIView ，想想是不是很激动，打完 Uber，然后车到了，同时在通知中给出司机的位置图片；收到快件正在配送的通知，同时显示快递员的照片。不禁再吼一声 excited！



### CallKit

callKit 增加了很多 VOIP 方面的特性，不过这个东西明显是动了天朝运营商的奶酪，以后的发展情况也不知道如何。

### Siri

Siri 现在变的更加智能，能够根据上下文、语义等理解你的要求，这是很复杂的工作，虽然 Siri 在英语环境中确实很棒，但是到了天朝毕竟还是有些水土不服，中文可是博大精深啊！就问 Cook 你怕不怕。

Siri 现在支持第三方的 App 拓展，SiriKit 提供了一些语义化的组件，能够让用户通过 Siri 调用你的 App。

比如你设置了「记账」、「类型」、「多少钱」，如果用户呼叫 Siri：「我需要记账，我刚才吃午饭花了 20 块钱」。

那么 Siri 就会命中你的设置，然后打开你的 App。

滴滴打车的 Siri 支持也是这种类型。

### iMessage

iMessage 做了很大的更新，具体的可以看有关 iMessage 相关的 session。