---
layout: post
title: 使用 iOS 中不一样的 NSNotification Center
date: 2017-09-09 12:14:36 +0800

---


通知中心利用观察者模式，能够很方便地进行一对多的通知，通俗点称作广播。

NSNotification default Center 内部维护了观察列表，并负责通知的派发。

使用 Notification 并不难，Foudation 提供了三个 class：

- NSNotification：通知的数据结构，包含通知的具体数据，包含通知名 name，一个对象 object 和一个可选的字典 userInfo。
- NSNotification Center：负责派发 notification 并维护观察者列表。每个 App 都有一个默认的实现，即 `[NSNoficationCenter defaultCenter]`。
- NSNotificationQueue：一个缓冲队列，可以在队列中过滤、合并通知。每个线程都有默认的 queue，即 defaultQueue。


<!-- more -->

## 接收和发送通知

### 接收通知

iOS 系统默认提供了一百多种通知，方便我们使用。实现一个接收通知的代码很简单，例如当前的 ViewController 需要在 App 即将进入后台时对页面进行高斯模糊，并且设置下次 App 恢复到前台时要求登陆密码。很多银行类 App 都有这种需求。

```Objc
- (void)viewDidLoad {
  // 向 Notification Center 添加观察者
  	UIImage *currentScreenImage = ...;
    [[NSNotificationCenter defaultCenter] addObserver:self selector:@selector(appDidEnterBack:) name:UIApplicationDidEnterBackgroundNotification object:currentScreenImage];
    
}

- (void)appDidEnterBack:(NSNotification *)notification
{
    notification.name;
    notification.object;
    notification.userInfo;
    
    // 在当前 view 上添加模糊图
  	// 开启登陆验证
    
}
```

当 App 进入后台时，NotificationCenter 发现我们这个 ViewController 注册了该通知，就会调用 `appDidEnterBack` 方法。可以在方法中访问 notification 的三个属性获取额外信息。

利用通知一般需要三个步骤：

### 添加 Observer

observer 就是要接受通知的对象，通知最后会发送到该对象上，即调用该对象在注册通知时的 selector。

通常会在 Notification Center 中添加该 Observer，并且传入需要观察的通知的 name（标识符）。另外还有一个 object 参数可供我们使用，一般 object 表示谁发出这个通知，即通知的发起方。

object 对象在处理多个同名通知时有用，比如当前 ViewController 有多个 UITextField，它们都发出 UITextFieldTextDidChangeNotification 这个通知，那么能够正确地区分每一个 TextField 将会使我们的工作简单很多。

### 处理通知

Obeserver 在被添加到 Notification Center 后，会持续不断地接收通知，除非从 NotificationCenter 中移除。

NotificationCenter 会在通知发生时调用 ``addObserver:selector:name:object:` ` 方法中的 selector，方法被掉起时的所在的线程与通知被 post 时一致。

还可以将通知处理的方法放在 block 中，提升阅读性：

```objc
- (id <NSObject>)addObserverForName:(nullable NSNotificationName)name object:(nullable id)obj queue:(nullable NSOperationQueue *)queue usingBlock:(void (^)(NSNotification *note))block API_AVAILABLE(macos(10.6), ios(4.0), watchos(2.0), tvos(9.0));
    // The return value is retained by the system, and should be held onto by the caller in
    // order to remove the observer with removeObserver: later, to stop observation.

```

### 移除 Obeserver

当不需要再对通知响应时，就需要将其移出，不然会因为给已经 dealloc 的实例发送方法而遇到 Crash。

```objc
- (void)removeObserver:(id)observer name:(NSNotificationName)aName object:(id)anObject;
- (void)removeObserver:(id)observer;
```

NSNotification 提供了上述两个方法。前者是移出指定的通知，后者是移出某个 observer 的所有通知。常用的做法是在 delloc 中移除该 observer 的所有通知，对象都销毁了，自然就无需监听通知了。

### 发送通知

可使用默认的通知中心发送通知，指定通知名 name 与发送方 senderObject 以及参数字典 userInfo即可。后面两个参数是可以省略的。

```objective-c
- (void)postNotificationName:(NSNotificationName)aName object:(id)anObject userInfo:(NSDictionary *)aUserInfo;
```

通知会在这个发起线程里，向所有已注册该 notification 的 observer 发送，这个顺序并不是确定的。通知是被同步发出，也就是说通知被发送且处理完成之后，post 方法才会返回。

### 通知的命名规则

按照编码规范，通知命名遵循如下标准：

> NameOfAssociatedClass +  [ Did | Will ] + [ UniquePartOfName ] + Notification

例如：ReigsterControllerDidPassedNotification

### 自定义通知

有时默认的 NSNotification 不能满足我们的要求，或者会产生很多胶水代码：

```objective-c
NSNotification *noti = [NSNotificaiton new];
noti.name = ...
noti.object = ...
noti.userInfo = @{
  @"#1" : ...,
  @"#2" : ...,
  @"#3" : ...,
};

// 使用 Category，以及计算属性，使得 Notification 可定制化
// 属性使用 readonly 声明，防止被修改
NSNotification *noti = [NSNotificaiton withUser:user];

@interface NSNotification (UserObserver)
@property (nonatomic, readonly) CVLUser *hpe_audioPlayer;
@property (nonatomic, readonly) NSTimeInterval hpe_endTime; 
@end
 
@implementation NSNotification (HPEAudioPlayerObserver)
 
- (HPEAudioPlayer*)hpe_audioPlayer {
    return self.object;
}
 
- (NSTimeInterval)hpe_endTime {
    return [[self.userInfo objectForKey:HPEAudioPlayerNotificationEndTimeKey] doubleValue];
}
 
@end

```

## 对 Notification 进行单元测试

查看  http://ocmock.org/reference/#observer-mocks 即可。

## 异步发出通知

NotificationCenter 是同步处理通知的，即通知发出后会立刻处理通知。如果通知特别频繁的话，那么对性能的影响比较大，同时也容易出现一些问题，比如频繁地更新 UI。

Foundation 提供了 NSNotificationQueue 用于处理这种场景，能够将通知放在队列中，合并或者延迟调用。NSNotificationQueue 与 Runloop 息息相关，Notification Queue 以 FIFO 顺序持有 notifications，notification 在队列中出队时，会发送给关联的 Notification Center 中。

由于 Notifications 与 Runloop 相关，因此有一些坑：

- 队列中的 Notifications 可能不会按时发送
- 队列中的 Notifications 可能不会再被派发到 NotificationCenter 中（比如线程对应的 Runloop 退出，剩余的 Notifications 会被丢弃）

因此，无法保证 NotificationQueue 中的 Notification 一定会被 post，或者在某个特定的线程接收。

### 合并以及推迟通知

使用下面的方法可以在队列中添加一个通知：

```objective-c
- (void)enqueueNotification:(NSNotification *)notification 
  	postingStyle:(NSPostingStyle)postingStyle 
    coalesceMask:(NSNotificationCoalescing)coalesceMask 
    forModes:(NSArray<NSRunLoopMode> *)modes;
```

由于这个方法是异步的，因此会立刻返回。

`postingStyle ` 决定了通知被延迟的方式：

- NSPostNow：同步发出通知，也就是一添加到队列中，如果 coalescing 没有被指定，那么就会立刻执行。
- NSPostASAP：当前 Runloop 完成时执行，相当于一个 0 毫秒延迟的 timer。
- NSPostWhenIdle：只有当 Runloop 处于 wait 或者 idle 状态时，才会去处理。

延迟处理 notification 特别有用，可以避免大量 notification 出现时的窘境。例如在一个 socket 连接中，将大量的 notfication 中的数据合并后作为一个 notification 发出。

coalescing 有三种选项：

- NSNotificationNoCoalescing：不合并
- NSNotificationCoalescingOnName：合并相同 name 的通知
- NSNotificationCoalescingOnSender：合并相同的 sender




## 小结

Notification 用起来好，但是不能泛滥，coalescing 合并也不是万金油，在需要使用 coalescing 时选项时，可以考虑是否能用其他方式替代。