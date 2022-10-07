---
layout: post
title: "iOS 并发编程指南(四)-Dispatch Sources"
date: 2015-05-08 10:38:38 +0800

tags: [并发编程, GCD, 翻译]
---

每当你与系统底层进行交互时，你必须要做好这种任务可能会大量耗时的准备。跟在自己的进程内相比较，调用内核或者其他系统层次涉及到的改变是相当耗时的。因为这些原因，许多系统的库都提供了异步的 API，允许你的代码向系统提交请求后继续做别的工作，这时请求仍在进行中。GCD 就是依据这种方式，允许你提交请求，并且使用 block 和调度队列来回调请求结果。

<!--more-->

## 关于 Dispatch Sources

Dispatch Source 是协调底层特殊系统事件的基本类型。GCD 支持以下 Dispatch Sources：

- 定时器 dispatch source ：生成周期性的通知
- 信号 dispatch source：当 NUIX 信号到达时通知你
- 描述符 source：通知你一些基于文件和套接字的操作，例如，当数据能够读取时、可以写入数据时、文件系统中文件被删除移动或重命名时、当文件的元信息被修改时
- 进程 disp source：通知你一些有关进程的事件，例如：进程存在时、进程发出 `fork` 或者 `exec` 类型的调用、当信号送达进程时
- 端口 dispatch source：通知你有关端口的相关事件
- 自定义 dispatch source：你自己定义和触发的

Dispatch sources 用来取代通常用于系统相关事件监控的异步回调函数。当你配置一个 dispatch source 时，你需要指定想要监听的事件、调度队列以及用于处理事件的代码。你可以使用 block 或者函数来处理事件。当监听的事件到达时，dispatch source 提交你的 block 或函数到指定的队列中去执行。

跟手动提交任务到队列相比，dispatch source 为应用程序提供了连续的事件源。dispatch source 保持附加在其队列上，除非你明确的取消它。当附加后，相应的事件发生后，它将提交对应的任务代码块到队列上。比如一些定时器事件，这种事件周期性的发生但大多数时间会发生在某些特定的状态下。因此，dispatch source 会 `retain` 自己的队列，防止过早释放，因为很可能释放后下一个事件发生后找不到队列。

为了防止队列中事件太多发生积压，dispatch source 实现了一种合并机制。如果新的事件已经到达，而队列中上一个事件尚未被处理，那么可以将这两个事件的数据合并后作为一个新的事件。根据事件类型的不同，合并的结果可能是取代旧事件或者更新信息。例如在基于信号的 dispatch source 中，仅提供最近的信号信息和距离上次处理之后，以及上次信号处理之后总共交付了多少信号。

## 创建 Dispatch Sources
创建 dispatch 除创建它自己以外还需要创建事件源，事件源就是处理事件需要的本地数据结构。举个🌰，基于描述符的 dispatch source，你需要用它打开描述符。基于进程的源，你需要获取目标程序的进程 ID。当你有了事件源后，可以按照下面描述来创建对应的 dispatch source：

1. 使用 `dispatch_source_create` 函数创建 dispatch source
2. 配置 dispatch source：给它分配一个事件处理操作；对于定时器源，使用 `dispatch_source_set_timer` 函数设置定时器信息；有时候，还需要为其分配取消处理操作；调用 `dispatch_resume` 函数来开始处理事件。

由于 dispatch source 在使用前需要一些额外的配置，所以 `dispatch_source_create` 函数返回的是一个挂起的状态。在挂起时，它接收事件但是不做处理。这样子就有了时间来设置事件处理操作和其他额外的配置。

下面几部分介绍了如何配置 dispatch source。

### 编写和配置事件处理操作
为了处理 dispatch source 产生的事件，你必须定义事件处理操作。事件处理操作可以是一个函数或者 block ，然后可以用 `dispatch_source_set_event_handler` 或 `dispatch_source_set_event_handler_f` 函数进行设置。当事件发生后，将提交事件处理操作到指定的队列上来处理事件。

事件处理操作需要处理所有可能到达的事件，如果事件处理操作已经在处理中，并且等待下一个事件，如果这时又来一个新的事件，dispatch source 会将这两个事件合并。事件处理操作通常看起来一直处理最新的操作，其实很可能处理多个事件的合并版。如果一个或多个新的事件到达时，它已经开始执行了，那么 dispatch source 将保留这些事件并且再次向队列请求。

基于函数的事件处理程序采用单个上下文指针，包含 dispatch source 对象，并且不返回值。基于 block 的事件处理程序没有参数也不返回值。

    // block-based event handler
    void (^dispatch_block_t)(void)

    // Function-based event handler
    void (*dispatch_function_t)(void *)


在事件处理程序中，你可以从 dispatch source 提供的事件中获取相应信息。由于基于函数的事件处理程序已经被传递了 dispatch source 的指针，所以基于 block 的事件处理程序需要自己捕获这个指针。例如下面的程序段：

    dispatch_source_t source = dispatch_source_create(DISPATCH_SOURCE_TYPE_READ, myDescriptor, 0 ,myQueue);

    dispatch_source_set_event_handler(source, ^{
        // 通过捕获的 source 对象获取信息
        size_t estimated = dispatch_source_get_data(source);
    
        // continued
    });

在 block 中捕获变量通常会更加动态灵活，当然，默认捕获的变量是只读的。尽管 block 提供了可以修改变量的功能，但是最好不要去尝试使用。Dispatch Sources 总是异步执行其事件处理程序。所以可能你捕获的变量随着事件处理程序的执行已经消失了。

下表列出了可以在事件处理程序中获取有关事件信息的函数

<table>
<tr>
    <th>函数</th>
    <th>说明</th>
</tr>
<tr>
    <td>dispatch_source_get_handle</td>
    <td>
    这个函数返回 dispatch source 管理的底层数据类型<br/><br/>
    
    对于描述符源，返回 int 类型，包含其对应的描述符<br/><br/>
    
    对于信号源，返回 int 类型，包含最近事件的信号数量<br/><br/>
    
    对于进程源，返回 pid_t 类型的数据结构，对应于被监控的进程<br/><br/>
    
    对于端口源，返回 mach_port_t 数据结构<br/><br/>
    
    对于其他源，其返回值是不明确的<br/><br/>
    </td>
</tr>
<tr>
    <td>dispatch_source_get_data</td>
    <td>
    函数返回对应事件产生的数据<br/><br/>
    
    对于描述符源，如果是读取数据，返回可读取的字节数；<br/>如果是写入数据，返回可写入的字节数；<br/>如果是监控文件系统的活动，返回一个常量指示事件发生的类型。其是一个 dispatch_source_vnode_flags_t 的枚举类型<br/><br/>
    
    对于进程源，返回常量，指示哪种进程事件发生，参考 dispatch_source_proc_flags_t 枚举类型<br/><br/>
    
    对于端口源，返回指示事件类型的常量，参考 dispatch_source_machport_flags_t <br/><br/>
    
    对于自定义源，返回由 dispatch_source_merge_data 函数合并现值和新值后创建的新值<br/><br/>
    
    </td>
</tr>
<tr>
    <td>dispatch_source_get_mask</td>
    <td>
    函数返回事件标识用于创建 dispatch source<br/><br/>
    
    对于进程，返回 dispatch source 收到事件的掩码，参照 dispatch_source_proc_flags_t 枚举类型<br/><br/>
    
    对于端口，返回期望事件的掩码，参照 dispatch_source_mach_send_flags_t 枚举类型<br/><br/>
    对于自定义类型，返回用于合并数据的掩码
    </td>
</tr>
</table>



### 写入一个取消处理程序

取消处理程序在 dispatch source 释放之前执行清空工作。多数类型的 dispatch source 不需要取消处理程序，除非你对 dispatch source 有自定义的行为需要在释放时执行。但是使用描述符或 Mach port 的 dispatch source 必须设置取消处理程序，用于关闭描述符或释放端口。否则可能导致微妙的 Bug，这些结构体会被系统其他部分或你的应用在不经意间重用。

你可以在任何时候加入取消处理程序，但通常我们在创建 dispatch source 时加入。使用 `dispatch_source_set_cancel_handler` 或 `dispatch_source_set_cancel_handler_f` 函数来设置取消处理程序。

    dispatch_source_set_cancel_handler(mySource,^{
        close(fd);
    });


### 修改目标队列
在创建 dispatch source 时可以指定一个队列，用于执行事件处理器和取消处理器。不过你也可以使用 `dispatch_set_target_queue` 函数在任何时候修改目标队列。修改队列可以改变执行dispatch source 事件的优先级。

修改 dispatch  source 是异步操作，也就是说已经执行的事件不会对其造成影响。之后的事件会到修改后的队列中执行。

### 关联自定义数据到 dispatch source
和 GCD 的其他类型一样，你可以使用 `dispatch_set_context` 函数关联自定义数据到 dispatch source。使用 context 指针存储处理器需要的数据。存储数据之后，就必须要加入取消处理器，因为你需要在取消操作中释放 context 自定义数据。

如果你使用 block 实现事件处理器，那么可以捕获本地变量在 block 中使用。虽然这样可以代替 context 指针，但是应该谨慎地使用这种方式。因为 dispatch source 可能长时间存在应用程序的生命周期中，block 捕获指针变量时需要特别小心，因为指针指向的数据可能会被释放，因此需要赋值这些数据或者 retain 指针。不管哪一种方式，你都需要提供一个取消处理器来释放这些数据。

### Dispatch Source 的内存管理
Dispatch Source 也是引用计数的数据类型，初始值为 1 ，可以使用 `dispatch_retain` 和 `dispatch_release` 函数来增加和减少引用计数。当引用计数为 0 时，系统自动释放 Dispatch Source 数据结构。

Dispatch Source 的所有权可以由它本身内部或外部进行管理。使用外部所有权时，另一个对象拥有 dispatch source，并负责在适当的时候释放。虽然外部所有权的方式比较常见，当你希望创建自主的 dispatch source ，并让它管理自己的行为时，可以使用内部所有权。例如 dispatch source 应用单一全局事件时，可以让它自己处理时间并退出。

## 使用 Dispatch Source 的一些例子
下面各节简要讲述了几个实用 Dispatch Source 的示例

### 创建定时器
定时器 dispatch source 定时产生事件，可以用来发起定时执行的任务，如在游戏或图形应用中，用来刷新屏幕或者实现动画。当然也可以设置定时器用于经常性的检查服务器的最新信息。

所有定时器 dispatch source 都是间隔定时器，即一旦创建，就按照指定的间隔周期递送事件。你需要为定时器指定一个期望的 leeway 值，让系统能够灵活地管理电源并唤醒内核。例如系统可以使用 leeway 值来提前或者延迟触发定时器，使其更好地与其他系统事件结合。

> 即使你指定 leeway 值为 0，也不要期望定时器能够十分精准的按照你设置的时间触发事件，系统只能保证尽最大可能满足你的需求

当计算机处于休眠状态时，所有的定时器源都将挂起。当计算机唤醒后，定时器源也自动的唤醒。根据你的配置，暂停定时器可能会影响下一次的触发。如果定时器 dispatch source 使用 `dispatch_time` 函数或 `DISPATCH_TIME_NOW` 常量设置，定时器源使用系统默认时钟来确定何时触发，但是默认时钟在计算机睡眠时不会继续。

如果你使用 `dispatch_walltime` 函数设置定时器源，那么它将跟随处理器时间启动定时器。这种方式比较适合触发间隔比较大的场合，可以防止定时器触发间隔出现太大的误差。

下面展示了一个定时器的例子，三十秒启动一次，误差范围一秒。由于时间间隔比较大，所有使用 `dispatch_walltime` 函数创建定时器源，定时器在创建后立即触发，随后美 30 秒触发一次。`MyPeriodicTask` 和 `MyStoreTimer` 是自定义函数，用于实现定时器的行为，并存储定时器到相应的数据结构。

    // 创建定时器源
    dispatch_source_t CreateDispatchTimer(    uint64_t interval, 
                                              uint64_t leeway,
                                              dispatch_queue_t queue,
                                              dispatch_block_t block)
    {
        dispatch_source_t timer timer = dispatch_source_create(DISPATCH_SOURCE_TYPE_TIMER,0,0,queue);
        if (timer) {
            dispatch_source_set_timer(timer, dispatch_walltime(NULL,0), interval,leeway);
            dispatch_source_set_event_handler(timer, block);
            dispatch_resume(timer);
        }
        return timer;
    }


    void MyCreateTimer()
      {
         dispatch_source_t aTimer = CreateDispatchTimer(30ull * NSEC_PER_SEC,
                                     1ull * NSEC_PER_SEC,
                                     dispatch_get_main_queue(),
                                     ^{ MyPeriodicTask(); });
         // Store it somewhere for later use.
          if (aTimer)
          {
              MyStoreTimer(aTimer);
          }
    }


虽然创建定时器可以解决很多基于时间的事件，但是有时候仍然有其他更好的方式。比如你想在一段时间后执行一次某个 block，你可以使用 `dispatch_after` 或 `dispatch_after_f`函数。这个函数除了可以指定时间延迟和队列外，基本和 `dispatch_async` 差不多，时间延迟可以根据你的需求选择绝对时间或相对时间。

### 通过描述符读取数据
通过文件或套接字获取数据时，需要打开文件或套接字并为之创建一个 `DISPATCH_SOURCE_TYPE_READ` 类型的 dispatch source。指定的事件处理器需要能够读取数据并能处理这些数据。例如对于文件读取，需要为之创建适合的数据结构来存储读取的数据。对于套接字需要对从服务器收到的最新数据做处理。

读取数据时，你总是应该对描述符使用非阻塞的操作，虽然你可以使用 `dispatch_source_get_data` 函数查看当前有多少数据可读，但是你调用之后再读取时，这个数据量可能会变化。如果底层文件被截断或者发生网络错误，从描述符中读取数据时会阻塞当前进程，执行会停止在事件处理器中并阻止 dispatch queue 去执行其他任务。对于串行的队列，还可能造成死锁，即使是并发的队列，也会减少队列能够并发执行的数量。

下面例子配置dispatch source从文件中读取数据，事件处理器读取指定文件的全部内容到缓冲区，并调用一个自定义函数来处理这些数据。调用方可以使用返回的dispatch source在读取操作完成之后，来取消这个事件。为了确保dispatch queue不会阻塞，这里使用了fcntl函数，配置文件描述符执行非阻塞操作。dispatch source安装了取消处理器，确保最后关闭了文件描述符。


    dispatch_source_t ProcessContentsOfFile(const char* filename)
    {
        // Prepare the file for reading.
        int fd = open(filename, O_RDONLY);
        if (fd == -1)
            return NULL;
        fcntl(fd, F_SETFL, O_NONBLOCK);  // Avoid blocking the read operation
        dispatch_queue_t queue = dispatch_get_global_queue(DISPATCH_QUEUE_PRIORITY_DEFAULT, 0);
        dispatch_source_t readSource = dispatch_source_create(DISPATCH_SOURCE_TYPE_READ, fd, 0, queue);
        if (!readSource)
        {
            close(fd);
            return NULL;
        }

        // Install the event handler
        dispatch_source_set_event_handler(readSource, ^{
            size_t estimated = dispatch_source_get_data(readSource) + 1;
            // Read the data into a text buffer.
            char* buffer = (char*)malloc(estimated);
            if (buffer)
            {
                ssize_t actual = read(fd, buffer, (estimated));
                Boolean done = MyProcessFileData(buffer, actual);  // Process the data.
            
                // Release the buffer when done.
                free(buffer);
                // If there is no more data, cancel the source.
                if (done)
                    dispatch_source_cancel(readSource);
            }
        });
    
        // Install the cancellation handler
        dispatch_source_set_cancel_handler(readSource, ^{close(fd);});
        // Start reading the file.
        dispatch_resume(readSource);
        return readSource;
    }


在这个例子中，自定义的 MyProcessFileData 函数确定读取到足够的数据，返回YES告诉dispatch source读取已经完成，可以取消任务。通常读取描述符的dispatch source在还有数据可读时，会重复调度事件处理器。如果socket连接关闭或到达文件末尾，dispatch source自动停止调度事件处理器。如果你自己确定不再需要dispatch source，也可以手动取消它。

### 通过描述符写入数据
向文件或socket写入数据非常类似于读取数据，配置描述符为写入操作后，创建一个 DISPATCH_SOURCE_TYPE_WRITE 类型的dispatch source，创建好之后，系统会调用事件处理器，让它开始向文件或socket写入数据。当你完成写入后，使用 dispatch_source_cancel 函数取消dispatch source。

写入数据也应该配置文件描述符使用非阻塞操作，虽然 dispatch_source_get_data 函数可以查看当前有多少可用写入空间，但这个值只是建议性的，而且在你执行写入操作时可能会发生变化。如果发生错误，写入数据到阻塞描述符，也会使事件处理器停止在执行中途，并阻止dispatch queue执行其它任务。串行queue会产生死锁，并发queue则会减少能够执行的任务数量。

下面是使用dispatch source写入数据到文件的例子，创建文件后，函数传递文件描述符到事件处理器。MyGetData函数负责提供要写入的数据，在数据写入到文件之后，事件处理器取消dispatch source，阻止再次调用。此时dispatch source的拥有者需负责释放dispatch source。


    dispatch_source_t WriteDataToFile(const char* filename) 
    { 
        int fd = open(filename, O_WRONLY | O_CREAT | O_TRUNC, 
                          (S_IRUSR | S_IWUSR | S_ISUID | S_ISGID)); 
        if (fd == -1) 
            return NULL; 
        fcntl(fd, F_SETFL); // Block during the write. 
  
        dispatch_queue_t queue = dispatch_get_global_queue(DISPATCH_QUEUE_PRIORITY_DEFAULT, 0); 
        dispatch_source_t writeSource = dispatch_source_create(DISPATCH_SOURCE_TYPE_WRITE, 
                                fd, 0, queue); 
        if (!writeSource) 
        { 
            close(fd); 
            return NULL; 
        } 
  
        dispatch_source_set_event_handler(writeSource, ^{ 
            size_t bufferSize = MyGetDataSize(); 
            void* buffer = malloc(bufferSize); 
  
            size_t actual = MyGetData(buffer, bufferSize); 
            write(fd, buffer, actual); 
  
            free(buffer); 
  
            // Cancel and release the dispatch source when done. 
            dispatch_source_cancel(writeSource); 
        }); 
  
        dispatch_source_set_cancel_handler(writeSource, ^{close(fd);}); 
        dispatch_resume(writeSource); 
        return (writeSource); 
    } 


### 监听文件系统对象
如果需要监控文件系统对象的变化，可以设置一个 DISPATCH_SOURCE_TYPE_VNODE 类型的dispatch source，你可以从这个dispatch source中接收文件删除、写入、重命名等通知。你还可以得到文件的特定元数据信息变化通知。

在dispatch source正在处理事件时，dispatch source中指定的文件描述符必须保持打开状态。

下面例子监控一个文件的文件名变化，并在文件名变化时执行一些操作(自定义的 MyUpdateFileName 函数)。由于文件描述符专门为dispatch source打开，dispatch source安装了取消处理器来关闭文件描述符。这个例子中的文件描述符关联到底层的文件系统对象，因此同一个dispatch source可以用来检测多次文件名变化。

    dispatch_source_t MonitorNameChangesToFile(const char* filename) 
    { 
       int fd = open(filename, O_EVTONLY); 
       if (fd == -1) 
          return NULL; 
  
       dispatch_queue_t queue = dispatch_get_global_queue(DISPATCH_QUEUE_PRIORITY_DEFAULT, 0); 
       dispatch_source_t source = dispatch_source_create(DISPATCH_SOURCE_TYPE_VNODE, 
                    fd, DISPATCH_VNODE_RENAME, queue); 
       if (source) 
       { 
          // Copy the filename for later use. 
          int length = strlen(filename); 
          char* newString = (char*)malloc(length + 1); 
          newString = strcpy(newString, filename); 
          dispatch_set_context(source, newString); 
  
          // Install the event handler to process the name change 
          dispatch_source_set_event_handler(source, ^{ 
                const char*  oldFilename = (char*)dispatch_get_context(source); 
                MyUpdateFileName(oldFilename, fd); 
          }); 
  
          // Install a cancellation handler to free the descriptor 
          // and the stored string. 
          dispatch_source_set_cancel_handler(source, ^{ 
              char* fileStr = (char*)dispatch_get_context(source); 
              free(fileStr); 
              close(fd); 
          }); 
  
          // Start processing events. 
          dispatch_resume(source); 
       } 
       else 
          close(fd); 
  
       return source; 
    } 


### 监听信号
应用可以接收许多不同类型的信号，如不可恢复的错误(非法指令)、或重要信息的通知(如子进程退出)。传统编程中，应用使用 sigaction 函数安装信号处理器函数，信号到达时同步处理信号。如果你只是想信号到达时得到通知，并不想实际地处理该信号，可以使用信号dispatch source来异步处理信号。

信号dispatch source不能替代 sigaction 函数提供的同步信号处理机制。同步信号处理器可以捕获一个信号，并阻止它中止应用。而信号dispatch source只允许你监测信号的到达。此外，你不能使用信号dispatch source获取所有类型的信号，如SIGILL, SIGBUS, SIGSEGV信号。

由于信号dispatch source在dispatch queue中异步执行，它没有同步信号处理器的一些限制。例如信号dispatch source的事件处理器可以调用任何函数。灵活性增大的代价是，信号到达和dispatch source事件处理器被调用的延迟可能会增大。

下面例子配置信号dispatch source来处理SIGHUP信号，事件处理器调用 MyProcessSIGHUP 函数，用来处理信号。

    void InstallSignalHandler() 
    { 
       // Make sure the signal does not terminate the application. 
       signal(SIGHUP, SIG_IGN); 
  
       dispatch_queue_t queue = dispatch_get_global_queue(DISPATCH_QUEUE_PRIORITY_DEFAULT, 0); 
       dispatch_source_t source = dispatch_source_create(DISPATCH_SOURCE_TYPE_SIGNAL, SIGHUP, 0, queue); 
  
       if (source) 
       { 
          dispatch_source_set_event_handler(source, ^{ 
             MyProcessSIGHUP(); 
          }); 
  
          // Start processing signals 
          dispatch_resume(source); 
       } 
    } 


### 监听进程
进程dispatch source可以监控特定进程的行为，并适当地响应。父进程可以使用dispatch source来监控自己创建的所有子进程，例如监控子进程的死亡;类似地，子进程也可以使用dispatch source来监控父进程，例如在父进程退出时自己也退出。

下面例子安装了一个进程dispatch source，监控父进程的终止。当父进程退出时，dispatch source设置一些内部状态信息，告知子进程自己应该退出。MySetAppExitFlag 函数应该设置一个适当的标志，允许子进程终止。由于dispatch source自主运行，因此自己拥有自己，在程序关闭时会取消并释放自己。

    void MonitorParentProcess() 
    { 
       pid_t parentPID = getppid(); 
  
       dispatch_queue_t queue = dispatch_get_global_queue(DISPATCH_QUEUE_PRIORITY_DEFAULT, 0); 
       dispatch_source_t source = dispatch_source_create(DISPATCH_SOURCE_TYPE_PROC, 
                                                          parentPID, DISPATCH_PROC_EXIT, queue); 
       if (source) 
       { 
          dispatch_source_set_event_handler(source, ^{ 
             MySetAppExitFlag(); 
             dispatch_source_cancel(source); 
             dispatch_release(source); 
          }); 
          dispatch_resume(source); 
       } 
    }

## 取消 dispatch source
除非你显式地调用 dispatch_source_cancel 函数，dispatch source将一直保持活动，取消一个dispatch source会停止递送新事件，并且不能撤销。因此你通常在取消dispatch source后立即释放它：


    void RemoveDispatchSource(dispatch_source_t mySource) 
    { 
       dispatch_source_cancel(mySource); 
       dispatch_release(mySource); 
    } 


取消一个dispatch source是异步操作，调用 dispatch_source_cancel 之后，不会再有新的事件被处理，但是正在被dispatch source处理的事件会继续被处理完成。在处理完最后的事件之后，dispatch source会执行自己的取消处理器。

取消处理器是你最后的执行机会，在那里执行内存或资源的释放工作。例如描述符或mach port类型的dispatch source，必须提供取消处理器，用来关闭描述符或mach port

### 挂起和恢复 dispatch source
你可以使用 dispatch_suspend 和 dispatch_resume 临时地挂起和继续dispatch source的事件递送。这两个函数分别增加和减少dispatch 对象的挂起计数。因此，你必须每次 dispatch_suspend 调用之后，都需要相应的 dispatch_resume 才能继续事件递送。

挂起一个dispatch source期间，发生的任何事件都会被累积，直到dispatch source继续。但是不会递送所有事件，而是先合并到单一事件，然后再一次递送。例如你监控一个文件的文件名变化，就只会递送最后一次的变化事件。






