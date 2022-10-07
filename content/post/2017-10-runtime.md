---
layout: post
title: 随便说 iOS 中的 runtime
date: 2017-11-19 18:24:28 +0800
---


Objc 作为动态语言，可以将消息转发给想要的对象去处理。但是编译器能做的只有将代码转为汇编指令，然后固定的指令流水执行。因此在 Objecive-C 中，还有一套强大的 runtime 运行时系统，专门提供消息转发等运行时能力，还提供了大量的 API，例如 objc_send， class_get 等。



<!-- more -->



## OC 中的类与对象

Objective-C 中的类是 Class 类型，看下其定义：

```objc
// objc.h

/// An opaque type that represents an Objective-C class.
typedef struct objc_class *Class;

```

注释中也说的很清楚，Class 就是一个指向 objc_class 结构体的指针，那么类自然就是由这个结构体来描述来，跟进继续查看：

```objc
// objc/runtime.h

struct objc_class {
  	/**
  	*	类是一个抽象概念，描述类的数据结构时使用的是具体的实体，
  	*	因此可说所有的类本身也是一个对象，对象又一定是某个类的实例，
  	*	所以这里有个 isa 指针，指向了所属的类。
  	*	这里比较特殊，是 Class 的 isa 指向了 Class，所以称为 metaClass，元类
  	*/
    Class _Nonnull isa  OBJC_ISA_AVAILABILITY;

#if !__OBJC2__
    Class _Nullable super_class  	// 指向父类     
    const char * _Nonnull name  	// class name
    long version
    long info
    long instance_size  	// 类初始化后实例的大小
    struct objc_ivar_list * _Nullable ivars  	// 实例变量的列表
    struct objc_method_list * _Nullable * _Nullable methodLists  	// 类方法列表
    struct objc_cache * _Nonnull cache  	// 缓存最近使用过的方法
    struct objc_protocol_list * _Nullable protocols // 实现的协议列表
#endif

} OBJC2_UNAVAILABLE;
/* Use `Class` instead of `struct objc_class *` */

```

当 class 收到一条消息时，需要判断是否能够处理，查找时优先从cache 中找，找不到后从 method_list 中查找，找到后会添加到 cache 中。这也是一个 性能上的优化。

看完了类，看下对象：

```objc
/// Represents an instance of a class.
struct objc_object {
    Class _Nonnull isa  OBJC_ISA_AVAILABILITY;
};

/// A pointer to an instance of a class.
typedef struct objc_object *id;
```

对象也是用一个结构体来描述，内部比较简单，就是一个 isa 指针指向自己的类，因为 class 中保存了所有有关的信息，且这些信息都是对象共享的。

代码的第二部分，解释了 id 是从何而来，id 就是指向这个结构体的指针，所有 OC 中所有的对象都可以用 id 来保存。

## MetaClass 元类

创建对象可以使用类，但是如果我们想创建出类呢？那就必须根据metaclass创建出类，所以：先定义metaclass，然后创建类。

对象是可以接收方法调用的，为什么类也可以使用类方法呢？因为类本身也是一个对象，这个对象由 metaClass 实例化。它接收方法调用，既然类是对象（之后称作类对象），那么就必然由一个 isa 指针指向该类对象的 Class。这个 Class 就是元类 metaClass。



![128529-b2aeea6630b9514a](/images/128529-b2aeea6630b9514a.jpg)

对于实例方法，如上图紫色的顺序查找方法。

对于类方法，按照绿色顺序查找方法。对于 Object，该类的 metaClass 是自己，形成闭环。

## 通过 YYModel 看 Runtime API

既然理解了 Class 是一个结构体指针，metaClass 描述 Class，那么就可以比较顺手地去使用 runtime 提供的强大 API 了。下面拿 YYModel 中的一些代码举例

```objc
/// Get the 'NSBlock' class.
static force_inline Class YYNSBlockClass() {
    static Class cls;
    static dispatch_once_t onceToken;
    dispatch_once(&onceToken, ^{
        void (^block)(void) = ^{};
        cls = ((NSObject *)block).class;
        while (class_getSuperclass(cls) != [NSObject class]) {
            cls = class_getSuperclass(cls);
        }
    });
    return cls; // current is "NSBlock"
}

```

`class_getSuperclass` ： 获取 cls 的 superClas。

例如，使用 YYModel 通过 Dictionary 生成 CVLModel 的对象：

```objc
// CVLModel 假设是我们自己的类，dict 是 server 返回的包
CVLModel *model = [CVLModel yy_modelWithDictionary:dict];
```

1 获取 CVLModel 的 Class 信息：

```objc
Class cls = [CVLModel class];
```

2 将 cls 中的信息，read 到自定义类 YYModelMeta 中，

```objc

_YYModelMeta *modelMeta = [_YYModelMeta metaWithClass:cls];

// metaWithClass 的具体实现中，涉及了大量的缓存操作提升性能，会将之前创建过的 metaClass 缓存起来，所以我们只关注下核心的一行代码

meta = [[_YYModelMeta alloc] initWithClass:cls];

```

最终会走到 YYClassInfo 的初始化方法中，在这里通过 runtime 的 API 获取 cls 的数据。

```objc

- (instancetype)initWithClass:(Class)cls {
    if (!cls) return nil;
    self = [super init];
  // 保存 cls 作为实例
    _cls = cls;
  // 获取父类的 cls
    _superCls = class_getSuperclass(cls);
  // 判断 cls 是否为 metaClass
    _isMeta = class_isMetaClass(cls);
    if (!_isMeta) {
      // 非 metaClass，获取当前类的 metaClass
        _metaCls = objc_getMetaClass(class_getName(cls));
    }
  // cls 的类名
    _name = NSStringFromClass(cls);
  // 
    [self _update];
	// 递归调用获取父类的 classInfo
    _superClassInfo = [self.class classInfoWithClass:_superCls];
    return self;
}

- (void)_update {
    _ivarInfos = nil;
    _methodInfos = nil;
    _propertyInfos = nil;
    
    Class cls = self.cls;
	// 开始获取方法列表
    unsigned int methodCount = 0;
    Method *methods = class_copyMethodList(cls, &methodCount);
    if (methods) {
        NSMutableDictionary *methodInfos = [NSMutableDictionary new];
        _methodInfos = methodInfos;
        for (unsigned int i = 0; i < methodCount; i++) {
          	// 将 method 也用 YYClassMethodInfo 保存起来
            YYClassMethodInfo *info = [[YYClassMethodInfo alloc] initWithMethod:methods[i]];
            if (info.name) methodInfos[info.name] = info;
        }
        free(methods);
    }
  	// copy 属性列表
    unsigned int propertyCount = 0;
    objc_property_t *properties = class_copyPropertyList(cls, &propertyCount);
    if (properties) {
        NSMutableDictionary *propertyInfos = [NSMutableDictionary new];
        _propertyInfos = propertyInfos;
        for (unsigned int i = 0; i < propertyCount; i++) {
            YYClassPropertyInfo *info = [[YYClassPropertyInfo alloc] initWithProperty:properties[i]];
            if (info.name) propertyInfos[info.name] = info;
        }
        free(properties);
    }
    
	// copy 实例变量列表
    unsigned int ivarCount = 0;
    Ivar *ivars = class_copyIvarList(cls, &ivarCount);
    if (ivars) {
        NSMutableDictionary *ivarInfos = [NSMutableDictionary new];
        _ivarInfos = ivarInfos;
        for (unsigned int i = 0; i < ivarCount; i++) {
            YYClassIvarInfo *info = [[YYClassIvarInfo alloc] initWithIvar:ivars[i]];
            if (info.name) ivarInfos[info.name] = info;
        }
        free(ivars);
    }
    
    if (!_ivarInfos) _ivarInfos = @{};
    if (!_methodInfos) _methodInfos = @{};
    if (!_propertyInfos) _propertyInfos = @{};
    
    _needUpdate = NO;
}
```

可以看出来，YYModel 使用  runtime API，将 CVLModel 拔了个精光。

## Runtime API

**object_getClass(id _Nullable obj)**

通过 obj 获取对应的 Class。这里传入对象，得到对象的类信息。传入类，也就是把类作为对象传入，那么返回的是该类的 metaClass。

**objc_getClass(const char * _Nonnull name)**

同样是获取 class 信息，这里传入的是该类的 name，当然这个 name 也可以通过 object_getClassName 传入一个对象获得。

**objc_getMetaClass(const char * _Nonnull name)**

获取 metaClass 信息，我认为等同于

```Objc
// 先获取普通 cls
Class cls = objc_getClass(clsName);
// 再获取 metaClass
Class metaCls = object_getClass(cls);
```



## 小结

通过 Objective-C 中类与对象的分析，再拓展到 runtime ，我们揭开这层神秘面纱，初探 runtime，可以看到 runtime 系统的厉害之处，再由 YYModel 的实现分析，学习如何动态创建对象并赋值。

之后在项目中有需求要实现时，了解 runtime 后，也许会帮我们打开新的思路。runtime 不可怕但也要保持谨慎敬畏，我们无法完全理解并使用，谁也不知道里面是否存在坑（比如就一个方法替换，坑就很多，网上也不乏关于这些坑的介绍以及如何避免）。


## 参考


1. [^类与对象]: http://southpeak.github.io/2014/10/25/objective-c-runtime-1/
2. [^YYModel]: https://github.com/ibireme/YYModel
3. [^Runtime 中关于MetaClass的问题]: http://www.jianshu.com/p/d0c6a3efb4d4

