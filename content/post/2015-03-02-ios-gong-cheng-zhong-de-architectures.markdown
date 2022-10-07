---
layout: post
title: "iOS 工程中的 Architectures"
date: 2015-03-02 11:05:58 +0800

---

## Arm 是什么
Arm 处理器是 Arm 公司面向市场推出的 RISC(精简指令集) 微处理器，目前手机上的 CPU 都是 RISC 类型，跟传统 PC 端的 CPU(CISC) 相比，具有体积小、低功耗、低成本、高性能的特点，很适合于移动终端。一般电脑上的 CPU 都是 CISC 类型的，iOS 模拟器是将 Arm 指令转换后在 X86 上运行。另外，Apple 本身就是 ARM 最大的股东之一。

<!--more-->

## Apple 系列产品的 Arm 架构

Armv6 的设备有 

`iPhone 1,2,3 代`、 `iPod 1,2 代`  

Armv7 的设备有 

`iPhone 3GS`、`iPhone 4`、`iPhone 4s`、

`iPad`、`iPad 2`、`(iPad 3)the new iPad`、`iPad mini`、

`iPod touch 3G`、`iPod touch 4`

Armv7s 的设备有 

`iPhone 5`、`iPhone 5c`
`iPad 4`

Arm64 的设备有 `iPhone 5s`、`iPhone 6`、`iPhone 6 plus`

`iPad mini 2`、`iPad mini 3`、`iPad Air`、`iPad Air 2`

可以看到，再以后的产品线中，应该全部都是支持 64 位了，如果是这样子，App 如果不支持64位，那么处理器再厉害也是白搭，性能损耗太大，苹果今年（2015）有关新上线 App 必须支持64位的决定也是有一定道理的。有关 App 如何支持64位的博客我会尽快的翻译官方的 GUI。

## Xcode 中的 Architectures 配置

Xcode 中有关指令集的配置如下图所示：

![Architectures Xcode 截图](http://7vzucb.com1.z0.glb.clouddn.com/blog.Architectures.png)



####Architectures:

表示工程将被编译成支持哪些指令集，支持指令集是通过编译生成对应的二进制数据包实现的，如果选择支持多个指令集，会造成编译出的数据包很大。Xcode 6 目前的默认设置为 `$(ARCHS_STANDARD)(armv7,arm64) ` ,armv7 包含了 Apple 的绝大数产品线，不包含的基本上已经被市场淘汰了，但是如果选择 armv7s，那么支持的 iOS 设备将会减少很多，另外 armv7 和 armv7s 性能差异比较小，所以 Apple 选择了 armv7，不排除以后 Xcode 升级修改到 armv7s。还有一个 arm64，这个就是让支持64位的设备全部运行64位数据包，由于64位与32位之间的性能差距比较大，再加上现在所有的 App 都需要支持64位，以后的 Xcode 配置中这一项不用改动，就按这个默认的 `$(ARCHS_STANDARD)` 来就行了。

####Build Active Architecture Only:

这个设置项是一个 BOOL 值，相当于一个开关，其意思是说，如果开关打开的话，我当前选择的设备是什么，我就编译跟设备匹配的二进制包。一般来说 Debug 模式下设置为 YES，这样子在测试 App 的时候，编译器只需要编译一种包，极大的减少了编译时间。Release 模式下设置为 NO，Release 模式主要是指打包为 ipa 分发测试或上线到 AppStore，这个时候你根本不知道安装 App 的设备是什么，当然是选择 NO，让编译起编译你所有想要支持的二进制包啦。

####Valid Architectures:

这里选择的是可能要编译的指令集，`Valid Architectures` 与 `Architectures` 指令集的交集，才会是 Xcode 最终编译出来的二进制包支持的指令集。如果此项为空，那么 Xcode 将不会编译任何东西。如果对 App 的体积有要求，可以在此项中设置最小的指令支持，但还是必须支持64位哦～

只要理解了 iOS 设备与指令集的关系，以后如果项目中遇到了指令集编译方面的错误，就可以迎刃而解了，尤其是使用第三方的 .a 静态库时，可能会出各种问题，其中很多问题都是指令集造成的，尤其是那些 _年代久远_ 的库。

#### 查看指定静态库所支持的指令集

比如要查看 libExample.a 这个静态库所支持的指令集，只需要 lipo 一下就好了

    lipo -info libExample.a
    
我自己在电脑上测试某 SDK 的库，得到的结果如下:

	Architectures in the fat file: output/Libxxx/libxxx.a are: armv7 armv7s i386 x86_64 arm64 
	
可以见得，在写 SDK 时，最好是 **全部都支持** ，这样子才不会给使用 SDK 的开发者造成麻烦。
