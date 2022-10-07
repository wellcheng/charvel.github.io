---
layout: post
title: "iOS 并发编程指南(三)-Dispatch Queues"
date: 2015-04-30 20:48:38 +0800

tags: [并发编程, GCD, 翻译]
---

GCD 调度队列是执行任务的强大工具，可以使用同步或异步方式执行任意的代码块。调度队列可以执行几乎所有的单线程代码。相比于线程代码，其更易于使用，且效率要高。

本章将介绍调度队列，以及如何使用它执行大多数任务。

## Dispatch Queue 介绍
调度队列是在程序中执行同步或异步代码的比较简单的一种方式，任务就是程序中需要执行的一些简单的工作。举个🌰，比如执行一些计算，创建或者修改一个数据结构，从文件中读取数据进行处理，等等这样的一些工作。你通过在函数或 block 中写入相应的代码来定义任务，然后将其添加至调度队列。

调度队列是一种类似对象的结构，管理你提交给它的任务。所有的调度队列都是先进先出（FIFO）的，因此任务开始执行的顺序与添加顺序一致。GCD 已经提供了一些常用的调度队列，你也可以根据需求创建新的调度队列。下面的表列出了几种调度队列，并做了简要的说明


<!--more-->



<table>
<tr>
    <th>类型</th>
    <th>说明</th>
</tr>
<tr>
    <td>串行队列（Serial）</td>
    <td>
    串行队列，也称私有调度队列，按照任务添加的顺序依次执行。当前执行的任务运行在不同（具体跟任务之间的差异有关）的线程上，并且由调度队列直接管理。串行队列经常用来执行一些同步操作，比如对于指定资源的存取
    </td>
</tr>
<tr>
    <td>并行队列（Concurrent）</td>
    <td>
    并行队列，也称全局队列，可以同时执行一个或多个任务。但是任务仍然按照添加的顺序执行，当前的任务运行在不同的线程上，并由调度队列管理这些线程。某个时间执行的任务数是变化的，取决于操作系统当前的状态。
    
    在 iOS 5 以后，你可以指定调度队列类型为 `DISPATCH_QUEUE_CONCURRENT` 来创建自己的并发调度队列，除此之外，系统已经提供了四个预先定义的调度队列供使用。
    </td>
</tr>
<tr>
    <td>主调度队列（Main dispatch queue）</td>
    <td>
    主调度队列是全局可用的串行队列，在程序的主线程上执行任务。与应用程序的其他事件执行交织在一起加入到 RunLoop 中。由于其运行在主线程中，所以经常用作关键的同步点。
    
    尽管你不需要创建自己的主队列，但是需要确保程序能够快速的执行完所添加的任务。因为会阻塞 UI。
    </td>
</tr>
</table>

当程序中加入并行开发时，相比较线程，调度队列提供了几个特有的优势。最直接的优势就是简单的工作队列以及编程模型。使用线程时，你不仅需要编写任务所需的代码，还需创建和管理自己的线程。调度队列能够让你专注于任务的处理，而无需担心线程的管理和创建，相反，操作系统会为你做所有线程创建和管理的工作。相比于程序，系统能够更加高效的管理线程。系统能够根据可用的资源以及当前的并发数，动态的扩展线程的数量。此外，系统线程通常会比你自己创建的线程更快的开始执行。

或许你可能觉得因为调度队列重写自己的代码会很麻烦，其实为调度队列编码要比为线程简单的多。编码的关键是设计能够异步执行且资源就绪的任务。然而，调度队列还有个优点是可预测性，如果你有两个不同的线程需要存取同一份资源，那么每个线程都有可能先修改资源，所以你需要锁来保证两个线程不会同时做修改。使用调度队列时，你可以将这两个任务添加进一条串行队列来确保同时只有一个任务可以修改资源。这种基于同步的队列要比锁效率更高一些，因为锁不管是不是在竞争条件下都需要一个昂贵的内核致陷。然而调度队列运行在你自己程序的空间中，只有在必要时才终端内核。

虽然你可能会指出，运行在串行队列上的任务不会同时执行，但是你必须要记住，如果两个线程同一时刻持有了一把锁，那么这两个线程将丢失并发性或者显著下降。更重要的是，线程模型必须要创建两条线程，它们都占据内核和用户内存。调度队列不需要不需要这些额外开销，并且能够保证线程一直运行且不会锁定。

下面是调度队列其他一些需要重点关注的地方：

- 所有调度队列同时执行它们队列中的任务，只有单个队列中才会执行序列化的任务。
- 系统决定同一时间类可以执行任务的总数，因此，程序中100个队列中的100个任务，可能不会全部同时执行（除非有100或更多的核心）。系统在选择任务时，首先考虑队列的优先级，添加进队列的任务必须满足可以执行的条件，私有队列是引用计数的对象，除了你自己的代码会引用它外，需要注意添加到其中的 dispatch source 也能够保留它。因此，你需要确保所有 dispatch source 都被取消了，并且保留和释放操作是平衡的。

## 队列相关的一些技术点
除了调度队列本身之外， GCD 还提供了有关使用队列编码的一些技术点，请看下表：

<table>
<tr>
    <th>技术点</th>
    <th>描述</th>
</tr>
<tr>
    <td>Dispatch groups</td>
    <td>
    Dispatch Group 用来观察一组 block 对象的完成。它提供了一种非常有用的同步机制，依赖其他任务的完成。
    </td>
</tr>
<tr>
    <td>Dispatch semaphores</td>
    <td>
    Dispatch semaphores类似于传统的信号量，通常要高效一些。信号量不可用，调用线程需阻塞时，Dispatch semaphores 才会调起内核。
    </td>
</tr>
<tr>
    <td>Dispatch Sources</td>
    <td>
    Dispatch Source 对特定类型的系统事件生成通知，你可以使用 Dispatch Source 来观察类似进程通知、信号或描述符事件等等。当监听的事件发生时，Dispatch Source 异步地提交你的任务代码到指定的队列去处理。
    </td>
</tr>
</table>

## 使用 block 实现任务

block 是基于 C 的语言功能，所以你能在 C、Objective-C、C++ 中使用它。block 能够很容易地定义一个自包含的工作单元，尽管看起来类似于函数指针，其实是用底层的数据结构表示，它类似于对象，并有编译器创建和管理。编译器将打包 block 中你的代码，封装成可以生存在堆上的形式，并且能够在程序中传递。

block 一个很重要的优点就是能够引用自己作用域外的变量，当你在函数或方法中创建 block ，它在某种程度上类似于传统的代码块。例如，block 可以读取上层作用域中的变量。在 block 中访问变量会将其复制到堆上，以便于之后对该变量的访问（栈上的变量一出栈就销毁了）。当 block 被添加到调度队列时，这些被引用的值必须为只读的形式。然而，同步执行的代码块，也可以使用 `__block ` 关键字前缀修饰的变量，并可返回到父作用域中。

你在代码的行内使用类似于函数指针的语法定义 block，block 和函数指针的主要区别是 block 前面使用 `^` 而不是 `*` 。像函数指针一样，可以传递参数给 block，并接受返回值。下面是个🌰：

    //  a simple block example

    int x = 123;
    int y = 456;
    
    void (^aBlock)(int) = ^(int z) {
        printf("%d %d %d\n",x,y,z);
    };
    
    aBlock(789);    // print:123 456 789

下面是一些使用 block 关键规则的总结：

- 对于准备在调度队列中异步执行的 block，可以安全的捕获父作用域中的常量和变量，并在 block中使用。然而，你不应该捕获很大的数据类型或者由上下文创建删除的指针对象。因为在执行 block 的时候，所引用的指针指向的内存可能已经被释放，当然，创建对象并将内存交由 block 管理是安全的。
- 当 block 添加到调度队列时会自动copy，所以你不必显示的去 copy。
- 尽管使用队列要比原生的县城效率高，但是创建 block 和在队列中执行仍然需要一定的开销，如果某个块执行的任务量很小，直接调用可能要比在队列中执行好一些。判断的方法是通过工具来收集一些数据加以判断。
- 不要试图缓存一些底层线程有关的数据，以望另一个不同的 block 能够获取这些数据时容易一些。如果同一个调度队列的任务希望共享数据，请使用调度队列的上下文指针存储数据。
- 如果你的 block 创建了很多的 Objective-C 对象，你可能想把这些代码括在 `@autorelease` block 中来管理内存。虽然 GCD 调度队列有自己的 autorelease pool，但是不能保证这些 pool 不会被 drain。如果你的程序内存比较紧张，创建自己的 autorelease pool 。

## 创建和管理调度队列

在添加 block 到队列前，你需要决定你使用哪种类型的队列，以及如何使用它。调度队列能够串行或并行执行任务。除此之外，如果你心里考虑队列的特殊用途，你也可以配置相应的属性。

### 获取全局的并发队列
如果你有大量的能够并行执行的队列，那么全局队列是一个很好的选择。并发队列仍然是先进先出的队列，然而，也有可能在之前的队列完成之前出队额外的任务。在任意给定的时间，全局并发队列同时执行任务的数量是动态的，受当前程序的影响。许多因素会影响并发数，包括处理器核心，其他进行执行任务的数量，以及其他串行队列中任务的数量以及优先级。

系统为每个程序提供 4 个并发队列，它们对于程序来说是全局的，唯一的区别是其优先级不同，由于它们是全局的，所以你无需显示的创建，使用 `dispatch_global_queue` 函数即可。如下：

    dispatch_queue_t aQueue =     dispatch_get_global_queue(DISPATCH_QUEUE_PRIORTITY_DEFAULT,0)

你还可以通过传入不同的值来获取不同优先级的队列。需要注意的是，第二个参数是为了以后来用，现在传入0 即可。

虽然调度队列是引用计数的对象，但是你不用 retain 和 release 全局队列，因为它在程序中是全局的，retain 和 release 操作都会被忽略。因此，你不要持有它们的引用。当你需要时直接调用函数即可。

### 创建串行调度队列
串行调度队列当希望任务按照指定顺序执行时特别有用，串行队列每次仅执行一个任务，并且从头开始执行，你可能需要串行队列来代替锁的操作，或者保护共享的资源以及其他的数据结构。不像锁，串行调度队列确保执行任务是按照可预见的顺序，只要你同步的提交你的任务到串行队列，那么队列就不会陷入僵局（死锁）

跟并发队列也不一样，你必须显示的创建和管理你需要使用的串行队列，你可以创建任意数量的串行队列，但是要避免创建大量的串行队列以达到同时执行的目的。希望大量任务同时执行时，将其添加到全局队列即可。当创建串行队列时，为每一个队列标识一个符号，这样子在调试时可以更加清晰。

下面的例子，展示了创建串行队列的步骤，`dispatch_queue_create` 函数中第一个参数标识队列名，第二个参数是队列的一些属性，第二个值填 NULL，以后可能会用到。

    dispatch_queue_t queue;
    queue = dispatch_queue_create("com.example.MyQueue",NULL)

除了你自己创建的串行队列，系统自动帮你创建了一条串行队列，并将其绑定在应用程序主线程上。

### 在运行时获取常见的对列
GCD 提供了一些函数，方便你在运行时获取一些常见的队列：

- 使用 `dispatch_get_current_queue` 获取当前队列的标识符，在 block 中调用时，可以获取当前 block 运行队列的标识符，在 block 外调用，将返回程序默认的并发队列。
- 使用 `dispatch_get_main_queue` 获取程序主线程对应的串行队列，这个队列是系统自动创建，调用 dispatch_main 函数或者配置 RunLoop 在主线程上。
- 使用 `dispatch_get_global_queue` 获取全局并发队列的任意一个。

### 调度队列的内存管理
调度队列和其他 dispatch 对象都是引用计数类型。当你创建一个串行队列时，其初始的引用计数为 1，你可以使用 dispatch_retain 或 dispatch_release 函数来增加或者减少引用计数，为 0 时，系统异步的释放这条队列。

retain 和 release dispatch 对象是特别重要的，例如为了确保队列使用时仍存在于内存中，作为 Cocoa 内存管理的对象，基本的守则是如果在代码中打算使用，那么用之前 retain 用完 release。这条基本的守则保证了当你使用时，它仍存在与内存中。

即使你实现了垃圾回收的程序，你仍然需要 retain 和 release 调度队列，因为 GCD 不支持垃圾回收。

### 使用队列存储自定义上下文信息
dispatch 对象（包括调度队列）允许你将对象和自定义上下文联系起来。为了设置或获取给定对象的数据，你使用 `dispatch_set_context` 和 `dispatch_get_context` 函数，系统任何时候都不会使用你自定义的数据，并且适时申请和释放这些数据都是你的责任。

对于队列来说，你可以使用上下文数据来存储 Objective 对象的指针，或者其他的数据结构，能够帮助你标识队列或者你代码的其他用途。你可以使用队列的结束方法在队列本身释放前释放上下文数据。

### 为队列提供清空方法
当你创建串行队列后，你可以为结束方法附加执行自定义的清除操作，当队列释放时被执行。调度队列时引用计数的对象，你可以使用  `dispatch_set_finalizer_f` 方法来指定函数，当引用计数为零时被调用。这个方法只有在上下文数据指针不为 NULL 时才可以被调用

下面展示了一个自定义的终结方法：

    void myFinalizerFunction(void *context)
    {
        myDataContext* theData = (MyDataContext *)context;
        
        // clearn up
        myCleanUpDataContextFunction(theData);
        
        // release 
        free(theData);
    }
    
    dispatch_queue_t createMyQueue()
    {
        MyDataContext* data = (MyDataContext *)malloc(sizeof(MyDataContext));
        myInitializeDataContextfuction(data);
        
        // create the queue and set the context data
        dispatch _queue_t serialQueue =
        dispatch_queue_create("com.example.CriticalTaskQueue",NULL);
        
        if (serialQueue)
        {
            dispatch_set_context(serialQueue, data);
            dispatch_set_finalizer_f (serialQueue, &myFinalizerFunction);
        }
        
        return serialQueue;
    }

## 添加任务到队列中
为了执行任务，我们必须将其添加到调度队列中，可以使用同步或异步的方式，也可以单个或成组的调度。一旦进入队列，考虑到其约束和现有的任务已经在队列中，就尽可能最快的开始执行任务。本章节向您展示一些调度任务的技术，并介绍了各自的优点：

### 添加单个任务到队列中
有两种方式将一个任务添加到队列中，同步的或异步的。如果可能的话，使用 `dispatch_async` 和 `dispatch_async_f` 函数的异步方案要优于同步的代码，当向队列中添加任务后，还没有办法知道具体开始执行的时间。因此，使用异步的方式，能够让你添加完之后继续做其他事情，这一点在主线程中很重要。

尽管你应该尽可能添加异步的任务，但是，有的时候，需要添加同步任务以避免竞争条件或其他一些同步的错误，在这些情况下，你可以使用 `dispatch_sync` 和 `dispatch_sync_f` 函数。这些函数阻塞当前线程的执行，直到任务完成。

> 你永远不应该调用同步任务到你需要执行任务的线程，这样子会造成死锁，在串行队列中会造成死锁，在并发线程中也应该避免这样做。

下面的例子演示了如何同步或异步执行基于 block 的变体：

    dispatch_queue_t myCustomQueue;
    myCustomQueue = dispatch_queue_create("com.example.MyCustomQueue", NULL);
    
    dispatch_async(myCustomQueue, ^{
        printf("Do some work here.\n");
    });
    
    printf("The first block may or may not have run.\n");
    
    dispatch_sync(myCustomQueue, ^{
        printf("Do some more work here.\n");
    });
    
    printf("Both blocks have completed.\n");


### 当任务完成时执行 completion block
实际上，任务在队列中独立的执行，然而，当任务完成时，你的程序可能仍然想知道结果。在传统的异步编程中，你可能使用回调机制来处理，但是在调度队列中，你可以使用 completion block。

completion block 仅仅只是在调度任务时添加在尾部的代码块。调用的代码通常会提供 completion block 作为一个参数，所有任务完成后的代码作为 completion block 提交给队列。

下面展示了一种普遍的用法：

    void average_async(int *data, size_t len,
       dispatch_queue_t queue, void (^block)(int))
    {
       // Retain the queue provided by the user to make
       // sure it does not disappear before the completion
       // block can be called.
       dispatch_retain(queue);
       // Do the work on the default concurrent queue and then
       // call the user-provided block with the results.
       dispatch_async(dispatch_get_global_queue(DISPATCH_QUEUE_PRIORITY_DEFAULT, 0),
    ^{
          int avg = average(data, len);
          dispatch_async(queue, ^{ block(avg);});
          // Release the user-provided queue when done
          dispatch_release(queue);
       });
    }


### 并行执行循环迭代
另一种并发队列特别有用的地方是循环迭代计算，例如，你可能使用 for 循环做一些计算。

如果每次迭代的工作与其他迭代是独立的，而且每次循环执行的顺序不重要，那么可以使用 `dispatch_apply` 或 `dispatch_apply` 函数来替换循环，这样可能会同时执行多个循环。

虽然你在调用函数时可以使用串行队列，但是这样子做没必要。没有什么性能上的优势。

> 重要提示：
> 像普通的 for 循环，`dispatch_apply` 或 `dispatch_apply` 函数在所有的循环迭代完成之前不会返回。因此，你应当在队列的上下文中执行时小心一些，如果传递的参数是串行队列，那么有可能造成死锁。
> 如果当前的任务比较耗时，最好在另一个线程中调用它。

例如，下面这个用 dispatch_apply 替换for循环的例子：

    dispatch_queue_t queue = dispatch_get_global_queue(DISPATCH_QUEUE_PRIORITY_DEFAULT, 0);
    dispatch_apply(count, queue, ^(size_t i) {
       printf("%u\n",i);
    });


你应该确保给每次循环迭代安排合适的工作量，因为有块添加到队列的开销以及调度执行代码的开销。如果每次循环迭代的任务量很小，那么这些开销会让你得不偿失。


### 在主线程中执行任务
GCD 提供了一个特殊的调度队列，能够在主线程上执行任务，这个队列自动给程序设置一个 RunLoop，并对主线程自动 drain 。如果你不想创建 Cocoa 程序，而且不想显示的设置 RunLoop，你必须调用 `dispatch_main` 方法 drain 主线程，当然你也可以添加任务到这个队列，但是它不会执行，直到你调用这个方法。

可以通过调用 `dispatch_get_main_queue` 函数，获取依附主线程的调度队列。因此可以作为应用程序的一个同步点。

### 在任务中使用 Objective-C 对象
GCD 对 Cocoa 内存管理提供了内建的支持，因此你可以在提交到队列的 block 中轻松的使用 Objective-C 的对象。每一个调度队列维护自己的自动引用池来确保自动释放的对象会在某个点释放，队列不能保证他们实际 release 了这些对象。

如果你的程序内存吃紧或 block 创建了很多自动释放的对象，创建你自己的自动引用池是唯一的办法来确保对象适时的 release。如果你 block 创建了很多的对象，可能需要相应的自动引用池。


## 暂停和继续队列
你可以通过挂起队列来暂停 block 的执行，使用 `dispatch_suspend` 函数挂起队列，并增加队列暂停引用计数；使用 `dispatch_resume` 函数继续队列的执行，并减少其引用计数。引用计数大于 0 时，会挂起。因此，你必须保持平衡，使所有的暂停操作都有匹配恢复操作。

> 暂停和恢复调用是异步的，并且只影响 block 的执行，正在执行的 block 不会被打断。

## 使用 dispatch 信号量控制可用的有限资源
如果你提交到队列中的任务需要访问一些有限的资源，可能需要使用 dispatch 信号量来控制同时访问资源的任务数量。dispatch 信号量与传统信号量类似但是有一点不同，当资源可获取时，dispatch 信号量的获取相比传统信号量只需要很少的时间。因为 GCD 在这种特殊情况下并不会调用内核。只有极少数资源无法获取时，才会调用内核，因为系统需要暂停你的线程直到该信号量状态消失。

使用调度信号量语义如下：

1. 当创建调度信号量时（使用 `dispatch_semaphore_create` 函数），你可以指定一个绝对值来表示当前可使用的资源数量。
2. 在每一个任务中，调用 `dispatch_semaphore_wait` 等待信号量
3. 当 wait 调用返回时，使用资源继续工作
4. 当完成与资源有关工作后，释放对资源的引用并且标识信号量，通过 `dispathch_semaphore_signal` 函数。

为了描述这些步骤，我们举一个使用文件描述符的例子。每个程序都给定了有限的文件描述符可以使用，如果你的任务需要处理大量的文件，你肯定不想同一时间打开大量的文件使得文件描述符超出了限制。你可使用信号量来处理这种问题，下面是一些基本的代码：

    // 创建信号量，初始化可用的文件描述符数量
    dispatch_semaphore_t fd_sema = dispatch_semaphore_create(getdtablesize() / 2);
    
    // 等待可用的文件描述符
    dispatch_semaphore_wait(fd_sema, DISPATCH_TIME_FOREVER);
    fd = open("/etc/services", O_RDONLY);
    
    // 完成工作后，释放资源，并且标识信号量
    close(fd);
    dispatch_semaphore_signal(fd_sema);



## 等待一组任务的完成
dispatch groups 是一种可以锁住一个线程直到其他任务完成执行的方式，如果当前任务需要其他一个或多个任务完成时才可以执行，这种情况下可以使用 dispatch groups。另一种使用方式是作为线程同步的替代方案，如果你打算使用多个子线程并且同步执行它们，那么你可以将它们对应的任务添加到 dispatch group 中并等待整个 group 的完成。

下面是一个基本的例子，`dispatch_group_async` 函数将任务与组关联起来，所有执行完成后出队，使用 `dispatch_group_wait` 函数等待一组任务的完成。

    // 创建全局队列
    dispatch_queue_t queue = dispatch_get_global_queue(DISPATCH_QUEUE_PRIORITY_DEFAULT, 0);
    
    // 创建一个组
    dispatch_group_t group = dispatch_group_create();
    
    // 添加任务进组
    dispatch_group_async(group, queue, ^{
       // 处理一些异步的任务
    });
    
    // 做一些其他的工作
    
    // 当你需要等待组中的任务完成后才能继续工作时
    dispatch_group_wait(group, DISPATCH_TIME_FOREVER);
    
    // 完成工作后释放这个组
    dispatch_release(group);

## 调度队列和线程安全

在调度队列中讨论线程安全的问题似乎看起来很奇怪，但是线程安全仍然是一个有意思的话题，当你在程序中使用并发时，下面是你需了解的知识点：

- Dispatch queues 本身是线程安全的，换句话说，系统没有采取锁或者同步这个队列时，你可以在任何线程向 dispatch queue 提交任务。
- 不要在 `dispatch_async` 中传入当前队列，并且对调用它，这会阻塞当前线程，即不要在当前线程中执行同步操作。
- 避免在提交队列的任务中使用锁，尽管这种行为是安全的，但是性能上会由很大影响，如果有需要用到锁的地方，可以使用串行队列来替代。
- 尽管你可以使用线程，但是最好不要这样做，你可以在后面章节了解更多有关调度队列和线程的知识。


> 个人的一点翻译小总结

> 翻译，一般来说要做到信雅达，一般来说，做到信和达就不错了，少数地方雅一下，就会使得翻译的内容更加有味道，但是技术书籍涉及到很多名词，有些名词大家都形成习惯了，就可以顺手翻译过来，但是大多数名词我认为还是不要翻译的好，毕竟是技术文章，不是写给文科生看的，就算要翻译，最好也是在括号中注明原英文

> 另外在翻译的同时，如果严格按照英文的意思来翻译，会显得有些拗口，毕竟老外表达的逻辑跟汉语还是有很大差距，在对技术纯熟了解的基础上，我认为可以随便翻，用自己的话说完全没有问题，但是技术纯熟的人一般不会做这种翻译的事情，所以大多数技术资料还是少数的开发者去翻，或者其他一些专门的翻译。

> 我个人翻译的目的：1. 联系专业英文，为了以后更棒的阅读能力。2.现在在学校，时间多。3.有些文档可能自己需要反复去看，翻一下方便 4.开源精神 5.翻译的同时，我会仔细体会文档每句话的意思，加深理解。






