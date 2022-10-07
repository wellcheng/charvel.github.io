title: GCD 的一点疑惑与自解
date: 2017-01-19 22:48:22
---

# GCD 的一点疑惑与自解
GCD 其实已经了解了很久了，在实际工程中也会经常用，但是其实很多实践都是浅尝辄止。

最近又在回头看这块儿的内容，有一些知识点，原来不甚明白的，现在有了新的认识，刚好也记录下。

GCD 是一套方案，GCD 本身管理了一组线程池，GCD 负责线程池中的线程创建销毁、并且这种能力是动态化的，即可以充分利用当前 CPU 的多核特性。
添加到 GCD 队列中的任务，会由 GCD 来决定运行在哪一个线程上。也就是说，GCD Queue 是一个抽象概念， task 是添加到 Queue 中不假，但是实际仍然是运行在线程上的。

<!--more-->

## GCD Concurrent Queue 并行队列 

> GCD 的队列中的 task 遵循 FIFO （First In First Out）原则，但是 task 完成的顺序是不可测的。

上面的这段描述，是在当时阅读官方文档的时候，产生了疑惑，我的想法是，既然 task 是按照先后添加到 Queue 中的顺序进行执行的，那为啥完成时间就不一定了呢？

现在明白了，task 的执行压根和 Queue 关系不大，GCD 只是按照添加顺序从 Queue 中一个个拿出 task，扔到线程上去执行。有的 task 一会就完成了，有的需要比较长的时间，当然这个完成顺序就不一定了。

现在想想，当时应该是想当然的认为 GCD 只有在完成上一个任务后才会开启下一个任务，跟串行队列混淆了，从而导致疑惑的。

## GCD 串行队列
串行队列就是只有完成上一个 task ，才会进行下一个 task 的执行。
所以完成顺序与添加顺序是相同的。

## Objc GCD 队列疑惑


![gcd-queues.png](http://upload-images.jianshu.io/upload_images/690040-1820f77761f9ed12.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)


上图是摘录自 Objc.cn 中关于 GCD 队列的一张图。
但是在阅读时，我有下面几点疑惑：

*  为什么 Custom Queues 这一层的关系如此复杂。
* 为什么 Serial Queue 既指向了 Serial Queue ，又能指向 Concurrent Queue。
* 最上面一层中的 Serial Queue 和 倒数第二层中的 Queues 有什么关系。

> GCD 公开有 5 个不同的队列：运行在主线程中的 main queue，3 个不同优先级的后台队列，以及一个优先级更低的后台队列（用于 I/O）。 另外，开发者可以创建自定义队列：串行或者并行队列。自定义队列非常强大，在自定义队列中被调度的所有 block 最终都将被放入到系统的全局队列中和线程池中。

Objc 原文如上描述。

细细品味，其意思是，GCD Queues 有 5 大钦定的 Queue。
开发者当然也可以自定义 Queue（使用 dispatch_queue_create 函数）, 前面说过，Queue 是一个抽象概念，只是简单的保存 task 而已，最终 task 还是要在 GCD Thread Pool 中完成的。

## 目标队列 Target Queue
之前讲过，task 最终是要从 Queue 中放到线程池中执行的。
其实还有一种情况，就是 task 可以从一个 Queue 中放到另一个 Queue 中。
需要设置当前 Queue 的 TargetQueue。
了解了这个概念，就能很清楚的明白上面的图了。

为了更加清楚的阐述这张图的表达意图，可以从这几条线说起：

### S1 -> Main Queue
MainQueue 这个队列中的 task 最终都会放到主线程上执行。如果所示，MainQueue 是指向    Main Thread。
也就是说，如果自定义队列时，如果这个自定义串行队列的 Target Queue 是 Main Queue，那么这些 task 将都会放在主线程执行。

### S3 -> Default Priority Queue
这里的意思很简单，自定义队列的默认 Target Queue 就是 Default Priority Queue，这也就是 Objc 所解释的 `在自定义队列中被调度的所有 block 最终都将被放入到系统的全局队列中和线程池中`	。
在 S3 队列中的任务，会一个接着一个地放入到 Global Queue 中，然后被执行。在前一个 task 完成之后，才会继续下一个 task。

### C2 ->  Default Priority Queue
同上，不过这里，会一咕噜的将 C2 中的 task 往 Global Queue 中放，不用管上一个 task 是否完成。

### S2 -> S3
Serial Queue 的 target queue 同样是 Serial Queue

### C1 + S4 -> C2
C1 的 Concurrent Queue 的 target queue = Concurrent Queue
这个就比较高级了。
在这个模式下，S4 中的 task 串行，C1 中的 task 并行。
但是最终它们这些任务都放在一个并发队列中。


## 最后
以上是个人对于 GCD 的一些见解，如有描述不对，还请及时指正。



