---
title: Swift 中的错误处理：try, try?, try!
date: 2016-08-24 21:59:29
---


错误处理（Error Handing）是很多语言都有的特性，程序中避免不了会出现错误或者异常的情况，我们需要对于这些「非期望」发生的错误进行正确的处理。本文很简单的将 Objective-C 和 Swift 中的错误处理做一点描述，描述一下自己的想法。

<!--more-->

## Objective-C 
在 OC 中，错误处理都是将一个 NSError 对象的指针作为函数参数传入，待函数调用同步返回后，判断 error 对象是不是 nil，就知道是否发生了错误：

```objective-c
NSError *error = nil;
[task doSomethingWithError: &error];
if (error != nil) {
    // 错误处理
}

- (void)doSomethingWithError:(NSError *)error {
    // 处理任务
    ...
    if (在这里发生了错误) {
        error = ... // 具体的错误
        return;
    } 
    
    // 程序正常执行剩下的代码
}
```

如果是异步的方法，那么在 block 中，Error 也会作为参数传入。AFNetworking 中的 API 就是如此设计。

```objective-c
NSURLSessionDataTask *dataTask = [manager dataTaskWithRequest:request completionHandler:^(NSURLResponse *response, id responseObject, NSError *error) {
    if (error) {
        NSLog(@"Error: %@", error);
    } else {
        NSLog(@"%@ %@", response, responseObject);
    }
}];
```

## Swift

Swift 中对于错误的处理是 throw，表面跟 Java 其他语言很像，但底层并不同相同。一般来讲，抛出异常是很耗费 CPU 的操作，因此在移动端不是特别适合。

Swift 的 throw error 其实就是对 OC  中 NSError 的封装，封装后在语法上做了响应的约束，让你不能够像 OC 中那样随意地「忽视」错误。

OC 中，NSError 这样的参数，很多开发者都是直接传入 nil，这其实是很不好的习惯。但是编译器不会阻止你，因此会在程序中留下很多隐患。

Swift 中在编译器层面就做了限制，Swift 中有4种处理错误的方式：

###  把函数抛出的错误传递给调用此函数的代码
比如有一个方法 `buyTicket() throws `, 后面的 throws 表示该方法可能会抛出一个错误，来描述买票过程中发生的异常情况，比如钱不够了，票不够了，票不是想要的。
还有另外几个方法 ：

- `buyTrainTicket() throws`：买火车票
- `buyBusTicket() throws`：买汽车票
- `buyAirlineTicket() throws`：买飞机票
- `buySpaceShipTicket() throws`买宇宙飞船票

上面的四个买票方法也是会抛出错误的，比如汽车满员了，比如飞船爆炸啦。

现在，我们在 `buyTicket() throws ` 方法中调用上面的四个方法：

```Swift
func buyTicket() throws {
    switch 交通工具 {
        case ... :
        try buyTrainTicket()
        case ... :
        try buyBusTicket()
        ...
    }
}
```
可以看到，将小方法中可能出现的错误，原封不动扔给了大方法。这就是 「把函数抛出的错误传递给调用此函数的代码」

### 用do-catch语句处理错误

在刚才的大方法，即  `buyTicket() throws ` 中，不管是在乘坐什么交通工具时出现了问题。都得去处理它才行，不然编译器就给你报错。

因此，处理错误可以这样子：

```swift
do {
    try buyTicket()
} catch error {
    print(error)
} 
```
上面的例子中，只是简单的将错误打印出来，实际上啥也没干。但是编译器就不报错了，因为你「处理」了错误。
所以也就能够理解了，只是为了治「开发者」的懒病，也许你良心发现，觉得光打印出来没啥用，我好好地处理下错误吧。

```swift
do {
    try buyTicket()
} catch error where error.type == train{
    // 只处理买火车票时的错误
    buyOtherTrain() // 买领一趟车
} catch error where error.type == airline{
    // 处理买机票时的问题
    buyAnAirplane() // 我有钱，我自己买一架
}
```
如果只是上面的代码，那么会报错，因为你没有处理完所有的错误可能性。

```swift
do {
    try buyTicket()
} catch error where error.type == train{
    // 只处理买火车票时的错误
    buyOtherTrain() // 买领一趟车
} catch error where error.type == airline{
    // 处理买机票时的问题
    buyAnAirplane() // 我有钱，我自己买一架
} catch {
    // 居然一张票都买不到，气死了
    killMySelf()
}
```

### 将错误作为可选类型处理

我们修改下买票方法，--> `func buyTicket() throws  -> ticket:Ticket`, 其实啥也没干，就是让它把买到的票返回。

对于老板来讲，他才不会 care 你买票时发生了什么，我只要~~产出和结果~~车票！因此老板是这样买票的：

```swift
// 让秘书去买票
let boss = BOSS()
let secretary = Secretary()
let boughtTicket = try? secretary->buyTicket()
if ticket = boughtTicket {
    BOOS.print("干的漂亮！")
} esle {
    BOOS.print("gun du zi！")
    BOSS.fire(secretary)
}
```

在这里，BOSS 只关心买票的时候，什么错误情况都没发生，要么拿到了票，要么这个返回的结果票是个 nil。
try ？ 会让函数在有 error throw 时返回 nil，没有 error 时返回正常值。也就是说上面的 boughtTicket 是可选类型。
### 断言此错误根本不会发生

老板的秘书特别厉害，老板特别相信她 or 他的能力，秘书上可九天摘星辰，下可五湖四海揽月。

```swift
// 让秘书去买票
let boss = BOSS()
let secretary = Secretary()
let boughtTicket = try! secretary->buyTicket()
BOOS.print("干的漂亮！")
BOSS.kiss(secretary)
```

try! 表示我作为一只有尊严的猿，笃定这里肯定不会出错。
当然，如果真的出了错，就发生一个 runtime error 打脸。



