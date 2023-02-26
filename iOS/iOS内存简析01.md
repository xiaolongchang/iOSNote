# iOS内存简析

## 一、重要概念

1. ### 虚拟内存：

   ​	在CPU访问内存的过程中，操作系统为了保障内存物理地址空间的安全性，使用虚拟寻址的方式开辟一个独立的、私有的、连续的地址空间，当然这个空间不是真实存在的。处理器和它的内存管理单元（MMU）维护一个**页表**来将程序的逻辑地址空间中的页面映射到计算机 RAM 中的硬件地址。当程序代码访问内存中的地址时，MMU 使用页表将指定的逻辑地址转换为实际的硬件内存地址，这种转换是自动发生的。

   - 虚拟内存保护了每个进程的的地址空间
   - 简化了内存管理(因为是一段独立、私有、连续的地址空间)
   - 利用硬盘空间拓展了内存空间

2. ### 物理内存

   ​	是相对于虚拟内存而言的，物理内存指通过物理内存条等硬件而获得的内存空间。

3. ### 内存分页

   ​	内存分页(Paging)是在使用MMU的基础上，提出的一种内存管理机制。它将虚拟地址和物理地址按固定大小，分割成页(page)和页帧(page frame)，并保证页与页帧的大小相同。虚拟地址空间划分成称为页（page）的单位，而相应的物理地址空间也被进行划分，单位是页帧(frame)，一个在内存，一个在磁盘，页和页桢的大小必须相同。

   - 支持了物理内存的离散使用
   - 方便了操作系统对物理内存的管理，最大化利用物理内存
   - 可以采用页面调度算法加大翻译(虚拟地址到物理地址)效率

4. ### CleanMemory

   ​	在内存分页中，并不是所有的分页都相同，其中分为cleanMemory、dirtyMemory、compressedMemory。cleanMemory 在一般的操作系统中指的是 可供Page Out 的部分，而page out指的是降优先级较低的内存交换到磁盘上的操作，但是**iOS中并没有内存交换的机制**，所以在iOS中这么定义cleanMemory是不准确，而准确的定义应该是在**iOS系统中cleanMemory是指是能被重新创建的内存**，主要包含以下几个类别：

   | 类别                          | 注意点                                                      |
   | ----------------------------- | ----------------------------------------------------------- |
   | App的二进制可执行文件         |                                                             |
   | Framework 中的 _DATA_CONST 段 | 当framework被使后，则不属于cleanMemory                      |
   | 文件映射的内存                | 该内存对应的文件是**只读**属性，如果不是则不属于cleanMemory |
   | 待写入数据的内存              | 未被写入数据的内存                                          |

5. ### DirtyMemory

   ​	在内存分页中，所有不属于cleanMemory的内存，都是dirtyMemory。这部分内存不能够被系统重新创建，所以会一直占据着物理内存，直至物理内存不够使用时，会被系统给清理掉。

6. ### CompressedMemory

   ​	当物理内存达到使用上限的时候，系统被将部分物理内存进行压缩操作，以达到节约内存的目的，这部分内存就是compressedMemory。当这部分内存需要重新使用的时候，会对它进行解压操作。

   

## 二、iOS系统内存机制

1. 使用了虚拟内存机制
2. 单App可使用内存较大
   - 中高端型号上接近总内存的45%~55%左右
3. 没有使用内存交换机制
   - 大多数移动设备不支持内存交换
   - 移动设备大都是闪存(Flash)，读写效率相比电脑等设备较低
   - 移动设备容量受限，闪存读写寿命受限
4. 内存告警机制
   - 当内存不够使用时，系统会发出内存警告，告知进程去清理内存
5. OOM崩溃(Out Of Memoory Crash)
   - 当收到内存警告后，进程清理了内存或者没有清理内存，此时内存不能满足使用，就会发生OOM崩溃

## 三、iOS-App内存管理

​	以上的内容，大都是在说明iOS系统层级的内存情况，是由操作系统自动完成的。而需要我们关注是进程内部的内存管理，也就是运行在iOS系统上的App的内存管理。

1. #### App内存地址分布

   |       内存分布区域       |    地址分布    | 说明                                                         | 读写      | 内存管理                   |
   | :----------------------: | :------------: | :----------------------------------------------------------- | --------- | -------------------------- |
   |           栈区           | 高地址(0xF..F) | 1.连续的内存区域<br />2.FILO先进后出，读写效率高，通常以0x7开头<br />3.栈内存较小，由编译器分配<br />4.通常存放函数参数、局部变量、对象指针等数据 | ReadWrite | 自动                       |
   |           堆区           |       ↓        | 1.不连续的内存区域，且会向高地址扩展<br />2.FIFO先进先出，读取速度慢，通常以0x6开头<br />3.访问需先获取对象地址(在栈区)，拿到地址在堆区获取对象 | ReadWrite | 手动                       |
   | 全局区<br />(全局静态区) |       ↓        | 1.程序运行过程中，该区域一直存在<br />2.通常以0x1开头，存放全局静态数据<br />3.其中已初始化的.data区域<br />4.其中未初始化的.bss区域 | ReadWrite | 程序结束后<br />由系统释放 |
   |          常量区          |       ↓        | 1.存放常量(字符串、整形、浮点、字符型等)<br />2.程序运行期间不能被改变 | ReadOnly  | 程序结束后<br />由系统释放 |
   |          代码区          | 低地址(0x0..0) | 1.存放程序的二进制代码<br />2.程序运行期间不能被改变         | ReadOnly  | 程序结束后<br />由系统释放 |

2. #### 引用计数管理

   引用计数是中一种高效的内存管理方式，在iOS中每个对象都有属于自己的引用计数器。当计数器不为0时，表示当前对象是有效对象。如果计数器为0，则表示该对象已经不被任何其他对象引用，即不在被需要，然后就会被回收掉。

   - MRC 手动引用计数

     ①：自己生成的对象，自己持有

     ②：自己强引用的对象，自己也可持有

     ③：谁持有，谁释放

   - ARC 自动引用计数

     ①：编译器自动写入 retain release 

     ②：使用更加底层C语言方法管理引用计数，提高性能

     ③：ARC变量声明时，需要标明其所有资权，包括：

     ​	α：__autoreleasing：修饰的变量赋值后，会被ARC处理为自动释放对象

     ​	β：__strong：修饰的对象被ARC处理后引用计数会增加，离开作用域后引用技术会减少

     ​	γ：__unsafe_unretained：描述弱引用，当修饰对象被释放后，指针继续保存着之前的地址，可能产生僵尸对象，

     ​											访问肯呢个会出现不安全情况。

     ​	Δ：__weak：对原对象弱引用，不改变引用计数，当原对象被释放后，此对象指针会被自动置nil

     ④：自动释放池：提供了一种延迟调用对象 release 的操作途径

## 四、常见的内存问题

1. #### 内存泄露（memory leak）

   - 定义：内存泄露是指，程序在申请内存后，无法释放已申请的内存。

   - 解决方式：

     ①静态分析Analyze，可以发现大部分问题，但是并不很准确，因为只是静态分析。

     ②动态分析Instrument工具库里中的Leaks，推荐使用，可以覆盖运行时发生的泄露问题。

   - 出现原因：

     ①：循环引用

     ②：野指针

2. #### 内存溢出（out of memory）

   ​	是指程序在申请内存时，没有足够的内存空间供其使用，导致程序Crash。

3. #### 循环引用 

   - 循环引用产生的原因：两个以上对象，相互引用，形成引用环。引用环的出现导致，对象引用计数不能为0，从而导致内存的持续占用，出现内存泄露。

   - 常见的循环引用：

     ​	①：代理strong修饰  

     ​	②：Block内部使用self  

     ​	③：多个类相互引用 

     ​	④：定时器不规范使用

   - 如何发现循环引用：

     ​	a：Xcode Memory Debug Graph

     ​	b：Instruments --> Leaks

   - 解决循环引用：断开引用环，weak操作，统一管理类等方式

4. #### 野指针

   - 定义：指针指向的对象被释放或者回收，但是没有对该指针做任何操作，以至于该指针还指向了之前对象的内存区域，该内存区域或许已被回收，或已经被重新使用，导致依赖该指针重新访问内存时，出现错误。

   - 野指针出现的原因：指针变量没有初始化、指针指向的内存释放后之后指针未置空

   - 如何发现野指针：

     ①使用三方库

     ②开启Xcode 僵尸对象检测

   - 如何解决：

     ​	①对已释放内存进行重新填充，从而造成野指针访问必然崩溃去定位问题，xcode->malloc scribble。腾讯bugly借鉴这一原理，通过修改`free`函数，对已释放对象进行非法数据填充，也有效的提高了野指针的崩溃率。但是这种方式，风险较高且对底层API能力要求较高，hook free的方法覆盖范围也较广C函数也能包含。

     ​	②利用Zombie Objects来实现定位，通过将释放的对象标记为zombie对象，然后给该僵尸对象发送消息，发生`crash`并且输出相关的调用信息。这套机制同时定位了发生`crash`的类对象以及有相对清晰的调用栈。但是相比于填充内存的方式，覆盖范围较小。

     

5. #### 野指针线上处理方案

   1. - ##### 内存重新填充方式 Malloc Scribble

        1. - ###### 1）原理：官方文档对Malloc Scribble的定义

             > If set, fill memory that has been allocated with 0xaa bytes.
             > This increases the likelihood that a program making assumptions about the contents of freshly allocated memory will fail.
             > Also if set, fill memory that has been deallocated with 0x55 bytes.
             > This increases the likelihood that a program will fail due to accessing memory that is no longer allocated.
             > Note that due to the way in which freed memory is managed internally, the 0x55 pattern may not appear in some parts of a deallocated memory block.
             >
             > 如果设置，则用0xaa字节填充已分配的内存。
             > 这增加了对新分配内存内容进行假设的程序失败的可能性。
             > 同样，如果设置了，则用0x55字节填充已释放的内存。
             > 这增加了程序由于访问不再分配的内存而失败的可能性。
             > 注意，由于内部管理释放内存的方式，0x55模式可能不会出现在已释放内存块的某些部分。

             ​	==总结：开启了malloc scribble后，申请内存 alloc 时在内存上填`0xAA`，释放内存 dealloc 在内存上填 `0x55`。==

             ​	该方案针对的2种情况而设定：①初始化未完成或者未开始，就被访问了	②对象被释放后，又被访问了

             ###### 2）如何实现

             ​		按照官方对 Malloc Scribble的思路，结合遇到的类似问题点，可以在对象被释放的时间点做处理，继而引出对象释放的流程：

             ​		**delloc( )** ---> **rootDelloc( )** ---> **fastPath( )判定** ---> **YES** ---> **free( )**

             ​																↓

             ​																**NO**		

             ​																↓			

        ​																执行 **object_dispose( )** --> **objc_destructInstance( )** ---> **free( )**

        ​					其中**objc_destructInstance**定义：Destroys an instance without freeing memory.

        ​					只会销毁实例属性，解除相应的引用关系，并不会释放对应内存。

        ​					综上流程，可以在释放的几个节点做处理，也可以在最终的free函数处理。

        ​					**2.1）处理结点一 free( ) 函数**

        ​						①使用fishhook库，实现对free( )函数的hook操作

        ​						②在hook函数中，对已经释放的内存，进行填充0x55的操作，另外需要保护该内存区域不被重新覆盖

        ​						③因为保存0x55内存会导致一定的内存压力，需要设置最大内存阀值，并且设置内存告警清理逻辑

        ​						④Crash出现时，为了获得较为全面的错误信息，可以通过NSProxy子类实现，重写消息转发的三个方法

        ​						⑤NSProxy只能做OC对象的代理，所以需要在hook函数中增加对象类型的判断

        ​					**2.2）处理节点二 delloc( )函数**

        ​						实现步骤与free类似，不同点就是侵入点的不同，在侵入点前加入了逻辑判断，以减少后续操作

        

      - ##### Zombie Objects 僵尸对象解决方式

        ###### 1）僵尸对象定义：

        > Once an Objective-C or Swift object no longer has any strong references to it, the object is deallocated. Attempting to further send messages to the object as if it were still a valid object is a “use after free” issue, with the deallocated object still receiving messages called a zombie object.
        >
        > 关键点在于给僵尸对象发消息，可以响应，然后发生崩溃

        ###### 2）如何实现：

        ​	①：method swizzling替换NSObject的allocWithZone方法，在新的方法中判断该类型对象是否需要加入野指针防护，如果需要，则通过objc_setAssociatedObject为该对象设置flag标记，被标记的对象后续会进入zombie流程

        > 该阶段的过滤流程，可以着重开发者维护的类，因为系统类的实例发生野指针的概率较小

        ​	②：method swizzling替换NSObject的dealloc方法，对flag标记的对象实例调用objc_destructInstance，释放该实例引用的相关属性，然后将实例的isa修改为自定义的ZombieObject。通过objc_setAssociatedObject 保存将原始类名保存在该实例中

        ​	③：在ZombieObject 通过消息转发机制forwardingTargetForSelector处理所有拦截的方法，根据selector动态添加能够处理方法的响应者的XXObject 实例，然后通过objc_getAssociatedObject 获取之前保存该实例对应的原始类名，统计错误数据

        > ZombieObject的处理和unrecognized selector crash 无响应的处理是一样，主要的目的就是拦截所有传给ZombieObject的函数，用一个返回为空的函数来替换，从而达到程序不崩溃的目的 <线上防Crash措施>

        ​	④：程序退到后台或者达到未释放实例的上限时或内存达到上限时，调用原有dealloc方法释放所有被zombie化的实例

        > Tips：僵尸对象的解决野指针，通过动态方式插入一个空实现的方法来防止Crash，但是交互层需要作额外处理，负责业务可能处于异常状态。另外内存方面因为生成了新的僵尸对象，释放内存需作考虑。野指针的zombie保护机制只能在其实例对象仍然缓存在zombie的缓存机制时才有效，若在实例真正释放之后，再调用野指针还是会出现Crash。

   

6. #### OOM

   ##### 名词释义：

   - OOM：Out of Memory 的缩写，指的是 App 占用的内存达到了 iOS 系统对单个 App 占用内存上限后，而被系统强杀掉的现象，而造成这一现象的机制是Jetsam。
   - Jetsam：操作系统为了控制内存资源过度使用而采用的一种资源管控机制。

   #### 产生原因：

   ​	由于系统为了控制内存资源的过度使用，在监控到内存压力的时，会发送通知，内存有压力的 App 就会去执行对应的代理，也就是熟悉的 didReceiveMemoryWarning 代理，如果没有处理，当达到峰值点，启动Jetsam机制杀死对应app进程。

   > 系统在强杀 App 前，会先做优先级判断。那么，这个优先级判断的依据是什么呢？iOS 系统内核里有一个数组，专门用于维护线程的优先级。这个优先级规定就是：内核用线程的优先级是最高的，操作系统的优先级其次，App 的优先级排在最后。并且，前台 App 程序的优先级是高于后台运行 App 的；线程使用优先级时，CPU 占用多的线程的优先级会被降低。iOS 系统在因为内存占用原因强杀掉 App 前，至少有 6 秒钟的时间可以用来做优先级判断。同时，JetSamEvent 日志也是在这 6 秒内生成的。

   #### 如何处理：

   - ##### 获取内存阀值

     1) 通过Jetsam日志获取

        ①位置 (设置→隐私→分析)中可以看到Jetsam开头的日志

        ②找到日志中"reason" : "per-process-limit"的部分，per-process-limit表示的是，App 占用的内存超过了系统对单个 App 的内存限制

        ③找到per-process-limit 部分的 rpages，rpages 表示的是 ，App 占用的内存页数量

        ④找到pageSize，pageSize表示的是单个内存页的大小

        ⑤计算出当前 App 的内存限制值：pageSize * rpages / 1024 /1024

        

     2) 通过XNU获取

        ​	在 XNU 中，有专门用于获取内存上限值的函数和宏。我们可以通过 memorystatus_priority_entry 这个结构体，得到进程的优先级和内存限制值。结构体代码如下：

        ```
        typedef struct memorystatus_priority_entry {
          pid_t pid;
          int32_t priority;				// 表示进程的优先级
          uint64_t user_data;
          int32_t limit;					// 进程内存的限制值
          uint32_t state;
        } memorystatus_priority_entry_t;
        ```

        ​	但这种方式，需要获取Root权限，没有越狱的手机获取不到这个内容！

        

     3) 通过内存警告获取

        ①：通过didReceiveMemoryWarning这个函数来获取操作点，即开始获取当前内存情况

        ②：使用系统函数 task_info，获取当前任务信息，如下：

        ```O
        struct mach_task_basic_info info;
        mach_msg_type_number_t size = sizeof(info);
        kern_return_t kl = task_info(mach_task_self(), MACH_TASK_BASIC_INFO, (task_info_t)&info, &size);
        ```

        ​	其中info结构里包含了一个 resident_size 字段，用于表示使用了多少内存。这样，我们就可以获取到发生内存警告时，当前 App 占用了多少内存，即调用：info.resident_size; 

        

   - ##### 获取Crash前内存可用信息

     - 定位内存问题点，需要获取到当前内存中的所有对象、对象所占用的内存大小、谁分配的内存，这样才能精确的定位问题点。

     - 内存分配函数：函数 malloc 和 calloc 等默认使用的是 nano_zone。nano_zone 是 256B 以下小内存的分配，大于 256B 的时候会使用 scalable_zone 来分配。

     - 关注大内存分配函数scalable_zone，而使用 scalable_zone 分配内存的函数都会调用 malloc_logger 函数，因为系统总是需要有一个地方来统计并管理内存的分配情况。malloc_zone_malloc 函数的实现，代码如下：

       ```
       void *malloc_zone_malloc(malloc_zone_t *zone, size_t size){
         MALLOC_TRACE(TRACE_malloc | DBG_FUNC_START, (uintptr_t)zone, size, 0, 0);
         void *ptr;
         if (malloc_check_start && (malloc_check_counter++ >= malloc_check_start)) {
           internal_check();
         }
         if (size > MALLOC_ABSOLUTE_MAX_SIZE) {
           return NULL;
         }
         ptr = zone->malloc(zone, size);
         // 在 zone 分配完内存后就开始使用 malloc_logger 进行进行记录
         if (malloc_logger) {
           malloc_logger(MALLOC_LOG_TYPE_ALLOCATE | MALLOC_LOG_TYPE_HAS_ZONE, (uintptr_t)zone, (uintptr_t)size, 0, (uintptr_t)ptr, 0);
         }
         MALLOC_TRACE(TRACE_malloc | DBG_FUNC_END, (uintptr_t)zone, size, (uintptr_t)ptr, 0);
         return ptr;
       }
       ```

     - 然后通过hook的方式，hook掉这个函数，加上其他的尽可能多的记录，生成本地日志，在合适的时机上传。

​	

