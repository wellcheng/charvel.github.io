---
layout: post
title: "iOS 并发编程指南(五)-将现有代码迁移为可并发"
date: 2015-05-12 23:20:49 +0800

tags: [并发编程, GCD]
---

将现有的代码迁移至可并发，例如 GCD 或 NSOperation ，有很多办法，尽管可能不是所有的代码都需要迁移，但是部分迁移能够提升程序性能：

- 使用dispatch queue和Operaiton queue相比线程拥有许多优点：
- 应用不再需要存储线程栈到内存空间
- 消除了创建和配置线程的代码
- 消除了管理和调度线程工作的代码
- 简化了你要编写的代码

本章节主要提供一些迁移相关的 tips

<!--more-->

## 使用Dispatch Queue替代线程

首先考虑应用可能使用线程的几种方式：

- **单一任务线程**：创建一个线程执行单一任务，任务完成时释放线程
- **工作线程(Worker)**：创建一个或多个工作线程执行特定的任务，定期地分配任务给每个线程
- **线程池**：创建一个通用的线程池，并为每个线程设置run loop，当你需要执行一个任务时，从池中抓取一个线程，并分配任务给它。如果没有空闲线程可用，任务进入等待队列。

虽然这些看上去是完全不同的技术，但实际上只是相同原理的变种。应用都是使用线程来执行某些任务，区别在于管理线程和任务排队的代码。使用dispatch queue和operation queue，你可以消除所有线程、及线程通信的代码，集中精力编写处理任务的代码。

如果你使用了上面的线程模型，你应该已经非常了解应用需要执行的任务类型，只需要封装任务到Operation对象或Block对象，然后dispatch到适当的queue，就一切搞定!

对于那些不使用锁的任务，你可以直接使用以下方法来进行迁移：

- 单一任务线程，封装任务到block或operation对象，并提交到并发queue
- 工作线程，首先你需要确定使用串行queue还是并发queue，如果工作线程需要同步特定任务的执行，就应该使用串行queue。如果工作线程只是执行任意任务，任务之间并无关联，就应该使用并发queue
- 线程池，封装任务到block或operation对象，并提交到并发queue中执行

当然，上面只是简单的情况。如果任务会争夺共享资源，理想的解决方案当然是消除或最小化共享资源的争夺。如果有办法重构代码，消除任务彼此对共享资源的依赖，这是最理想的。

如果做不到消除共享资源依赖，你仍然可以使用queue，因为queue能够提供可预测的代码执行顺序。可预测意味着你不需要锁或其它重量级的同步机制，就可以实现代码的同步执行。

你可以使用queue来取代锁执行以下任务：

- 如果任务必须按特定顺序执行，提交到串行dispatch queue;如果你想使用Operation queue，就使用Operation对象依赖来确保这些对象的执行顺序。
- 如果你已经使用锁来保护共享资源，创建一个串行queue来执行任务并修改该资源。串行queue可以替换现有的锁，直接作为同步机制使用。
- 如果你使用线程join来等待后台任务完成，考虑使用dispatch group;也可以使用一个 NSBlockOperation 对象，或者Operation对象依赖，同样可以达到group-completion的行为。
- 如果你使用“生产者-消费者”模型来管理有限资源池，考虑使用 dispatch queue 来简化“生产者-消费者”
- 如果你使用线程来读取和写入描述符，或者监控文件操作，使用dispatch source

记住queue不是替代线程的万能药!queue提供的异步编程模型适合于延迟无关紧要的场合。虽然queue提供配置任务执行优先级的方法，但更高的优先级也不能确保任务一定能在特定时间得到执行。因此线程仍然是实现最小延迟的适当选择，例如音频和视频playback等场合。

## 消除基于锁的代码

在线程代码中，锁是传统的多个线程之间同步资源的访问机制。但是锁的开销本身比较大，线程还需等待锁的释放。

使用queue替代基于锁的线程代码，消除了锁带来的开销，并且简化了代码编写。你可以将任务放到串行queue，来控制任务对共享资源的访问。queue的开销要远远小于锁，因为将任务放入queue不需要陷入内核来获得mutex

将任务放入queue时，你做的主要决定是同步还是异步，异步提交任务到queue让当前线程继续运行;同步提交任务则阻塞当前线程，直到任务执行完成。两种机制各有各的用途，不过通常异步优先于同步。

### 实现异步锁

异步锁可以保护共享资源，而又不阻塞任何修改资源的代码。当代码的部分工作需要修改一个数据结构时，可以使用异步锁。使用传统的线程，你的实现方式是：获得共享资源的锁，做必要的修改，释放锁，继续任务的其它部分工作。但是使用dispatch queue，调用代码可以异步修改，无需等待这些修改操作完成。

下面是异步锁实现的一个例子，受保护的资源定义了自己的串行dispatch queue。调用代码提交一个block到这个queue，在block中执行对资源的修改。由于queue串行的执行所有block，对这个资源的修改可以确保按顺序进行;而且由于任务是异步执行的，调用线程不会阻塞。

    dispatch_async(obj->serial_queue, ^{

    // Critical section

    });

### 同步执行临界区

如果当前代码必须等到指定任务完成，你可以使用 dispatch_sync 函数同步的提交任务，这个函数将任务添加到dispatch queue，并阻塞当前线程直到任务完成执行。dispatch queue本身可以是串行或并发queue，你可以根据具体的需要来选择使用。由于 dispatch_sync 函数会阻塞当前线程，你只应该在确实需要的时候才使用。

下面是使用 dispatch_sync 实现临界区的例子：

    dispatch_sync(my_queue, ^{

    // Critical section

    });

如果你已经使用串行queue保护一个共享资源，同步提交到串行queue，并不能比异步提交提供更多的保护。同步提交的唯一理由是，阻止当前代码在临界区完成之前继续执行。如果当前代码不需要等待临界区完成，或者可以简单的提交接下来的任务到相同的串行queue，就应该使用异步提交。

## 改进循环代码

如果循环每次迭代执行的工作互相独立，可以考虑使用 dispatch_apply 或 dispatch_apply_f 函数来重新实现循环。这两个函数将循环的每次迭代提交到dispatch queue进行处理。结合并发queue使用时，可以并发地执行迭代以提高性能。

dispatch_apply 和 dispatch_apply_f 是同步函数，会阻塞当前线程直到所有循环迭代执行完成。当提交到并发queue时，循环迭代的执行顺序是不确定的。因此你用来执行循环迭代的Block对象(或函数)必须可重入(reentrant)。

下面例子使用dispatch来替换循环，你传递给 dispatch_apply 或 dispatch_apply_f 的Block或函数必须有一个整数参数，用来标识当前的循环迭代：

    queue = dispatch_get_global_queue(DISPATCH_QUEUE_PRIORITY_DEFAULT, 0);

    dispatch_apply(count, queue, ^(size_t i) {

    printf("%u\n", i);

    });


你需要明智地使用这项技术，因为dispatch queue的开销虽然非常小，但仍然存在，你的循环代码必须拥有足够的工作量，才能忽略掉dispatch queue的这些开销。

提升每次循环迭代工作量最简单的办法是striding(跨步)，重写block代码执行多个循环迭代。从而减少了 dispatch_apply 函数指定的count值。

    int stride = 137;

    dispatch_queue_t queue = dispatch_get_global_queue(DISPATCH_QUEUE_PRIORITY_DEFAULT, 0);

    dispatch_apply(count / stride, queue, ^(size_t idx){

    size_t j = idx * stride;

    size_t j_stop = j + stride;

    do {

    printf("%u\n", (unsigned int)j++);

    }while (j < j_stop);

    });

    // 执行剩余的循环迭代

    size_t i;

    for (i = count - (count % stride); i < count; i++)

    printf("%u\n", (unsigned int)i);
    
如果循环迭代次数非常多，使用stride可以提升性能。

## 替换线程Join

线程join允许你生成多个线程，然后让当前线程等待所有线程完成。线程创建子线程时指定为joinable，如果父线程在子线程完成之前不能继续处理，就可以join子线程。join会阻塞父线程直到子线程完成任务并退出，这时候父线程可以获得子线程的结果状态，并继续自己的工作。父线程可以一次性join多个子线程。

Dispatch Group提供了类似于线程join的语义，但拥有更多优点。dispatch group可以让线程阻塞直到一个或多个任务完成。和线程join不一样的是，dispatch goup同时等待所有子任务完成。而且由于dispatch group使用dispatch queue来执行任务，更加高效。

以下步骤可以使用dispatch group替换线程join：

1. 使用 dispatch_group_create 函数创建一个新的dispatch group
2. 使用 dispatch_group_async 或 dispatch_group_async_f 函数添加任务到Group，这些是你要等待完成的任务
3. 如果当前线程不能继续处理任何工作，调用 dispatch_group_wait 函数等待这个group，会阻塞当前线程直到group中的所有任务执行完成。

如果你使用Operation对象来实现任务，可以使用依赖来实现线程join。不过这时候不是让父线程等待所有任务完成，而是将父代码移到一个Operation对象，然后设置父Operation对象依赖于所有子Operation对象。这样父Operation对象就会等到所有子Operation执行完成后才开始执行。

## 修改“生产者-消费者”实现

生产者-消费者 模型可以管理有限数量动态生产的资源。生产者生成新资源，消费者等待并消耗这些资源。实现生产者-消费者模型的典型机制是条件或信号量。

使用条件(Condition)时，生产者线程通常如下：

1. 锁住与condition关联的mutex(使用pthread_mutex_lock)

2. 生产资源(或工作)

3. Signal条件变量，通知有资源(或工作)可以消费(使用pthread_cond_signal)

4. 解锁mutex(使用pthread_mutex_unlock)

对应的消费者线程则如下：

1. 锁住condition关联的mutex(使用pthread_mutex_lock)
2. 设置一个while循环[list=1]
    - 检查是否有资源(或工作)
    - 如果没有资源(或工作)，调用pthread_cond_wait阻塞当前线程，直到相应的condition触发

3. 获得生产者提供的资源(或工作)
4. 解锁mutex(使用pthread_mutex_unlock)
5. 处理资源(或工作)

使用 dispatch queue，你可以简化生产者-消费者为一个调用：

    dispatch_async(queue, ^{

    // Process a work item.

    });
    
当生产者有工作需要做时，只需要将工作添加到queue，并让queue去处理该工作。唯一需要确定的是queue的类型，如果生产者生成的任务需要按特定顺序执行，就使用串行queue;否则使用并发Queue，让系统尽可能多地同时执行任务。

## 替换Semaphore代码

使用信号量可以限制对共享资源的访问，你应该考虑使用dispatch semaphore来替换普通信号量。传统的信号量需要陷入内核，而dispatch semaphore可以在用户空间快速地测试状态，只有测试失败调用线程需要阻塞时才会陷入内核。这样dispatch semaphore拥有比传统semaphore快得多的性能。两者的行为是一致的。

## 替换Run-Loop代码

如果你使用run loop来管理一个或多个线程执行的工作，你会发现使用queue来实现和维护任务会简单许多。设置自定义run loop需要同时设置底层线程和run loop本身。run-loop代码则需要设置一个或多个run loop source，并编写回调来处理这些source事件到达。你可以创建一个串行queue，并dispatch任务到queue中，这样一行代码就能够替换原有的run-loop创建代码：


    dispatch_queue_t myNewRunLoop = dispatch_queue_create("com.apple.MyQueue", NULL);


由于queue自动执行添加进来的任务，不需要编写额外的代码来管理queue。你也不需要创建和配置线程，更不需要创建或附加任何run-loop source。此外，你可以通过简单地添加任务就能让queue执行其它类型的任务，而run loop要实现这一点，必须修改现有run loop source，或者创建一个新的run loop source。

run loop的一个常用配置是处理网络socket异步到达的数据，现在你可以附加dispatch source到需要的queue中，来实现这个行为。dispatch source还能提供更多处理数据的选项，支持更多类型的系统事件处理。

## 与POSIX线程的兼容性

Grand Central Dispatch管理了任务和运行线程之间的关系，通常你应该避免在任务代码中使用POSIX线程函数，如果一定要使用，请小心。

应用不能删除或mutate不是自己创建的数据结构。使用dispatch queue执行的block对象不能调用以下函数：

- `pthread_detach`
- `pthread_cancel`
- `pthread_join`
- `pthread_kill`
- `pthread_exit`

任务运行时修改线程状态是可以的，但你必须还原线程原来的状态。只要你记得还原线程的状态，下面函数是安全的：

- `pthread_setcancelstate`
- `pthread_setcanceltype`
- `pthread_setschedparam`
- `pthread_sigmask`
- `pthread_setspecific`


特定block的执行线程可以在多次调用间会发生变化，因此应用不应该依赖于以下函数返回的信息：

- `pthread_self`
- `pthread_getschedparam`
- `pthread_get_stacksize_np`
- `pthread_get_stackaddr_np`
- `pthread_mach_thread_np`
- `pthread_from_mach_thread_np`
- `pthread_getspecific`

Block必须捕获和禁止任何语言级的异常，Block执行期间的其它错误也应该由block处理，或者通知应用

