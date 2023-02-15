# iOS中的锁

## PART 1 背景

在日常开发过程中，为了使应用变得更加高效，或多或少都会运用多线程的方式来进行**性能优化**。但在多线程的使用过程中，就涉及到**多个线程获取同一资源目标的问题**，如果不对该共有资源进行处理，就可能会出现数据错乱的问题。而锁就是用来解决类似问题，根据不同的情况，选择不同锁的处理方式，会更加高效！

## PART 2 锁的类型
### TYPE 2.1互斥锁：
**当资源被一条线程占用时，其他线程去获取锁的时，会获取不到，此时该线程会进入休眠状态，等锁被释放后该线程会被重新唤醒**
#### 2.1.1 递归锁互斥锁：
**释义：**同一线程在锁被释放前可以再次获取锁，即可以递归调用，**当前后代码相互等待就会死锁**
	
* **NSRecusiveLock**：使用和NSLock类似，不过**NSRecursiveLock是递归互斥锁**。

	> NSRecursiveLock defines a lock that may be acquired multiple times by the same thread without causing a deadlock, a situation where a thread is permanently blocked waiting for itself to relinquish a lock.<br/>
	
	NSRecursiveLock定义了一个在同一线程可以多次获取的锁，而不会发生死锁，在这种情况下线程被阻止等待它自己放弃锁.<br/>
	
	> While the locking thread has one or more locks, all other threads are prevented from accessing the code protected by the lock.<br/>

	当线程被一个或者多个锁锁定，所有其他线程的访问会被阻止。<br/>

	**理解总结**：<br/>①同一线程可以多次获取而不会导致死锁的锁，**注意点是同一线程**。<br/>②同一线程获取虽然不会死锁，但是**多个线程相互等待造成的死锁**还需要注意！

* **@synchronized**：使用较为简单的递归互斥锁，但性能较低，不适用任何条件下。<br/>
	`源码太多，解释不多,不贴了，只介绍主要的`<br/>
	* @synchronized在底层实现了**objc\_sync_enter**和 **objc\_sync_exit**两个方法函数<br/>
	* 如果锁的对象obj不存在时,分别会走**objc\_sync_nil()(enter)**和**不做任何操作(exit)**,这也是@synchronized能作为递归锁的原因<br/>
	* 性能低的原因：其在任意地方均可以使用，只有在底层维护一个全局表的方式可以实现，而底层代码中的SyncList和SyncData结构也印证了这一点，频繁的表读取操作，是性能消耗的大头<br/>
	* 不能使用非OC对象作为加锁条件——id2data中接收参数为id类型<br/>
	* 加锁对象不能为nil，否则加锁无效，不能保证线程安全<br/>
	* 替代解决方案：NSLock
	


#### 2.1.1 非递归互斥锁：
**释义：**线程必须要在锁被释放后才能获取锁，不可递归调用，**强行使用递归就会造成堵塞而非死锁**

* **os\_unfair_lock** : To create a lock, allocate a variable of this type and initialize it to OS\_UNFAIR\_LOCK\_INIT. <br/>
	> Note<br/>This is a replacement for the deprecated OSSpinLock.<br/>
	
	SSpLock的替代方式<br/>
	
	> This function doesn't spin on contention,but instead waits in the kernel to be awoken by an unlock.<br/> 

	该方法不会在争用(获取锁)时候旋转，而是在内核中等待被解锁唤醒<br/>
	
	> Like OSSpinLock, this function does not enforce fairness or lock ordering—for example,an unlocker could potentially reacquire the lock immediately,before an awoken waiter gets an opportunity to attempt to acquire the lock.<br/>
		
	像OSSpinLock一样，这个锁方法并不是等价锁，当唤醒线程正在执行的时候，这个锁可能会被立即重新锁上<br/>
	
	> This may be advantageous for performance reasons, but also makes starvation of waiters a possibility.<br/>
	
	这样有可能会提升性能，但是也有让其他线程资源获取变的更难<br><br/>
**理解总结**：os\_unfair_lock 会使用后续线程休眠等待唤醒，当前线程使用资源结束，释放时，后续进程重新唤醒的过程，有可能因为其他的线程访问获取到了使用权，继而获取失败然后继续休眠。<待考证！！！>

* **pthread_mutex**:C语言定义下的多线程加锁方式，当锁被占用，而其他线程申请锁时，不是使用忙等，而是**阻塞线程并睡眠**。<br/>mutex底层有实现一个阻塞队列，如果当前有其他任务正在执行，则加入到队列中，放弃当前cpu时间片。一旦其他任务执行完，则从队列中取出等待执行的线程对象，恢复上下文重新执行,YYKit中运用大量该锁的方式。

* **NSLock** :基于互斥锁pthroad_mutex封装。<br/>
	> The NSLock class uses POSIX threads to implement its locking behavior.<br/> 
	
	NSLock使用POSIX线程来实现锁的功能<br/>
	
	> When sending an unlock message to an NSLock object, you must be sure that message is sent from the same thread that sent the initial lock message. <br/>
	
	当向NSLock对象发送解锁消息时，必须确保该消息是从发送初始锁定消息的同一个线程发送的<br/>
	
	> Unlocking a lock from a different thread can result in undefined behavior.
	
	如果是从其他线程操作，那么将会有不可预知的结果发生。
	
	> You should not use this class to implement a recursive lock. Calling the lock method twice on the same thread will lock up your thread permanently. Use the NSRecursiveLock class to implement recursive locks instead.<br/>
	
	**在递归中不能使用该锁，在同一个线程中调用lock两次该锁，会使该线程永久锁定。递归中可以通过NSRecursiveLock来实现。**
	
	> Unlocking a lock that is not locked is considered a programmer error and should be fixed in your code. The NSLock class reports such errors by printing an error message to the console when they occur.<br/>
	
	解锁没有锁定的锁，会出现错误！错误信息控制台可见。
	
	**理解总结：**<br/>①同一线程上调用两次lock操作，会永久锁定该线程。<br/>
	②解锁操作的消息发送，必须来自锁定该锁的线程，否则会有未知结果发生
	
* **NSCondition**：条件锁，基于互斥锁pthroad_mutex封装。当需要等待某个条件的时候，也就是条件不满足的时候，就可以使用wait方法来阻塞线程，当条件满足了，使用signal方法发送信号唤醒线程。<br/>
	> Lock the condition object.
	
	锁定条件对象.<br/>
	
	> Test a boolean predicate.  (This predicate is a boolean flag or other variable in your code that indicates whether it is safe to perform the task protected by the condition.).<br/>
	
	条件判断（这个条件是布尔值或者其他定义的变量，条件用于指示执行受条件保护的任务是否安全）.<br/>
	> If the boolean predicate is false, call the condition object’s wait or waitUntilDate: method to block the thread.  Upon returning from these methods, go to step 2 to retest your boolean predicate.  (Continue waiting and retesting the predicate until it is true.).<br/>
	
	如果条件判断是错误，那么线程会被阻止直到条件满足。<br/>
	
	> If the boolean predicate is true, perform the task.<br/>
	
	如果条件判断正确，则继续执行任务<br/>
	
	> Optionally update any predicates (or signal any conditions) affected by your task.
	
	可选地更新受任务影响的任何条件<br/>
	
	> When your task is done, unlock the condition object.<br/>
	
	任务结束时，解锁条件对象
	
	**理解总结：**线程A需要等到其条件a满足才会往下走，否则就会堵塞等待，直至条件满足，一旦获得了锁并执行了代码的关键部分，线程就可以放弃该锁并将关联条件设置为新的条件。条件本身是任意的：可以根据应用程序的需要定义它们。
	
* **NSConditionLock**：条件锁，对NSCondition的封装，带条件判断参数，更加灵活。<br/>
	**理解总结：**<br/>①对NSCondition的封装
	<br/>②相比于NSCondition的手动使线程进入等待而阻塞线程，手动释放锁唤醒线程，NSConditionLock可以通过外部传值，自动根据值判断进行线程的阻塞和唤醒操作。
	
* **Semaphore:** 信号量，GCD中锁的互斥锁另一种形式。通过控制持有计数信号，来控制或者阻塞线程的方式。<br/>
> dispatch\_semaphore\_create()：创建信号量<br/>
> dispatch\_semaphore\_wait()：等待信号量，信号量减1。当 信号量<0 时会阻塞当前线程，根据传入的等待时间决定接下来的操作<br/>
> dispatch\_semaphore\_signal()：释放信号量，信号量加1。当信号量>= 0 会执行wait之后的代码


### TYPE 2.2自旋锁
**线程重复检查锁是否可⽤。线程在这⼀过程中保持持续执⾏， 处于一种忙等待状态。⼀旦获取了锁，线程会⼀直保持该锁，直⾄释放!!!**<br/>

> ① 持续获取，占用CPU资源较高，但效率也高<br/>
> ② 可能会出现死锁，特别在递归调用过程中

* <del>**OsspinLock:** iOS10后被抛弃</del>，官方推出os\_unfair_lock来进行替代<br/>
> **废弃原因:**
	<br>①：OSSpinLock不会记录持有它的线程信息，当发生优先级反转的时候，系统找不到低优先级的线程，导致系统可能无法通过提高优先级解决优先级反转问题
	<br>②：高优先级线程使用自旋锁忙等待的时候一直在占用CPU时间片，导致低优先级线程拿到时间片的概率降低

* **atomic:** 内部最初使用<del>OSSpinLock</del>实现，后用os\_unfair_lock替代
	> 原子性修饰的属性进行了spinlock加锁处理<br>
	> 非原子性的属性除了没加锁，其他逻辑与atomic一般无二<br>
	> atomic保证变量在取值和赋值时的线程安全(setter、getter方法的线程安全),但不能保证其引用的安全
* **pthread\_rwlock_t:** 读写锁，读写锁实际是一种特殊的自旋锁，它把对共享资源的访问者划分成读者和写者，读者只对共享资源进行读访问，写者则需要对共享资源进行写操作。这种锁相对于自旋锁而言，能提高并发性，因为在多处理器系统中，它允许同时有多个读者来访问共享资源，最大可能的读者数为实际的CPU数。<br>
	> 特点1：写者是排他性的，⼀个读写锁同时只能有⼀个写者或多个读者（与CPU数相关），但不能同时既有读者⼜有写者。在读写锁保持期间也是抢占失效的 <br>
	> 特点2：如果读写锁当前没有读者，也没有写者，那么写者可以⽴刻获得读写锁，否则它必须⾃旋在那⾥，直到没有任何写者或读者。如果读写锁没有写者，那么读者可以⽴即获得该读写锁，否则读者必须⾃旋在那⾥，直到写者释放该读写锁


## PART 3 如何选用
### 3.1性能方向
* 优先使用os\_unfair_lock、dispatch\_semaphore、pthread\_mutex性能远远高于其他锁<br>
* @synchronize、NSConditionLock性能较差，因地制宜使用<br>
* NSLock多使用

### 3.2 什么情况使用自旋锁

* 预计线程等待锁的时间很短<br>
* 加锁的代码（临界区）经常被调用，但竞争情况很少发生<br>
* CPU资源不紧张<br>
* 多核处理器<br>

### 3.3 什么情况使用互斥锁

* 预计线程等待锁的时间较长<br>
* 单核处理器（尽量减少CPU的消耗）<br>
* 临界区有IO操作（IO操作比较占用CPU资源）<br>
* 临界区代码复杂或者循环量大<br>
* 临界区竞争非常激烈<br>


## PART 4 结语
### 注意点：
* OSSpinLock不再安全，底层用os\_unfair_lock替代
* atomic只能保证setter、getter时线程安全
* 读写锁更多使用栅栏函数来实现
* @synchronized在底层维护了一个哈希链表进行data的存储，有性能影响
* NSLock、NSRecursiveLock、NSCondition和NSConditionLock底层都是对pthread_mutex的封装
* 普通场景下涉及到线程安全，可以用NSLock
* 循环调用时用NSRecursiveLock
* 循环调用且有线程影响时，注意死锁，如果有死锁问题请使用@synchronized
