---
title: iOS 目录结构
date: 2016-09-24 22:38:13
---



最近整理了下工程的目录结构，有一些体会，现在写出来

<!--more-->

- Application：应用程序入口
- StoryBoards：在使用storyBoard进行页面布局时，要经常使用，因此将所有文件放在一起也是比较方便的。
- Categories：分类
- Controllers：所有的控制器，也可以理解为页面
	- Tab-HomePage：可以理解为第一个 TabBar，也是业务
	- Tab-Article-Feeds：消息盒子
	- Tab-Mine：我的页面
- Models：数据模型
- Helpers：助手类
	- Utils：工具类的一些脚手架代码，与 App 无关，放在哪里都能用
- Views：主要是放一些自定义的视图，在需要重用时用它
- Macros：整个应用都会使用的宏定义
	- Macros.h：全局点的定义
	- AppMacro.h：App 的需要使用的
	- NotificationMacro：通知模式下
	- UtilsMacro：工具类中用到的定义
	- VendorMacro.h：第三方的常量
- Vendor：第三方库
- Resource：资源文件


## Group 与 Folder
Group 是在 Xcode 中的布局方式，文件夹的颜色显示为黄色。
Folder 是真正的文件夹，可以与 Group 关联在一起。
一般来讲，做到第一层和第二层能够相关关联就可以，这样在提交代码或者通过 Git 管理代码时比较方便。



