---
layout: post
title: iOS 中的多线程
date: 2017-07-12 11:10:26 +0800

---



## 线程的使用

在常见的多线程模型中，进程即一个运行着多个线程的程序。

线程可以理解为一条独立的代码路径，在线程中，代码一行一行执行。

线程在使用上，iOS 推荐使用一些高级封装。例如 OperationQueue、GCD、NSNotificationQueue、系统自带的异步方法、定时器等。

iOS 和 macOS 底层都是基于 mach port 的，常用的仍然是 POSIX 模型的多线程 API。

线程可以拥有一个 runloop，负责监听事件，有事件发生就唤醒去处理，不然就休眠。

线程之间是可以共用所属进程的内存空间，所以跨线程通信的方法很多：

- 直接使用 performSelector 发送消息。
- 全局变量
- 条件变量
- Runloop source
- 端口和套接字
- 消息队列

<!-- more -->

## 常见的线程使用技巧

避免使用太底层的 API，使用高级的 API 处理多线程编程问题。

多使用 copy，让数据结构变为 inmutable ，比如 Swift 对于 struct 的使用。

在主线程中绘制 UI。

使用锁来保证数据竞争时的安全性。

## 线程管理

线程的创建需要在占用内核空间，以及 RAM 联动内存，是需要成本的。

> 联动内存：

大约是 1KB 的内核内存，512KB 的次级线程，时间不到 100ms。

因此使用线程池是比较高性能的。

线程创建时一般需要执行一个代码执行入口，以及相关的一些配置。

`NSThread` 的 `isMultiThreaded` 方法可以检查应用是否是多线程。

应用退出时，detached 线程可以立刻结束，但是 joinable 线程必须全都被 join 后进程才能退出。因此 joinable 线程适用于执行不能中断的重要工作，比如向磁盘写入数据。

线程的优先级设置不当容易产生优先级反转等 case，需要谨慎使用。

## Runloop

在 iOS 中线程与 Runloop 息息相关，Runloop 在线程中主要起到了`保活`的作用。一般线程在创建并开始执行任务，任务完成后就会销毁，但 Runloop 可以让线程休眠，并且提供了一套机制，当事件到达时重新激活线程处理任务，而在休眠期间，线程消耗的资源特别低。

每个线程都有对应的 Runloop，主线层默认已经启动了 Runloop，其他线程需要主动开启。

Runloop 对于事件分成了几种不同的 source：

- Port-Based Source（Source1）
- Custom  Input Source（Source0）
- performSelector：onThread：
- Timer Source

存在大量事件，那么就必然有优先级的划分，不然无法提升效率和灵活性，Runloop 将 source 区分为 Mode，只有 source 在当前 Runloop 的 Mode 中才会被处理。

Runloop 是可以被监听的，注册相对应的 notification 即可，

1. 进入 run loop
2. 当 run loop 即将处理一个 timer
3. 当 run loop 即将处理一个 input source
4. 当 run loop 即将休眠
5. 当 run loop 已经被唤醒，但在它处理唤醒它的事件之前
6. 退出 run loop

比如检测当前页面的滑动帧率 FPS，就是利用这个原理。

![](https://blog.ibireme.com/wp-content/uploads/2015/05/RunLoop_1.png)



上图是 Runloop 运行的大致流程。



## 多线程编程

线程同步是多线程编程中必然面对的问题，尽管引入多线程给编程带来了困惑与潜在的问题，但其巨大的性能提升仍然值得我们去研究并解决同步问题，毕竟办法还是比困难多一些。

处理同步问题有很多方式，比如内存屏障和 Volatile 变量，这些都是编译器与 OS 底层的处理，我们使用不多，只需要明白其大概原理即可。

### 锁

锁是最常用的工具之一，主要用于保护临界区代码。

将容易引发问题的代码片段放在临界区中，保证同一时刻只有一个线程可以访问这块区域的代码。

NSLock 的代码可以在 https://github.com/gnustep/libs-base/blob/master/Source/NSLock.m 中找到。

`NSLock` `NSRecursiveLock` `NSCondition` `NSConditionLock` 内部持有 pthread_mutex_t 类型的信号量作为 Lock 的实现, 只是 mutex 在初始化时，指定的属性不一样。

对于 lock 方法，`NSLock` `NSRecursiveLock` `NSCondition`几个锁都是默认的实现：

```objc
// 
- (void) lock {
	int err = pthread_mutex_lock(&_mutex);\、
	if (EINVAL == err) {
		[NSException raise: NSLockException format: @"failed to lock mutex"];      
    }
	if (EDEADLK == err) {
		_NSLockError(self, _cmd, YES);
    }
}

```




#### 互斥锁 Mutex

互斥锁是一种概念，主要是对于临界区的访问是互斥的，同一时刻只能被一个线程访问。

```objc
- (id) init
{
    if (nil != (self = [super init]))
    {
      // 创建 pthread_mutex_t 类型的信号量
        if (0 != pthread_mutex_init(&_mutex, &attr_reporting))
        {
            DESTROY(self);
        }
    }
    return self;
}


- (BOOL) lockBeforeDate: (NSDate*)limit
{
    do
    {
      // 尝试加锁
        int err = pthread_mutex_trylock(&_mutex);
        if (0 == err)
        {
          // 加锁成功直接返回
            return YES;
        }
        if (EDEADLK == err)
        {
            _NSLockError(self, _cmd, NO);
        }
      // 加锁失败，让出当前的 CPU，让别的高优先级线程先执行
        sched_yield();
      // 轮询，直到超时或者加锁成功
    } while ([limit timeIntervalSinceNow] > 0);
    return NO;
}

```

#### 递归锁

递归锁也叫重入锁，解决了递归时互斥锁不再适用的不足。

#### 读写锁

读写锁可以提高性能，比如对于读，我们不希望有锁，只希望在写入时加锁。

#### 自旋锁

自旋锁相当于不断的去查询是否锁已经解开，但是这种轮询的开销特别小。

#### 条件变量

条件变量时信号量的另一种类型，也就是对资源数量的限制。它允许在某个条件满足时向其他线程发送信号。

`NSConditionLock` 和 `NSLock` 类似，都遵循 `NSLocking` 协议，方法都类似，只是多了一个 `Condition` 属性，以及每个操作都多了一个关于 `Condition` 属性的方法。例如 `tryLock`， `tryLockWhenCondition:`， `NSConditionLock` 可以称为条件锁，只有 `Condition` 参数与初始化时候的 `Condition` 相等， `lock` 才能正确进行加锁操作。而 `unlockWithCondition:` 并不是当 `Condition` 符合条件时才解锁，而是解锁之后，修改 `Condition` 的值。

NSCondition 提供了 signal 和 wait 机制，来支持对于资源的等待，另外就是可以使用 boradcase 方法来提醒所有等待锁的消费者，锁已经释放。

## 小结

锁一般在开发中会是大量使用的，一般使用 GCD 信号量就可以了。