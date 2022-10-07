---
title: Swift 中闭包 closure 的对象捕获
date: 2016-09-14 22:32:18
---


在 swift 中，closure 可以捕获当前作用域中的变量，并且持有它。

函数是闭包的一种，因此函数也是可以持有变量的。

<!--more-->

```swift
import UIKit
import XCPlayground

func makeGrowthTracker(forGrowth growth:Int) -> () -> Int {
    var totalGrowth = 0
    func growthTracker() -> Int {
        totalGrowth += growth	// 在 closure 中使用了外部 scope 中的变量，将其捕获
        return totalGrowth
    }    
    return growthTracker
}

var currentPopulation = 2
// growBy500 是函数类型，其类型为 () -> Int
let growBy500 = makeGrowthTracker(forGrowth: 500) 

growBy500()  // 500
growBy500() // 1000

func printValue(value:() -> Int) {
    print(value())
}

printValue(growBy500) 	// print 1500
printValue(growBy500)	// print 2000

currentPopulation += growBy500()	// current = 2502

growBy500()	// 3000
```

在上面的函数 `makeGrowthTracker` 中，传入一个增长值，然后会返回一个函数，调用返回的函数，函数的返回值会增加一个增长值，就是说，函数每次调用的结果是不相同的。

函数的定义与存在也是依托于计算机内存，你可以想象下，函数就是变量，然后函数存放在内存中的某个位置，`growBy500` 指向了这块内存，当函数被调用时，内存中的函数体就会被执行一次。

如果函数是普通的函数，那么每次函数调用时，内部的状态应该都是全新的。但是函数如果持有了变量，那么就可能会不同。

如果变量 `totalGrowth` 在函数的内部，如下：


```
func makeGrowthTracker(forGrowth growth:Int) -> () -> Int {
    ~~var totalGrowth = 0~~ 	// 被删除的代码
    func growthTracker() -> Int {
        var totalGrowth = 0	// 将变量的定义挪动到这里
        totalGrowth += growth
        return totalGrowth
    }    
    return growthTracker
}
```

那么，函数的每次调用都会重新初始化变量 `totalGrowth`，然后函数 growBy500 的调用，每次都是同样的值。

变量 `totalGrowth` 放在外部时，在函数 growthTracker 内部引用它时，函数会将引用的外部变量持有，你可以认为函数会创建一个自己的内部变量，这个内部变量指向了变量 `totalGrowth` 的内存地址。所以第一次执行完 `growBy500` 之后，变量 `totalGrowth` 的值会由 0 变为 500。
当函数执行完毕之后，这个值仍然是 500 ，并不会消失或者重新变为 0 。因为这块内存被引用，那么也不会被释放。

这个就是 closure 持有变量的原理。当然，真实的情况下要比这个复杂的多，涉及到内存的申请和释放，都是比较谨慎的，以免产生野指针或者内存泄露。

