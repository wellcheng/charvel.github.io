---
layout: post
title: "iOS 并发编程指南（二）- Operation Queue"
date: 2015-04-27 10:58:41 +0800

tags: [并发编程, GCD, 翻译]
---

Cocoa operations 通过面向对象的方式封装了执行异步操作的工作。operations 是设计用来连接 operation queue。因为它们是基于 Objective-C 的，operations 常用于 iOS 和 OS X 中。

本章节描述了如何定义和使用 operations

<!--more-->

## Operation Object 简介

operation 对象是 NSOperation 类的一个实例（Foundation Framework 中），用来封装程序要执行的一些操作。NSOperation 类是抽象基类，需要实现自己的子类来执行任务。尽管它是一个抽象的基类，但是仍然已经实现了一些操作，以便于你方便的继承它。除此之外，Foundation Framework 已经提供了两个定义好的子类，你可以直接在代码中使用。如下：

**NSInvocationOperation**:
基于一个对象和方法选择器创建的对象。当你有一个现有的方法，可以使用这个类，它不需要子类化，你还可以用这个类创建更加动态化的方式。

**NSBlockOperation**:
可以使用这个类并发的执行一个或多个 block 对象。因为它可以执行多个 block，它们使用一组语义，当所有相关的 block 执行完毕后，operation 对象才完成。

**NSOperation**:
自定义 operation 对象所需的基类，子类化 NSOperation 并实现你自己的操作，能够得到完整的控制权，包括改变 operation 执行时默认的方式以及状态报告。

所有的操作对象都支持一下关键功能点：

- 支持 operation 对象间基于图论的依赖关系的建立，这些依赖项阻止对象的运行，直到对象所依赖的其他都想都完成。
- 支持一个可选的 completion block，当 operation 的主要任务完成时调用。
- 支持监听 operation 状态的改变，需要 operation 使用 KVO
- 支持优先级，会影响 operation 的执行顺序
- 支持取消 operation 的执行，正在执行性也可以取消。

operation 旨在帮助提高程序的并发性，同时也是一种很好的封装应用程序行为的方式。你应该提交 operation 对象到 operation queue 上，让它们在单独的线程或者多个线程上执行相应的工作，而不是在主线程上做大量的操作。

## 并行与非并行工作组件

尽管你通常是创建 operation 对象然后将其加入到队列中，其实这并不是必须的。也可以调用 operation 对象的 start 方法，但是这样做不能保证该操作执行时是并发的。NSOperation 的 isConcurrent 方法返回对象运行相关的线程是同步还是异步的。默认情况下，返回 NO 表示 operation 在调用的线程中同步的执行。

如果想要实现并发方法，那就是，调起方法的线程是异步的，你必须多写一些代码来异步的开启 operation。例如，可能会生成一个单独的线程，调用系统的异步函数，或者做一些其他操作保证其开始执行任务后能够在任务完成之前立刻返回。

大多数的程序员并不需要去具体实现如何并发执行 operation 对象，因为你总是将其添加到 operation queue 中，如果你添加一个非并发的 operation 到队列中，队列会自行创建一个线程用来执行你的任务。因此，将非并发的任务添加到队列中仍然会异步的执行，只有在需要执行异步操作而不加入队列中时，才会考虑实现并发 operations 的具体操作。

## 创建 NSInvocationOperation 对象

NSInvocationOperation 类是 NSOperation 类的子类，当其运行时，调用指定对象的指定方法。使用这个方法来避免大量定义一些自定义的 operation 对象为你程序中的每一个任务。尤其是你要修改现有的程序，可以用它执行已经存在的一些方法。还可以基于用户输入动态的选择方法选择子。

创建 invocation 对象的流程很简单，创建并且实例化这个类的对象即可，传入所需的对象和选择子。如下：

    @implementation MyCustomClass
    - (NSOperation*)taskWithData:(id)data {
        NSInvocationOperation* theOp = [[NSInvocationOperation alloc] initWithTarget:self selector:@selector(myTaskMethod:) object:data];

        return theOp;
    }

    - (void)myTaskMethod:(id)data {
        // 执行任务
    }

    @end

    
## 创建 NSBlockOperation 对象

NSBlockOperation 是 NSOperation 的子类，用于封装一个或多个 block。应用程序中已经存在 Operation Queue，但是不想使用 dispatch 时，它则提供了一种对于 block 面向对象的封装。你可以使用 operation Queue 的一些有用的特性，例如操作依赖、KVO 通知和其他功能，这些是 dispatch 不具有的。

当你初始化 block operation 时，至少需要传入一个 block，之后也可以添加更多的 block。当 NSBlockOperation 对象执行时，它会提交所有默认优先级的 block，并发调度队列。对象然后等待所有的 block 执行完毕，最后一个 block 执行完毕时，block operation 对象本身标记为已完成。因此你可以使用 block operation 跟踪一组 block，很像你使用一个线程合并其他线程返回的结果。不同的是，由于 block operation 运行在单独的线程中，在等待 block operation 完成的同时，应用程序的其他线程仍可以做其他工作。如下：

    NSBlockOperation* theOp = [NSBlockOperation blockOperationWithBlock: ^{
          NSLog(@"Beginning operation.\n");
          // Do some work.
    }];

创建一个 block operation 对象后，你可以使用 `addExecutionBlock:` 来添加更多的 block，如果需要串行的执行，那么需要提交到串行执行的队列上。

## 自定义 Operation 对象

如果 block operation 和 invocation operation 仍然不能完全满足你应用程序的需要，那么你可以直接子类化 NSOperation 来定义自己的 operation 对象，为其添加所需的任何行为。NSOperation 类为所有的 operation 对象提供了一些基本的子类化点。同时提供了大量的基础方法用于实现依赖于 KVO，然而，有时候你仍然需要补充现有的基础结构，来确保业务的正常运行。其他一些额外的工作量取决于你需要实现并发或者非并发的操作。

定义非并发的操作要比并发简单的多，对于非并发的操作，你只需要执行主要任务并回应取消事件。对于并发操作，必须重载现有的一些代码，下面各节教你怎么做：

### 执行主任务

每个操作对象至少需要实现这两个方法：

- 初始化方法
- 主要任务

你需要自定义初始化方法使得你的 operation 对象得到已知的状态和一个需要执行的主任务。如果需要，可以实现下列可选的方法：

- 在实现 `main` 方法时，你打算调用的自定义方法
- 访问器方法，用于设置数据值和访问操作的结果
- 实现 NSCoding 协议用于持久化 operation 对象

下面列出了自定义的模板，
    
    @interface MyNonConcurrentOperation : NSOperation
    
    @property id (strong) myData;
    -(id)initWithData:(id)data;

    @end
    
    @implementation MyNonConcurrentOperation
    - (id)initWithData:(id)data {
       if (self = [super init])
          myData = data;
       return self;
    }
    -(void)main {
       @try {
          // Do some work on myData and report the results.
       }
       @catch(...) {
          // Do not rethrow exceptions.
    } }
    @end


具体的操作可以参考[示例代码](https://developer.apple.com/library/mac/samplecode/NSOperationSample/Introduction/Intro.html)

### 响应取消事件

当 operation 对象开始执行后，只有你在明确的告诉它取消时，才会结束执行。取消可以发生在任何时间，甚至是在 operation 还未开始执行时。尽管 NSOperation 类向用户提供了一种取消的方式，但是识别取消事件是资源有必要的。如果对象被彻底终止，可能无法收回已经分配的资源，其结果是，操作对象应该在操作期间检查取消事件并且优雅的处理。

为了支持取消，你需要做的就是在自己的代码中周期性的调用对象的 isCancelled 方法，并且当不返回 YES 时立刻返回。支持取消操作是很重要的。isCancelled 方法非常快，可以频繁的调用，不会造成额外的开销。你可能需要在以下地方调用 isCancelled 方法：

- 再执行任何自然操作时立刻调用
- 在循环中至少调用一次，或者调用多次当循环耗时较长时
- 任何相当容易终止操作的地方

例如：
    
    - (void)main {
       @try {
          BOOL isDone = NO;
          while (![self isCancelled] && !isDone) {
              // Do some work and set isDone to YES when finished
        } 
    }
       @catch(...) {
          // Do not rethrow exceptions.
    
          } 
    }



尽管上面的例子没有做任何清楚操作，但是你实际中应该去释放已经占用的资源。

### 配置 Operation 并发执行

默认情况下操作队列以同步方式执行，也就是说它们从开始调用的线程中执行任务。因为操作队列为非并发提供了线程，不过，大多数操作需要异步运行。但是，如果需要异步执行，那么需要定义下。

下面列出了并发执行时需要重载的方法：

- **start** ：（必须）所有的并发操作必须重载这个方法，你调用 start 方法手动执行操作。因此，这个方法的实现是一个起点，并且是你设置线程或者其他运行环境的地方。任何时候都不能调用 super。
- **main**：（可选）这个方法通常用于实现操作对象相应的任务。尽管你可能执行这个任务在 start 方法中，但是将其转移到这里会有较好的代码条理。
- **isExcuting**、**isFinished**：（必须）并发操作应该建立运行环境并且报告状态给外面的用户，然而，一个并发操作必须维护状态信息，来了解何时开启任务，何时完成任务。你实现的这些方法必须是线程安全的，可能与 KVO 有关
- **isConcurrent**：重载并且返回 YES 即可。

下面是一个代码示例：

    @interface MyOperation : NSOperation {
          BOOL        executing;
          BOOL        finished;
    ￼￼￼￼}
    - (void)completeOperation;
    @end

    
    @implementation MyOperation
    - (id)init {
        self = [super init];
        if (self) {
            executing = NO;
            finished = NO;
        }
        return self;
    }
    - (BOOL)isConcurrent {
        return YES;
    }
    - (BOOL)isExecuting {
        return executing;
    }
    - (BOOL)isFinished {
        return finished;
    }
    @end


下面是 start 方法：

    - (void)start {
       // Always check for cancellation before launching the task.
       if ([self isCancelled]) {
            // Must move the operation to the finished state if it is canceled.
            [self willChangeValueForKey:@"isFinished"];
            finished = YES;
            [self didChangeValueForKey:@"isFinished"];
            return; 
        }
        // If the operation is not canceled, begin executing the task.
        [self willChangeValueForKey:@"isExecuting"];
        [NSThread detachNewThreadSelector:@selector(main) toTarget:self withObject:nil];
        executing = YES;
        [self didChangeValueForKey:@"isExecuting"];
    }



下面是 main  方法：

    - (void)main {
       @try {
           // Do the main work of the operation here.
           [self completeOperation];
       }
       @catch(...) {
          // Do not rethrow exceptions.
    } }
    - (void)completeOperation {
        [self willChangeValueForKey:@"isFinished"];
        [self willChangeValueForKey:@"isExecuting"];
        executing = NO;
        finished = YES;
        [self didChangeValueForKey:@"isExecuting"];
        [self didChangeValueForKey:@"isFinished"];
    }

即使 operation 被取消，你总是应该通知 KVO 观察者你现在完成了任务。依赖于其他对象时，它们检测这些 isFinished ，当所有依赖项全部完成时，它本身才可以执行，如果不生成通知，可能会影响其他操作的执行。

### 保证 KVO 规则

NSOperation 对于以下实现 KVO：

- isCancelled
- isConcurrent
- isExecuting
- isFinished
- isReady
- dependencies
- queuePriority
- completionBlock

如果你对 NSOperation 重写 start 方法或者做了其他一些重大的定制，除了重载 mian 方法，你需要确保你的自定义对象仍然保证了 KVO 规则。

如果你想要实现对其他操作队列依赖项的支持，你可以重载 isReady 方法，直到你自定义的依赖项已经满足，不然一直返回 NO。（应该调用 super ），当状态改变时，告知 KVO。除非你重写 addDependency 或 removeDependency 方法，不然无需担心 KVO

尽管你可以生成其他 key  的 KVO，但是你可能不需要这样做。如果你取消 operation ，只需调用 cancel 方法即可，不需要修改优先级。最后，除非你的 operation 会动态改变 isConcurrent 属性，不然无需重载其。

## 自定义 operation 对象的执行行为

创建 operation 对象后，再将其添加到队列之前，你可以对 operation 对象进行配置。这一节中描述的配置类型可以应用于所有的 operation 对象，无论是自定义的继承于 NSOperation 的子类还是系统提供的子类。

### 配置操作之间的依赖关系

依赖关系是一种序列化不同操作对象的方式。一个对象依赖于其他对象的执行，只有当其依赖的对象全部完成时，它自己才可以开始执行。因此，你可以使用依赖关系来创立复杂的执行顺序。

使用 NSOperation 的 `addDependency：` 方法创建两个对象之间的依赖关系。这个方法创建一个从当前对象到目标对象单向的依赖关系，可以作为一个参数。这种依赖性表示当前的对象只有在目标对象完成时才可以开始执行。这种依赖关系并不局限于同一个队列中的操作对象，因为操作对象管理自己的依赖项，所以，完全可以在不同队列中的不同操作对象之间创建依赖关系。需要注意的是，不要产生循环依赖的关系。

当所有的依赖项都已经完成时，当前操作对象将准备开始执行（当你重载 isReady 方法后，这个状态需要使用你自己创立的标准）。如果对象已经处于队列中，那么队列可能随时开始这个对象的执行，如果你打算手动管理对象的执行，需要你调用 operation 的 start 方法。

>你应该总是在 operation 运行之前或者添加到队列之前配置它的依赖关系，再运行之后添加依赖关系可能无效。

依赖关系依赖于每个对象当状态变化时发送 KVO 通知，当你自定义对象的这种行为时，为了避免出错，可能需要在你的代码中生成对应的 KVO 通知。

### 更改操之执行的优先级

对于添加到队列的 operation，执行顺序首先取决于已经入队的 operation 的准备情况，然后取决于其优先级。准备情况取决于 operation 依赖的其他对象，但是优先级是 operation 自身的属性，默认情况下，所有新建的 operation 都有一个默认的优先级，但是你可以通过 `setQueuePriority：`方法增大或者减少优先级。

优先级只适用于同一个队列中的不同对象，如果你的程序有多个操作队列，每个对象在不同的队列中的优先级是相互独立的。因此，低优先级的 operation 可能在不同的队列中执行的比高优先级的 operation 要早。

优先级不是依赖关系的替代，优先级只决定某个队列中已经做好执行准备的操作对象，举个例子，如果一个队列中，存在已经准备好的两个不同优先级的对象，那么首先执行高优先级的，但是，如果高优先级的还未准备好，但是低优先级的已经准备好了，那么首先会执行低优先级的，
如果你是想要使某个 operation 必须在另一个 operation 执行完毕之后才执行，那么应该使用依赖关系。

### 更改底层的线程优先级

在 OS X 10.6 版本之后，可以配置 operation 底层的线程优先级，系统线程的规则由操作系统内核管理，但是一般高优先级的线程一般要比低优先级的执行的快。operation 对象中，线程优先级的范围为 0 ~ 1.0 之间的浮点数，默认为 0.5。

使用 `setThreadPriority:` 方法设置 operation 线程的优先级，当执行 operation 时，默认的 start 方法使用你指定的优先机去修改当前线程的优先级。这个优先级仅仅在 operation 的 main 方法中有效，其他地方的代码仍然运行在默认优先级的线程上，如果你自己创建一个并发的 operation Queue ，那么需要自行去配置线程的优先级。

### 设置 completion block

在 OS X 10.6 之后，operation 在主任务完成后能够执行一个完成块，你可以使用这个 block 来做一些事情，并发的操作可能会使用这个块来生成其完成的 KVO 通知。

使用 `setCompletionBlock：` 即可。

## 实现 operation 对象的技巧

尽管 operation 对象很容易实现，但是还是有一些地方需要你去考虑，下面是一些你在编码时需要考虑的几点：

### Operation 对象的内存管理

下面几点说明了 operation 对象内存管理的几个关键点

#### 避免每个线程一份存储

operation Queue 通常会将 operation 运行在一条线程上作为非并发编程的方式，当队列提供这条线程时，应该考虑到线程是 Queue 拥有的，你不应该去碰它。特别的，你不应该将不是你自己创建和管理的线程与任何数据关联起来。operation queue 管理的线程池依赖于系统和你程序的需要，因此，通过线程数据存储传递数据在不同的 operation 之间是不可靠的，很有可能会失败。

在 operation 对象中，你没有任何理由需要使用线程数据存储。如果你初始化了一个 operation 对象，你应该提供所有它执行需要的数据，因此，operation 对象提供所有的上下文环境，所有传入和传出的数据都应该存储在那里，直到返回到程序中或者不在需要了。

#### 如果有需要，保持对 operation 对象的引用

只是因为操作对象是以异步的方式执行的，你不应该假定你能够创建它们。它们仍是对象，当代码中需要时需要由你来管理它们的引用，例如从一个已经完成的对象中检索数据。

还有一个你需要保持对象引用很重要的原因是，你没有办法通过 queue 获取加入到里面的 operation 对象。在许多情况下，当 operation 对象加入到 queue 后立刻开始执行，当你自己的代码去 queue 中获取 operation 对象时，可能已经完成并从队列中删除了。

### 处理错误和异常

由于 operation 是程序中的离散项，所以需要处理错误和异常，在 OS X 10.6 以后，默认的 start 方法中不再支持异常捕获，你的代码应该直接去处理这些异常，检查错误并且有必要时通知你代码的其他部分，如果你替换掉了 start 方法，你同样必须捕获这些异常。

## 确定 operation 对象的合理范围

虽然可以将任意数量的 operation 对象添加到队列中，但是这样做是不切实际的。像其他对象一样，operation 对象在初始化和运行时也有内存开销，如果每个 operation 只做少量的工作，但是你创建了很多，那么效率一定会很低，在内存紧张时可能进一步降低效率。

高效使用 operation 的关键点是找到工作量和计算机能力的平衡，并使计算机一直处于忙碌中，设法你操作的数量是合理的。

还应该避免一下子向操作队列中添加大量的 operation ，或者避免太快地向操作队列添加 operation，可以考虑使用批处理，当一批任务完成后，通知程序添加新的一批 operation。当你需要做大量工作时，可能考虑让队列充满很多 operation，以便于计算机忙碌起来，但是实际中可能造成内存溢出。

当然，你应该使用性能分析工具来决定这个数量。

## 执行 operations

最终，程序需要执行这些 operations 来完成对应的工作，下面各节提供了几种执行的方式供学习。

### 向操作队列中添加 operation

到目前为止，执行 operation 最简单的方式就是将其添加到一个 operation queue，你的应用程序负责创建和维护需要使用的操作队列。程序可以拥有多个操作队列，但是太多会有限制。操作队列与系统根据内核和系统负载，决定某一时刻维持多少并发的数量，因此，创建太多的操作队列不一定能够立即执行。创建队列与创建其他对象一样：

    NSOperationQueue *aQueue = [[NSOperationQueue alloc] init];

可以使用  `addOperation:` 方法添加 operation，同时还可以添加 block, 使用 `addOperationWithBlock:` ，所有这些方法进队一个 operation 并提示开始执行。大多数情况下，operations 会在添加到队列后就开始执行，但是仍然还有其他一些原因可能会推迟执行。例如某个 operation 依赖于其他 operation 的完成，或者操作队列本身是被悬挂的，或者操作数超过了设置的最大并发数。

> 一定要注意当 operation 添加到操作队列后，不能去改变它，你可以使用 NSOperation 类的其他方法来查看它当前的状态。

尽管操作队列是并发的，但是仍然可以设置其最大并发数为 1 使其强制串行执行，尽管同一时间只有一个 operation 执行，但是其执行顺序仍然受其他因素的影响，因此串行的操作队列并不能提供像 GCD 一样的行为，如果串行执行的顺序对你很重要，那么使用依赖关系吧。


### 手动执行 operation

尽管操作队列很方便的执行 operation，但是有时候仍希望不通过操作队列来执行某些 operation
，下面是一些手动执行需要注意的地方，需要注意的是，你必须要保证其准备好执行并且调用其 start 方法。

operation 不会在 isReady 方法返回 YES 之前执行，isReady 方法取决于其依赖的对象是否全部执行完毕。

手动执行时，你必须调用其 start 方法，不能使用 main 或者其他方法，因为 start 方法在真正运行前做了一些检查，尤其是 start 方法有关 KVO 通知的生成，其他依赖对象需要依赖项的正确。同时还保证在取消之后不去执行或者没有准备好执行时抛出异常。

如果程序定义了一些并发的operation 对象，你应该考虑调用 isConcurrent 方法，当它返回 NO 时，你本地的代码能够决定在当前线程同步执行 operation 还是开启另一个线程。


    - (BOOL)performOperation:(NSOperation*)anOp
    {
         BOOL ranIt = NO;
        if ([anOp isReady] && ![anOp isCancelled])
        {
        if (![anOp isConcurrent])
            [anOp start];
        else
            [NSThread detachNewThreadSelector:@selector(start)
                        toTarget:anOp withObject:nil];
        ranIt = YES; 
    
        }
        else if ([anOp isCancelled])
        {
        // If it was canceled before it was started,
        //  move the operation to the finished state.
        [self willChangeValueForKey:@"isFinished"];
        [self willChangeValueForKey:@"isExecuting"];
        executing = NO;
        finished = YES;
        [self didChangeValueForKey:@"isExecuting"];
        [self didChangeValueForKey:@"isFinished"];
        // Set ranIt to YES to prevent the operation from
        // being passed to this method again in the future.
        ranIt = YES;
        }
        return ranIt;
    }


### 取消 operation 

当添加 operation 到操作队列后，将由操作队列高效的管理并且不能被移除，唯一出队的方式就是取消它，你可以取消单个的 operation 通过调用其 cancel 方法，也可以调用 queue 的 cancelAllOperation 方法取消所有 operations。

你只有在确定不再需要它时才取消它，取消其实相当于已经完成，如果被依赖项被取消，那么这个依赖关系就不再成立了。

### 等待 operation 完成

为了更好地表现，你应该让你的 operation 尽可能的并发，让你的程序能够去做别的事情，如果创建 operation 的代码同时需要处理结果，可以调用 `waitUntilFinished` 方法。但是尽量避免使用这个方法。

> 你永远不能在程序主线程中等待 operation，应该从工作线程或者其他 operation 对象，阻塞主线程会使程序迟钝。

除了等待单个 operation 操作完成，也可以等待整个操作队列完成，然后可以继续向其添加 operation。

###暂停和恢复队列

如果你需要暂停某个 operation 的执行，你可以使用队列的 `setSuspended` 方法，挂起队列不会影响已经执行的 operation，只会暂停新添加的 operation。







