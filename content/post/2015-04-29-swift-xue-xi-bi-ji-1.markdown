---
layout: post
title: "Swift 学习笔记(一)"
date: 2015-04-29 19:22:58 +0800

tags: [Swift, 学习笔记]
---

> swift 这门语言一直想取深入了解下，苦于时间少，现在快毕业了，呆在学校时间还算比较充足，所以开始学习一哈。

<!--more-->


## 基础部分

### 常量和变量

常量和变量必须在使用前说明，常量使用 let ，变量使用 var 声明，其定义与其他语言一致，强烈建立在使用不会变化的值时，将其声明为常量。

声明常量或者变量的时候还可以加上 `类型标注 (type annotation)`, 说明常量或者变量中值的类型，格式一般为常量(变量)名 + 冒号 + 空格 + 类型名称

    var userName: String 
    
声明中的冒号其实就是 `是 ...  的类型`，所以上面代码可以理解为：

- “声明一个类型为 String，名字为 userName 的变量”
- “类型为 String ” 的意思就是可以存储任意 String 类型的值

现在，userName 变量可以被设置为任何 String 类型的值：

    userName = "Well Cheng"
    
需要注意的是，绝大部分情况下，你是不需要去为变量或者常量去写类型标注的，因为在初始化的过程中，Swift 可以根据初始化的初值推断出其类型。上面的例子是因为声明时没有给 userName 赋值，而是在声明后赋值的，所以需要类型标注。实际中，应该是酱紫的：

    var userName =  "Well Cheng"
    
Swift 能够根据你赋的初值 “Well Cheng” 推断出 userName 是 String 类型的变量，常量也是如此。


对于常量或者变量的命名，相信很多看过 WWDC 的开发者都惊讶了，表情竟然和也可以作为变量名，（字母或下划线开头什么的见鬼去吧~）。其实，因为表情是 Unicode 字符，所以才可以作为变量名。常量与变量名，不能包含数学符号、箭头、非法（保留）的 Unicode 码位、连线和制表符。当然，也不能以数字开头。

一旦常量和变量被声明后，一是不能再次被声明，二是不能改变其存储值的类型，三是不能改变常量的值，最后，常量和变量之间不能进行互转。

还有一点就是起名的时候，要注意不能与 Swift 的保留关键字冲突，实在实在没有办法可以使用反引号将其包住。

常量的值是可以改变为其他同类型的值，例如：

    var userName = "Well"
    userName = "Cheng"
    
常量的值一旦确定不能改变：
    
    let languageName = "Swift"
    languageName = "Swift ++"    // 编译器报错

可以使用 println 函数来输出常量或者变量的值，默认最后换行。
Swift 可以使用叫做`字符串插值(string interpolation)` 的方式把常量或者变量当做占位符插入到字符串中，格式为反斜杠后跟括号，括号里面为变量或常量名。

    println("current user name is \(userName)")
    
### 注释
单行注释双斜杠：`// 这是一条注释`，多行注释用 */ 包含，例如 `/* 这里面可以放多行注释*/`。当然这些也都是其他语言有的，Swift 不同的是嵌套注释，即 `/*` 是可以多层嵌套的，这样的话，以前大片大片的注释代码的时候，已经被注释掉的代码就不会出来闹眼子了。

### 分号
在 Swift 中，分号是可以省略的，除了你在一行里面写了多条语句，如下：

    let cat = "🐱"; println(cat)

### 整数
整数就是没有小数部分的数字，Swift 提供了 8，16，32 和 64 位的有符号以及无符号整数类型。跟之前的 `UInt8`、 `Int32` 这些基本一致。

#### 整数范围
可以访问不同整数类型的 `min` 和 `max` 方法来获取

#### Int
一般来说，在使用整型时，不需要专门指定其长度，因为有 Swift 提供的 Int 类型。

- 32位：Int 与 Int32 长度相同
- 64位：Int 与 Int64 长度相同


#### UInt
无符号整型：

- 32位：UInt 与 UInt32 长度相同
- 64位：UInt 与 UInt64 长度相同

需要注意的是，尽量不要使用 UInt，就算你知道这个值是非负的，这个与类型安全和类型推断有关，使用默认的 Int 真真是极好的。

### 浮点数
Double 表示 64位浮点数，Float 为 32 位浮点数，根据你需要的精度正确选择即可。

### 类型安全和类型判断
Swift 是类型安全（type safe）的语言。类型安全的语言可以让你清楚的知道需要被处理的值的类型。Swift 因为是类型安全，所以会在安全期间进行类型检查。

如果没有显示的声明类型，Swift 还会使用类型推断来选择适合的类型。因为这个特性，所以 Swift 很少需要去声明类型。

在声明常量或者变量并初始化时，类型推断特别有用，它会按照你传入的字面量推断出其类型。例如：
    
    let meaningOfLife = 22
    // Swift 根据字面量 ‘22’ 推断出其类型为整型
    
在推断浮点数时， Swift 默认选择 Double，这个应该是为了安全考虑
表达式中同时出现整数和浮点数时，推断为 Double

### 数值型字面量
直接上代码，
这些是整数：

    let decimalInteger = 17
    let binaryInteger = 0b100101    // 二进制，0b 开头
    let octalInteger = 0o164527     // 八进制，0o 开头
    let hexadecimalInteger = 0x1F2C // 十六进制，0x开头
    

浮点数中，小数点两边必须要有数字，还可以采用科学计数法：

    let floatValue = 1.24e2    // 1.25 x 10^2 即 125。0
    let hexFloatValue = 0xFp2    // 15 x 2^2 即 60.0
    
    let decimalDouble = 12.1875
    let exponentDouble = 1.21875e1
    let hexadecimalDouble = 0xC.3p0
    
增加无用的 ‘0’ 和 下划线，并不影响字面量的值

### 数值类型转换
经常使用默认的 Int 类型，可以保证你的整数常量和变量可以直接被复用，并且可以匹配整数类型字面量的类型推断。只有在必要的时候，才会使用其他整型类型。

#### 整数转换
不同类型的整数可以存储不同范围的值，所以你有时候必须要进行类型转换，而且是显示的，这种显示的类型转换有很多好处，这样子能够更加清楚的表达意图。

在类型转换时，使用目标类型转换数值，如下：

    let twoThousand: UInt16 = 2_000
    let one: UInt8 = 1
    let twoThousandAndOne = twoThousand + UInt16(one)

这种 `SomeType(ofInitalvalue)` 的形式，是调用 Swift 构造器并传入初始值的默认方法。其中的原理就是，UInt16 类型内部有一个可以接受 UInt8 类型的构造器，所以能够对 UInt8 类型进行转换。如果需要转换的目标类型不存在对应构造器是不能转换的。不过你可以使用拓展来增加一个。

#### 整数和浮点数转换
两者的相互转换都需要显示的转换，不过在使用数值字面量的时候，不需要，例如

    let π = 3 + 0.14
    
因为字面量本身是没有类型的，编译器会推断它的类型(π被推导为 Double 类型)


### 类型别名
类型别名（type aliases）就是给现有的类型起另外一个名字，使用 `typealias` 关键字定义，实际运用中主要为方面理解，我的个人理解编译器编译时会使用真实的类型替换掉别名。

### 布尔值
Swift 中布尔类型为 Bool，即逻辑上的真假，有 true 和 false，在需要使用 Bool 类型的地方使用别的值会报错，如下：

    let i = 1
    if i {
        // 报错，Swift 没有隐形的类型转换
    }
    
    if Bool(i) {
        // 正确
    }
    
    if i == 1 {
        // 正确
    }

### 元组
元组（tuples）把多个值组合成为一个复合值。元组内的值是任意类型，不要求是相同类型。

例如，（404，“Not Found”） 是一个 HTTP 状态码的元组
    
    let http404Error = (404, "Not Found")

上面 http404Error 的类型为元组 （Int，String），也可以将元组的内容分解为单独的变量或常量

    let (statusCode, statusMessage) = http404Error
    println(the error code = \(statusCode) and info is \(statusMesage))
    
如果只需要元组中其中一部分的值，可以将需要忽略的部分用下划线替换 （_）:

    let (code, _) = http404Error

还可以使用下标访问单个元素：

    let code = http404Error.0
    
还可以定义元组时给其命名，之后使用名称访问：

    let http200Status = (code: 200, des: "OK")
    
    println("code is (\http200Status.code) and info is \(http200Status.des)")

元组经常用于函数返回，当有多个值需要返回时，可以将其组装为一个元组

> 元组在临时使用时很方便，如果你的程序中有多处用到相同类型的元组，需要考虑用类或者结构体将其重新定义

### 可选类型
可选类型（optionals）用来处理值可能缺失的情况。可选类型表示，有值就等于本身，或者没有值时为可选值或者 nil（此时的 nil 不同于 OC ，Swift 中 nil 可以表示任何类型的值缺失情况）

#### if 语句以及强制解析
可以使用 if 语句来判断一个可选是否包含值，如果可选类型有值，返回 true；没有值时返回 false，当确定其有值后，可以使用 ！来表示我确定这个可选类型是存在值的，你编译器就当做正常的值来调用吧。这个过程称为强制解析：
    
    let possibleNumber = "123"
    if convertedNumber = possibleNumber.toInt()    // convertedNumber 类型推断为 “Int?” ,表示一个可选类型，因为编译期间无法确定 possibleNumber 能否被正确转换
    
    if convertedNumber != nil {
        println("\(possibleNumber) has a integer value of \(convertedNumber!)")    // 使用感叹号
    } else {
        println("\(possibleNumber) could not be converted to an integer")
    }
    
如果你使用了 ！，但是运行期间值并不存在，那么会导致运行期间发生错误，所以一定要使用 if 语句来判断

#### 可选绑定
使用可选绑定（optional binding）来判断可选类型是否包含值，如果包含就把值赋给临时常量或者变量。常用语 if  while 语句：

    if let constantName = someOptional {
        ...
    }
    
重写上面的例子:

    if let actualNumber = possibleNumber.toInt() {
        // use actualNumber
    } else {
        // handle error
    }

if 语句的意思就是，“如果possibleNumber.toInt返回的可选Int包含一个值，创建一个叫做actualNumber的新常量并将可选包含的值赋给它。”，然后就可以在第一个分支使用新的常量值了，这样子做的好处就是，使用了新的常量来存储，不必像之前那样加 ！来告诉编译器，对于使用 let 还是 var 根据你的实际情况，如果接下来的任务中需要对 actualNumber 做修改，那么使用 var

#### nil
可以使用 nil 给可选变量表示没有值：

    var serverResponseCode: Int? = 404
    // serverResponseCode 包含一个可选的 Int 值 404
    serverResponseCode = nil
    // serverResponseCode 现在不包含值

需要注意的是，nil 不能用于非可选的常量和变量。

如果声明的可选常量或变量没有赋值，默认设置为 nil。

*Swift 的nil和 Objective-C 中的nil并不一样。在 Objective-C 中，nil是一个指向不存在对象的指针。在 Swift 中，nil不是指针——它是一个确定的值，用来表示值缺失。任何类型的可选状态都可以被设置为nil，不只是对象类型*

#### 隐式解析可选类型
上面我们学习过了，可选类型暗示了常量或者变量可以 “没有值“，然后在实际处理中通过 if 语句来判断，但是很多情况下，一开始是没有值的，后来我很确定有值了，如果仍然像之前那样子判断，是一种很低效的方法。

这种类型的可选状态定义为 `隐式解析可选类型（implicitly unwrapped optionals）`，将可选类型后面的 ？ 改为 ！ 即表示我当前声明的时一个隐式解析可选类型。

隐式解析可选类型可以当做非可选类型处理，也可以当做可选类型处理。

### 断言
可选类型判断值是否存在，然后在代码中优雅的处理值确实的情况，然而有的时候值缺失或者不满足条件的情况下，代码可能无法继续执行，这个时候可以使用断言（assertion）结束代码运行并通过调试找到值缺失的原因。

#### 使用断言进行调试

断言即在运行时判断逻辑条件，可以附加一条调试信息：

    let age = -3
    assert(age >= 0,"年龄不能小于 0 ")

在执行到这些代码时，如果断言被满足，即 age >= 0 ,那么代码继续执行，如果断言不满足，那么停止运行并且输出后面的这句调试信息。

#### 什么时候使用断言
当你认为某个条件可能为假时，可以使用断言来调试，一般有以下场景：

- 整数类型的下标索引可能太大太小
- 给函数传值，但是非法的值会导致函数不能正常执行
- 一个可选值现在为 nil ，但是之后的代码需要一个非 nil 的值

> 在测试应用时很方便，上线 AppStore 时，还是需要将其注释掉











