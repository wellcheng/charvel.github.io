---
layout: post
title: "创建自己的cocoaPods - podspec 文件"
date: 2015-04-04 20:23:36 +0800

tags: [CocoaPods, 依赖]
---

> 现在很多开源的第三方库都支持 cocoaPods 来管理，这对于 iOS 开发人员来说是一个福音，基础的运用为创建 `podfile` 然后在文件中加入自己需要依赖的第三方库就可以啦。不过在实际的使用中，有可能需要依赖自己的库或者公司的库，或者有些库没有加入到 cocoaPods 中，这个时候就需要我们制定自己的 cocoaPods 库了。

<!--more-->

## CocoaPods 基本原理


### CocoaPods 的基本流程大概为：

1. 初始化一个 WorkSpace，将原有的 project 包含进来，同时创建一个专用于管理依赖的 project： `Pods` 。这个时候，你的项目应该是这样的结构：一个 WorkSpace 中有两个 project ，一个是你真正的项目，另一个是用于管理依赖的 Pods project。
2. 通过 git 下载 `Podfile` 文件中的第三方库，每一个库作为 Pods 项目的 target，编译生成一个以库名命名的静态库文件，例如 `AFNetworking.a`。
3. 将所有的 .a 文件合并为最终主项目依赖的静态库，名为 `libPods.a` 
4. 依据每个第三方库的.podspec 文件，将其被添加进工程中。.podspec文件可以标识该第三方库所需要的源码文件、依赖库、编译选项，以及其他第三方库需要的配置。
5. 最后，Pods 修改主项目中的编译、链接参数，让你的项目完美集成第三方库。

### 添加第三方库

当Cocoapods向项目中增加了一个第三方库的时候，不仅仅是将添加代码这么简单。由于每个第三方库有不同的target，所以每次添加第三方库时，都只有几个文件被添加。每个源代码都需要：

#### 一个包含编译选项的.xcconfig文件
这个文件主要描述当前的库需要哪些系统库的支持，举几个例子就很容易明白啦：

- AFNetworking：

		PODS_AFNETWORKING_OTHER_LDFLAGS = -framework "CoreGraphics" 
		-framework "MobileCoreServices" 
		-framework "Security" 
		-framework "SystemConfiguration"

- Base64：

		Base64 只提供了方法，不需要系统库支持，所以这个文件为空
		
- FMDB：

		PODS_FMDB_OTHER_LDFLAGS = -l"sqlite3"
		
- Reachability：

		PODS_REACHABILITY_OTHER_LDFLAGS = 
		-framework "SystemConfiguration"



#### 一个同时拥有编译设置和CocoaPods默认配置的私有.xcconfig文件

xcconfig文件正如其名字一样，就是xcode里的config文件。 我们在开发过程中，需要配置一些参数，这些都可以在xcode工程的setting对项目进行配置，xcconfig就是将这些配置项以文件的形式独立出来，方便共享与配置。比如两个项目用到相同的配置，那么只需要在xcode中选择对应的xcconfig文件即可，方便与灵活共享。

CocoaPods 需要这个文件来设置当前 target 的配置选项，从而正确的编译出静态库 .a 文件。


#### 编译所必须的prefix.pch文件
#### 另一个编译必须的文件dummy.m

一旦每个pod的target都完成了以上步骤，整个Pods的Target就会被创建。这增加了相同的文件，与另外几个。如果有源代码中包含了资源bundle，向app的target中添加bundle的方式将写入Pods-Resources.sh。还有一个叫Pods-environment.h的文件，文件中含有许多检查组件是否来自pod的宏定义。最后，将生成两个确认文件，一个.plist文件，一个用于给用户查阅许可信息的markdown文件。

> 待续