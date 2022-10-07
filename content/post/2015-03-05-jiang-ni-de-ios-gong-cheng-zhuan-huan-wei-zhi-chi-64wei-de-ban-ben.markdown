---
layout: post
title: "将你的iOS 工程转换为支持64位的版本"
date: 2015-03-05 15:21:41 +0800

tags: [编译, 翻译]
---


总的来说，可以根据以下步骤创建一个同时支持32位和64位运行环境的 App:

1. 确保 Xcode 版本至少为 5.0.1
2. 打开你的项目，Xcode 提示适配你的工程，编译64位版本的时候可能会给你提出一些重要的警告或者错误。
3. 设置你的项目最低为 iOS 5.1.1, iOS 5.1 之前的系统不支持64位
4. 修改工程的编译指令集为 “Standard Architectures”，即 Xcode 默认版本，其中包括 ARMv7 和 ARM64
5. 更新你的 App 以支持64位运行环境，编译器发出的警告和错误能够引导你处理整个过程，但是 Xcode 不是万能的，你还需要根据本文档来做一些事情。
6. 在 64 位的真机上测试你的 App，模拟器也可以帮助你测试，但是可能有一部分问题只有在真机上才会出现。
7. 使用 Instruments 来分析内存消耗以及性能。
8. 提交支持32位和64位的 App

接下来的内容会陈列一些在转换过程中经常出现的问题，根据引导仔细检查你的代码。

<!--more-->


## 不要将指针转换为整型
有一些地方可能需要将指针转换为整型数据，但是需要确保接受转换的变量有足够大的位数。

例如，下面 __Listing__ 2-1 的代码，将一个指针变量转换为 int 类型，在32位环境下没有问题，因为他们的长度都是4字节，但是在64位环境下会发生截取。

__Listing__ 2-1 Casting a pointer to int

	int *c = something passed as a argument...
	int *d = (int *)((int)c + 4);	// 错误的！
	int *d = c + 1;					// 正确
	
	int *d = (int *)((uintptr_t)c + 4);	// 正确

	
如果你必须要将指针转换为整型，可以使用 `uintptr_t` 类型来避免发生截取。需要注意的是将指针转换为整型做一些运算后又转换为指针违反了一些基本的数据类型规则。处理器使用这个指针去访问的时候可能会导致一个不可预估的行为。

## 保持数据类型的一致性
当你在编码的时候，不注意数据类型的一致性时，会发生各种错误或者警告。编译器会提出许多警告，你需要在代码中意识到这些问题。

当调用一个方法的时候，方法接收的参数应该和调用时传入的参数匹配，如果方法返回的数据类型要比接收的变量大，那么就会发生数据截取。 __Listing__ 2-2 使用了一个简单的例子来陈述这个问题，`PerformCalculation` 方法返回了一个 `long integer` ，在32位环境下，long 和 int 都是32bit，即使这种代码不正确，这种赋值也是不会出错的。但是在64位环境下，赋值的时候，高位的32bit将会被截取。

__Listing__ 2-2 Truncation when assigning a return value to a variable

	long PerformCalculation(void);
    int  x = PerformCalculation(); // incorrect
    long y = PerformCalculation(); // correct!

    
当作为参数传递的时候也有可能出现这种错误，例如 __Listing__ 2-3:

__Listing__ 2-3 Truncation of an input parameter

	int PerformAnotherCalculation(int input);
      

    long i = LONG_MAX;
    int x = PerformCalculation(i);


在 __Listing__ 2-4 中，下面的例子也是错误的，因为返回的值的范围超过了方法原型中的返回类型


__Listing__ 2-4 Truncation when returning a value

	int ReturnMax()
	{
    	return LONG_MAX;
	}

	

这些例子都是因为代码假定 `int` 类型和 `long` 是相同的。ANSI C 标准不认同这种假设，在 64位运行环境下是错误的。默认情况下，你适配了你的工程，那么 `-Wshorten-64-to-32` 编译器选项是自动开启的，所以当发生数据截断的时候编译器会自动的提示你。如果没有自动适配，那么你需要开启这项，根据需要，你可能想要开启 `_Wconversion` 选项，它更加详细且能发现更多潜在的问题。


在 Cocoa Touch  框架下开发的 App，查找下面的整型数据类型，确保你是正确的使用它们：

- long

- NSInteger

- CFIndex

- size_t ( `sizeof` 操作的返回值)


并且在所有的运行环境中， `fpos_t` 和 `off_t` 类型都是64位，所以千万不要将它们赋值给 `int` 类型。 



## 枚举也是一种类型

在 LLVM 编译器中，枚举类型能够定义枚举变量的大小，这意味着可能一些枚举类型可能比你预期的要大，在这种情况下不要对数据类型的大小做出任何假设，将枚举类型复制给变量时一定要选用适合的数据类型。

## Cocoa Touch 中常见的类型转换问题

Cocoa Touch,尤其是 Core Foundation 和 Foundation ，你需要额外的关注，因为它们提供了从 C 类型到 Objective-C 的转换。

__NSInteger__ __changes__ __size__ __in__ __64-bit__ __code__.
`NSInteger` 类型的使用遍布了整个 Cocoa Touch ，在32位环境中是32bit，在64位环境中增到到了8bit，所以如果从系统的 API 中接收到了 NSInteger 类型的值，也请使用 NSInteger 来保存这个值。

即使你从未假设 `NSInteger` 和 `int` 具有相同的大小，下面一些重要的例子你也需要看一下：

- 转换 `NSNumber` 对象的时候
- 使用 `NSCode` 类进行编码和解码的时候，尤其是将 `NSInteger` 类型在64位设备上编码，又在32位设备上解码的时候，这会造成数据溢出。你应该使用明确的整型变量替换它。
- 在 Framework 中使用 NSInteger 定义一些常量，比如 `NSNotFound` 常量,在64位环境中，它的值比 int 类型的最大值还要大，所以截断它的值常常会发生一些错误。

__CGFloat__ __changes__ __size__ __in__ __64-bit__ __code__:

`CGFloat` 在64位环境中是 8bit，你不能假定 `CGFloat` 等同于 `float` 或者 `double`,所以始终使用 CGFloat 吧。
__Listing__ 2-5 举了一个使用 Core Foundation 创建 CFNumber 的例子，但是代码错误的假定了 CGFloat 和 float 具有相同的大小。

__Listing__ 2-5 Use CGFloat types consistently

	// 错误！
	CGFloat value = 200.0;
	CFNumberCreate(kCFAllocatorDefault, kCFNumberFloatType, &value);

	
	// 正确～
	CGFloat value = 200.0;
	CFNumberCreate(kCFAllocatorDefault, kCFNumberCGFloatType, &value);

## 注意整型计算

尽管数据被截取是一个常见的情况，但也可能碰到其他与整型有关的问题。下面的指南会引导你修改你的代码。

### C 语言和基于 C 语言的其他语言的符号位拓展规则

C 语言和其他一些类似的语言，使用了一套赋值扩展规则，用来决定当整型的值转换为位数比较大的变量时,符号位的拓展规则。

1. 当从一个小值扩展到大值时，无符号的值用 0 扩展。
2. 有符号的值扩展时，符号位跟着扩展
3. 常量(除非带后缀，例如 0x8L) 按照最小位数来对待，Nsnumber 对象被编译器认为是有符号或者无符号的 int、long、long long。小数一致被当作有符号类型。
4. 一个具有符号位的值和相同位数无符号值的和仍是无符号的。

请看下面的例子：

	int a = -2;
	unsigned int b = 1;
	long c = a + b;
	long long d = c;
	
	printf("%lld\n",d); 
	
`问题`： 当运行在 32 位系统上时，结果是 －1 （0xFFFF FFFF）64位上为 4294967295（0x0000 0000 FFFF FFFF）

`原因`：有符号值与无符号的值相加，是一个无符号的数，当被赋值给一个更大的数时，导致符号位没有拓展

`解决办法`：在32位上，使 b 转换为 long 类型，这样子使得 b 扩展到 64 位。

下面是另一个与之有关的错误例子：

	unsigned short a=1;
	unsigned long b = (a << 31);
	unsigned long long c=b;
	printf("%llx\n", c);

`问题`： 当运行在 32 位系统上时，结果是 0x8000 0000，但是在64位上为 0xFFFF FFFF 8000 0000

`原因`：为什么符号位做了扩展？首先，当左位移操作被调用时，a 变量扩展到 int 类型，因为所有 short 类型的数值，都能正常转换为有符号的 int，这样值就是有符号的了。

第二步，当左移操作完成时，这个值存储到 long 类型，在 32位 中，long 和 int 是匹配的，所以不会有问题，但是在 64位 上，会做有符号位的扩展。

`解决办法`：在左移前，转换 a 为 long 类型。	

### 位和位掩码

在 64位的值中使用位和掩码时，你需要避免不小心带入 32位的值，下面是一些提示：

`不要认为数据类型拥有固定的长度` 如果你转换一个存储在 long integer 类型中的 bits 来转移，那么使用 LONG_BIT 值来指明 long integer 中有多少位。超过了变量的长度之后的转换结果与体系结构有关系。

`在必要的时候使用倒掩码` 在对 long 类型使用位掩码的时候需要特别小心，因为它在32位下和64位下是不同的，下面有两种创建掩码的方式，取决于你是使用零位扩展还是符号位扩展。

1. If you want the mask value to contain zeros in the upper 32 bits in the 64-bit runtime, the usual fixed-width mask works as expected, because it will be extended in an unsigned fashion to a 64-bit quantity.

2. If you want the mask value to contain ones in the upper bits, write the mask as the bitwise inverse of its inverse,如下所示.


Using an inverted mask for sign extension:

	function_name(long value)
	{
    	long mask = ~0x3; // 0xfffffffc or 0xfffffffffffffffc
	    return (value & mask);
	}


## 创建数据结构时使用固定的大小和对齐方式

当数据被同时用于 32位 和 64位下时，需要使得它们的表现相同，还需要注意的是，用户有可能在 32位环境下存入数据，在64位环境上使用数据。


### 使用明确的整型数据

C99 标准提供了内建的数据类型保证了其大小的固定，	而不用理会底层的硬件和体系结构。你需要使用这些数据类型。同时也需要注意避免内存浪费。

下面是 C99 的标准

<table>
 <tr>
  <td>Type</td>
  <td>Range</td>  
 </tr>
 <tr>
  <td>int8_t </td>
  <td>-128 to 127</td>
 </tr>
 <tr>
  <td>int16_t</td>
  <td> -32,768 to 32,767</td>
 </tr>
 <tr>
  <td>int32_t</td>
  <td> -2,147,483,648 to 2,147,483,647</td>
 </tr>
 <tr>
  <td>int64_t</td>
  <td> -9,223,372,036,854,775,808 to 9,223,372,036,854,775,807</td>
 </tr>
 <tr>
  <td>uint8_t </td>
  <td>0 to 255</td>
 </tr>
 <tr>
  <td>uint16_t</td>
  <td> 0 to 65,535</td>
 </tr>
 <tr>
  <td>uint32_t</td>
  <td> 0 to 4,294,967,295 </td>
 </tr>
 <tr>
  <td>uint64_t</td>
  <td> 0 to 18,446,744,073,709,551,615</td>
 </tr>
</table>


## 注意 64位的对齐方式

在64位运行环境中，字节对齐从 4 bit 增加到了 8bit，即使你明确指定了每个类型的长度，但是在不同的运行环境中仍然可能不同。


Alignment of 64-bit integers in structures

	struct bar {
	    int32_t foo0;
	    int32_t foo1;
	    int32_t foo2;
	    int64_t bar;
	};

当这段代码在 32位环境下编译时，bar 的结构是从一开始的 12 个字节，在 64位下，是从一开始的 16 个字节，这是由于不同的字节对齐导致的。

32位下：

	foo0 4字节 0x0000 0000 开始
	foo1 4字节 0x0000 0004 开始
	foo2 4字节 0x0000 0008 开始
	bar  8字节 0x0000 0012 开始
	 	
64位下：
	
	foo0 4字节 0x0000 0000 开始
	foo1 4字节 0x0000 0004 开始
	foo2 4字节 0x0000 0008 开始
	bar  8字节 0x0000 0016 开始 // 需要在之前填充 4 个字节

	
如果需要重新定义一个结构体，使得对齐字节比较大的的元素在前面，比较小的在后面，这样子会减少填充字节。如果为了向后兼容，可以使用程序强制 32位 下的对齐，如下：

Listing 2-10 Using pragmas to control alignment

	#pragma pack(4)
	struct bar {
	    int32_t foo0;
	    int32_t foo1;
	    int32_t foo2;
	    int64_t bar;
	};
	#pragma options align=reset
	
上面的做法是实在没办法才去使用的，因为这样子会有性能上面的损失。

## 使用 sizeof 分配内存空间

永远不要使用固定的数字作为 Malloc 的参数，应该使用 sizeof。因为你无法确定到底是多大。

## 修改格式化字符串来支持不同的平台

为了解决不同平台下的问题，可以使用 pointer-sized 整型（uintptr_t）或其他一些标准类型，都定义在 inttypes.h 头文件中。


#####Standard format strings

<table>
	<tr>
		<td>Type</td>
		<td>Format string</td>		
	</tr>
	<tr>
		<td>int</td>
		<td>%d</td>		
	</tr>
	<tr>
		<td>long</td>
		<td>%ld</td>		
	</tr>
	<tr>
		<td>long long</td>
		<td>%lld</td>		
	</tr>
	<tr>
		<td>size_t</td>
		<td>%zu</td>
	</tr>
	<tr>
		<td>ptrdiff_t</td>
		<td>%td</td>
	</tr>
	<tr>
		<td>any pointer</td>
		<td>%p</td>
	</tr>		
<table/>	


#####Additional inttypes.h format strings (where N is some number)

<table>
 <tr>
  <td>Type</td>
  <td>Format string</td>
 </tr>
 <tr>
  <td>int[N]_t (such as int32_t)</td>
  <td>PRId[N] (such as PRId32</td>
 </tr>
 <tr>
  <td>uint[N]_t</td>
  <td>PRIu[N]</td>
 </tr>
 <tr>
  <td>int_least[N]_t</td>
  <td>PRIdLEAST[N]</td>
 </tr>
 <tr>
  <td>uint_least[N]_t</td>
  <td>PRIuLEAST[N]</td>
 </tr>
 <tr>
  <td>int_fast[N]_t</td>
  <td>PRIdFAST[N]</td>
 </tr>
 <tr>
  <td>uint_fast[N]_t</td>
  <td>PRIuFAST[N]</td>
 </tr>
 <tr>
  <td>intptr_t</td>
  <td>PRIdPTR</td>
 </tr>
 <tr>
  <td>uintptr_t</td>
  <td>PRIuPTR</td>
 </tr>
 <tr>
  <td>intmax_t</td>
  <td>PRIdMAX</td>
 </tr>
 <tr>
  <td>uintmax_t</td>
  <td>PRIuMAX</td>
 </tr>
</table>

例如需要打印一个 `intptr_t` 的变量，如下

	#include <inttypes.h>
	void *foo;
    intptr_t k = (intptr_t) foo;
    void *ptr = &k;
    printf("The value of k is %" PRIdPTR "\n", k);
    printf("The value of ptr is %p\n", ptr);
    
##注意函数和函数指针

函数调用在 32 位和 64位的处理上是不一致的，关键在于在调用固定参数方法和可变参数方法使用不同的指令读取参数。下面展示了两种不同的方法原型，第一个方法 `fixedFunction` 总是使用一对整数。第二个方法使用至少两个的可变参数，在32位下，方法调用的指令时相同的，但是在 64位下是不同的，因为使用了一些转换。

64-bit calling conventions vary by function type

	int fixedFunction(int a, int b);
	int variadicFunction(int a, ...);
	int main {
    	int value2 = fixedFunction(5,5);
    	int value1 = variadicFunction(5,5);
	}

在64位下的调用约定变的更加精确，所以要确保你调用是正确的。并且调用者总能找到被调用的方法。

### 一定要定义方法原型

在使用最新的工程配置来编译的时候，如果你企图调用不太明确的方法，编译器会提示一些错误。你必须要提供一个方法原型，这样编译器才知道它是可变的方法或者别的。

### 方法指针必须使用正确的类型

如果在代码中传递方法指针，那么它的调用约定必须是保持一致的。必须要保持同样的方法参数。不要将可变方法转换为不可变方法。下面的例子就是一个错误的方法调用，因为方法调用使用了不同的调用规则，调用者传入的参数并不是方法所希望的。

Casting between variadic and nonvariadic functions results in an error

	int MyFunction(int a, int b, ...);
	int (*action)(int, int, int) = (int (*)(int, int, int)) MyFunction;
	action(1,2,3); // Error!

	
> 如果你在自己的代码中使用到了如上的方法转换，编译器可能不会给你警告或者错误，在模拟器上测试可能也不会出现问题，所以一定要在真机上进行测试。

### 使用方法原型进行消息分发

转换规则有一个特例如标题描述，就是在使用 `objc_msgSend` 方法或者 OBJC 运行时其他的一些消息传递方法。尽管这些方法是可变参数的形式，但是 Objc 运行时的消息传递调用这些方法时不会共享相同的原型。Objc Runtime 直接调用到这些方法的具体实现，所以就如之前所说，这些调用规则是不适用的。所以需要在调用之前转换 `objc_msgSend` 方法为一个原型匹配的方法。

下面的例子展示了如何使用底层的消息调用分发消息到另一个对象。在这个例子中，`doSomething:` 方法使用了单个参数，不是可变参数。需要注意的是 method function 总是使用 id、SEL 作为前两个参数。当 `objc_msgSend` 转换为函数指针后，调用会经过相同的函数指针。

Using a cast to call the Objective-C message sending functions

	(int) doSomething:(int) x { ... }
	- (void) doSomethingElse {
	    int (*action)(id, SEL, int) = (int (*)(id, SEL, int)) objc_msgSend;
	    action(self, @selector(doSomething:), 0);
	}

### 调用可变参数的方法时需要特别小心

可变参数列表 (`varargs`) 不会提供参数的类型信息，而且不会自动提升到大的类型。如果你需要区分传入数据的类型，你有望使用格式化的字符串或者其他类似的机制来提供 ` varargs` 方法的信息。如果调用的方法没有提供正确的信息，你可能得到错误的结果。

尤其是你的 `varargs` 方法希望一个 long integer 而你传入了一个 32bit 的值，那么很有可能方法接受了这个32bit 的值，又从下一个参数中读取了 32bit 的垃圾。同样的，如果方法希望传入 long ，而你传入了 int ，也会造成错误。

### 千万不要直接保存 Objective-C 指针

如果你的代码直接保存对象的 isa 字段，那么在64位环境下出错。isa 字段不再保存一个指针。做为替代，它保存了指针以及其他一些运行时的信息。这种优化提高了内存使用率以及性能。

使用 `class` 原型或着调用 `object_getClass` 读取 isa 字段。使用 `objec_setClass` 写入。

这些问题也不会出现在模拟器中，需要在真机中测试。

### 使用内置的同步原语

有的时候，App 为了提高性能实现了自己的同步原语。iOS Runtime 环境提供了一整套的原语，并且对于每一种 CPU 做了优化。Runtime 文档也做了更新，总之，还是使用内置的原语吧。

### 不要硬编码改变虚拟内存页面的大小

通常是不需要知道页面大小的，如果是为了做缓存分配或者其他动态库调用，使用 `getpagesize( )`来获取。

### 使用位置无关代码

64位环境仅仅支持 `独立位置的可执行区域 PIE` , 默认情况下 App 的编译都是使用位置无关，如果你有一些代码使得你无法使用位置无关，比如一些静态链接库或者汇编代码，那么你需要改变它。

### 在 32 位环境下也要 Run Well

当前，支持64位环境的 App 也是支持 32 位的，最好就是一份设计支持两种环境，当然偶尔需要分别对待。

举个🌰，你可能为了方便在整个代码中使用 64bit 整型，两种环境都支持 64bit 整型，并且能够简化你 App 的设计，但是在 32bit 环境下可能会慢一些。如果你的计算在32位下是足够的，那么还是使用 32bit。


# 优化内存性能

由于 64位 编译器的支持，64位的 App 能够运行的更快一些，但是同样，占用的内存也会大一些，为了减少内存压力，你需要对你的 App 做一些内存方面的优化。

## 剖析你的 App

在对你的 App 进行内存优化之前，你首先需要创建一些标准测试同时跑在 32bit 和 64bit 的环境下，来量化内存性能的损耗，以及量化你的优化结果。至少需要一个轻量化的测试用例，以及很多复杂的测试用例。这些测试的目标是量化内存消耗的具体位置。

## 常见的内存使用问题

下面是一些常见的内容使用问题以及解决办法。

### Foundation 的对象对于小的载体可能太过

许多 Foundation 框架下的类提供了一个灵活的集合，灵活性跟其他简单的数据结构相比较消耗了比较大的内存，举个🌰，使用 `NSDictionary` 对象来保存简单的 `key－value` 键值对就比简单的变量保存要消耗更多的内存。如果是大量这样的 NSDictionary 对象，那么就会占用更大的内存，尤其是在 64 位下。所以这种对象需要适合的使用。

### 使用紧凑的数据描述

在定义数据结构的时候，不同的顺序会有不同的大小，这跟字节对齐有关系。

	struct date {
	    NSInteger second;
	    NSInteger minute;
	    NSInteger hour;
	    NSInteger day;
	    NSInteger month;
	    NSInteger year;
	};

	
上面的结构体在32位下 24个字节，64下 48字节。改为下面的结构就简单多了

	struct date {
	    time_t seconds;
	};

需要注意的是， `time_t` 数据类型在 32bit 和 64bit 下的大小是不一样的。


### 整理结构体

为了数据对齐，编译器会增加一些空内存。例如：

	struct bad {
	    char       a;	// offset 0 
	    int32_t    b;	// offset 4
	    char       c;	// offset 8
	    int64_t    d;	// offset 16
	};	

这个结构体只有 14字节的数据，但是由于空内存块提升到了 24字节

下面是一个比较好的例子：


	struct good {
	    int64_t    d;	// offset 0
	    int32_t    b;	// offset 8
	    char       a;	// offset 12;
	    char       c;	// offset 13;
	};

### 少用指针


避免滥用指针，思考下面的实现：

	struct node {
	    node        *previous;
	    node        *next;
	    uint32_t    value;
	};	

32位编译下，12个字节中只有 4字节用作有效数据，在64位下，有效数据占用了 20%，所以思考用数组或者其他方式来实现吧	


### 字节对齐导致的内存分配

当你直接调用 malloc 方法的时候（例如 objctive-C 对象 alloc 的时候），操作系统会额外申请一些内存空间来维持字节对齐。当初始化 C 结构体的时候，申请几个大的内存块比为每一个结构体单独申请要有效率的多。


### 只有在需要的时候才去缓存

缓存之前的数据或者计算结果是一种常见的提高性能的方式。当然，你仍然需要查看缓存是否真正的提高了你 App 的效率。如之前所说，64位下的内存消耗要比32位大的多，如果你的 App 依赖于太多的缓存策略，过多的使用虚拟缓存可能反而会降低性能。


避免以下典型的错误例子:

- 缓存一些类能够快速创建的数据
- 缓存一些能够快速从其他对象获取到的对象或者数据
- 缓存系统可以快速创建的对象
- 缓存一些能够快速映射到内存的只读数据


一定要通过测试确保缓存提高了 App 的性能，使用钩子来可以选择性的开启某些缓存策略，使用不同的数据集合来验证你的缓存算法。

>更新时间： 2014-02-11