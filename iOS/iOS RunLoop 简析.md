# iOS RunLoop 简析

## 一、什么是runloop

- **释义**：runloop又叫运行循环，是一种保证线程随时处理事件，在没有事件处理时，不退出而是进行休眠的机制，是一种典型的事件循环机制。查看官方文档可以看到，runloop是一套用于管理**输入源**的接口。
- **Runloop模式**：runloop可以通过模式来筛选需要处理的事件，有时为了体验会做模式的切换，用来提高体验度。
  - Default：默认模式，主线程runloop开启也是默认的模式
  - ConnectionReply：处理NSConnection信息，开发者不需要使用该模式
  - ModalPanel：系统使用处理模态面板事件，开发者不需要使用该模式
  - EventTracking：当用户与应用交互时，主线程会切换default到该模式，该模式专门处理用户的交互输入，该模式会阻止其他事件的处理 
  - common：聚合模式，其他几种模式的聚合体，在该模式下信号源添加后，其内部的各个模式都会自动添加信号源，然后作出响应

## 二、runloop的运行机制

- runloop每次运行需要处理的事件分为三类：Observer 监听事件、Timer定时器事件、Source输入源事件。

  - Observer 监听事件：用来监听Runloop状态变化，当状态发生变更时，会触发对应的回调	

  - Timer定时器事件：特指定时器的回调，当定时器被添加到runloop中后，会根据定时器设定的频率在runloop中注册一系列时间点，当时间点到时，会进行runloop的唤醒，然后进行事件处理

  - source输入源事件分为2种：

    - source0 事件：该事件不会主动触发，需要将其标记为**待处理**之后，手动唤醒runloop进行处理

      > source0是App内部事件，由App自己管理的**UIEvent**、**CFSocket**、**Cocoa Perform Selector Source**都是source0。当一个source0事件准备执行的时候，必须要先把它标记为signal状态。source0是非基于Port的，只包含了一个回调（函数指针），它并不能主动触发事件。使用时，需要先调用 CFRunLoopSourceSignal(source)，将这个 Source 标记为待处理，然后手动调用 CFRunLoopWakeUp(runloop) 来唤醒 RunLoop，让其处理这个事件。以下是source0的结构体：

      ```objective-c
      typedef struct {
          CFIndex 		version;
          void *  		info;
          const void *(*retain)(const void *info);
          void    		(*release)(const void *info);
          CFStringRef (*copyDescription)(const void *info);
          Boolean 		(*equal)(const void *info1, const void *info2);
          CFHashCode  (*hash)(const void *info);
          void    		(*schedule)(void *info, CFRunLoopRef rl, CFStringRef mode);
          void    		(*cancel)(void *info, CFRunLoopRef rl, CFStringRef mode);
          void    		(*perform)(void *info);
      } CFRunLoopSourceContext;
      ```

      

    - source1 事件：主动唤醒runloop进行处理，通常用来线程间通讯和处理硬件接口信号

      > source1由runLoop和内核管理，source1带有mach_port_t，可以接收内核消息并触发回调。如 **CFMachPort**、**CFMessagePort**都是source1。Source1除了包含回调指针外包含一个mach port，Source1可以监听系统端，通过内核和其他线程通信，接收、分发系统事件，它能够主动唤醒RunLoop(由操作系统内核进行管理，例如CFMessagePort消息)。官方也指出可以自定义Source，因此对于CFRunLoopSourceRef来说它更像一种协议，框架已经默认定义了两种实现，如果有必要开发人员也可以自定义，详细情况可以查看[官方文档](https://link.juejin.cn?target=https%3A%2F%2Fdeveloper.apple.com%2Flibrary%2Fcontent%2Fdocumentation%2FCocoa%2FConceptual%2FMultithreading%2FRunLoopManagement%2FRunLoopManagement.html)。

      ```objective-c
      typedef struct {
          CFIndex 	version;
          void *  	info;
          const void *(*retain)(const void *info);
          void    	(*release)(const void *info);
          CFStringRef (*copyDescription)(const void *info);
          Boolean 	(*equal)(const void *info1, const void *info2);
          CFHashCode  (*hash)(const void *info);
      #if (TARGET_OS_MAC && !(TARGET_OS_EMBEDDED || TARGET_OS_IPHONE)) || (TARGET_OS_EMBEDDED || TARGET_OS_IPHONE)
          mach_port_t (*getPort)(void *info);
          void *  (*perform)(void *msg, CFIndex size, CFAllocatorRef allocator, void *info);
      #else
          void *  (*getPort)(void *info);
          void    (*perform)(void *info);
      #endif
      } CFRunLoopSourceContext1;
      ```

      

## 三、runloop与线程的关系

- 线程与Runloop是一一对应的
  - 线程与runloop的对应关系被存放在 **一个全局字典对象中**
  - 对于**主线程**来说，程序运行时，会**自动创建runloop并开启**
  - 开发者手动创建的**子线程**，并不是一开始就创建了其对应的runloop对象，而是**采用懒加载的方式**。只有开发者**操作这个runloop对象时，才会被创建**

## 三、runloop有什么用

- ### 保证程序不退出

  - 我们知道程序的执行是根据代码的逻辑顺序由前向后执行，但在iOS的程序中，应用没有在执行到某个节点就停止，而是一直运行直到系统或者用户主动关闭程序，而这就是runloop的作用

- ### 处理事件

  - 在程序启动后，主线程会自动开启其对应的runloop，之后程序会进入一个无线循环的状态，每一次循环过程中，runloop都会对 **硬件接口信号**、**用户操作信号**、**页面刷新任务**、**开发者指定的任务** 做出处理。

## 四、如何使用runloop

- NSTimer

  ```
  [NSTimer scheduledTimerWithTimeInterval:1.0f target:self selector:@selector(testTimer) userInfo:nil repeats:YES];
  ```

  > 定时器的注意点模式冲突的问题，比如deftault与EventTracking，需要注意。

- ImageView显示

  ```
  [self.imageView performSelector:@selector(setImage:) withObject:[UIImage imageNamed:@"xxx"] afterDelay:2.0 inModes:@[NSRunLoopCommonModes]];
  ```

  > 同样，imageview的显示也要注意模式的冲突，使用聚合模式。

- 监控卡顿

  - 通过监控，runloop的状态切换的时间差，设定最大时间段，超过时间就定义为异常信息

- 线程保活

  ```
  //此种方法不太稳定
  [[NSRunLoop currentRunLoop] addPort:[NSPort port] forMode:NSDefaultRunLoopMode];
  [[NSRunLoop currentRunLoop] run];
  [self performSelector:@selector(test:) onThread:_thread withObject:nil waitUntilDone:YES];
  ```

