# Java常用锁
![java主流锁](https://github.com/Adams112/jdk-reading/blob/master/images/20190616200439354_.png 'java主流锁')

整理给自己看的，讲不清楚，记下重点。  
## volatile
volatile并不是锁，在这里一并写上。  
volatile是java语言提供的一种同步方法。volatile可以保证**有序性**，即禁止JVM对指令进行重排序，可以保证**可见性**，即一个线程可以看到另一个线程对变量做的修改，但不能保证**原子性**。  
  
**volatile如何保证可见性**----缓存一致性协议MESI  
缓存一致性协议给缓存行（通常为64字节）定义了个状态：独占（exclusive）、共享（share）、修改（modified）、失效（invalid），用来描述该缓存行是否被多处理器共享、是否修改。所以缓存一致性协议也称MESI协议。

独占（exclusive）：仅当前处理器拥有该缓存行，并且没有修改过，是最新的值。  
共享（share）：有多个处理器拥有该缓存行，每个处理器都没有修改过缓存，是最新的值。  
修改（modified）：仅当前处理器拥有该缓存行，并且缓存行被修改过了，一定时间内会写回主存，会写成功状态会变为S。  
失效（invalid）：缓存行被其他处理器修改过，该值不是最新的值，需要读取主存上最新的值。  

一个处于M状态的缓存行，必须时刻监听所有试图读取该缓存行对应的主存地址的操作，如果监听到，则必须在此操作执行前把其缓存行中的数据写回CPU。  
一个处于S状态的缓存行，必须时刻监听使该缓存行无效或者独享该缓存行的请求，如果监听到，则必须把其缓存行状态设置为I。  
一个处于E状态的缓存行，必须时刻监听其他试图读取该缓存行对应的主存地址的操作，如果监听到，则必须把其缓存行状态设置为S。  
当CPU需要读取数据时，如果其缓存行的状态是I的，则需要从内存中读取，并把自己状态变成S，如果不是I，则可以直接读取缓存中的值，但在此之前，必须要等待其他CPU的监听结果，如其他CPU也有该数据的缓存且状态是M，则需要等待其把缓存更新到内存之后，再读取。  
当CPU需要写数据时，只有在其缓存行是M或者E的时候才能执行，否则需要发出特殊的RFO指令(Read Or Ownership，这是一种总线事务)，通知其他CPU置缓存无效(I)，这种情况下会性能开销是相对较大的。在写入完成后，修改其缓存状态为M。  

**volatile如何保证有序性**----内存屏障  
内存屏障分为4种：
LoadLoad屏障：（指令Load1; LoadLoad; Load2），在Load2及后续读取操作要读取的数据被访问前，保证Load1要读取的数据被读取完毕。  
LoadStore屏障：（指令Load1; LoadStore; Store2），在Store2及后续写入操作被刷出前，保证Load1要读取的数据被读取完毕。  
StoreStore屏障：（指令Store1; StoreStore; Store2），在Store2及后续写入操作执行前，保证Store1的写入操作对其它处理器可见。  
StoreLoad屏障：（指令Store1; StoreLoad; Load2），在Load2及后续所有读取操作执行前，保证Store1的写入对所有处理器可见。它的开销是四种屏障中最大的。在大多数处理器的实现中，这个屏障是个万能屏障，兼具其它三种内存屏障的功能  

这一篇比较详细(https://www.jianshu.com/p/6745203ae1fe)

## synchronized
**细节没看明白**  
锁升级：无锁-偏向锁-轻量级锁-重量级锁  
锁优化：适应性自旋锁，锁粗化，锁消除    

同步代码块，锁住括号中的对象  
同步方法，锁住调用对象
静态同步方法，锁住.class对象  

**底层是如何加锁和释放锁的？**--对象头：锁标志，是否可偏向，（线程id--偏向锁，栈中锁指针--轻量级锁，互斥量指针--重量级锁）  
1. 锁标志01(无锁或偏向锁)  
好像没有地方讲cas操作，比较的值是什么？我的理解是比较的永远是无锁状态的markword，也就是每个线程都是从无锁状态得到锁的。  
如果是无锁状态。cas替换线程id，替换成功获得锁  
如果是偏向锁状态。如果线程id与自身id相等，直接获得锁。如果不相等，查看原线程状态，如果原线程已经退出或者不拥有该锁，则将该对象置为无锁状态，可以重新偏向其他线程。如果继续持有该锁，则锁升级为轻量级锁。  
偏向锁不会主动释放锁。  

2. 锁标志00（轻量级锁）  
通过cas获取锁，不成功则自旋。不成功超过一定次数则锁会升级。  
释放，cas释放锁。  

3. 锁标志10(重量级锁)  
如果别的线程拥有锁，直接挂起，等待唤醒

**锁是如何升级的**  
1. 偏向锁-轻量级锁  
当一个线程持有锁，另一个线程还需要锁时，锁会升级  

2. 轻量级锁-重量级锁
自旋超过一定次数。

![synchronized加锁过程](https://github.com/Adams112/jdk-reading/blob/master/images/20180908110545722.png 'synchronized加锁过程')

## AQS(AbstractQueuedSynchronizer)
AQS是一个队列同步器，基于该类可以实现公平/非公平锁、读写锁、互斥锁等，只需要实现锁的获取和释放逻辑。大部分讲的很详细，这里只记下几个问题和重点。  
源码的分析：(https://www.cnblogs.com/waterystone/p/4920797.html)  

**重点1 对于独享锁，如何获取和释放锁**  
获取锁：
1. 调用自定义同步器的tryAcquire()尝试直接去获取资源，如果成功则直接返回；
2. 没成功，则addWaiter()将该线程加入等待队列的尾部，并标记为独占模式；
3. acquireQueued()使线程在等待队列中休息，有机会时（轮到自己，会被unpark()）会去尝试获取资源。获取到资源后才返回。如果在整个等待过程中被中断过，则返回true，否则返回false。
4. 如果线程在等待过程中被中断过，它是不响应的。只是获取资源后才再进行自我中断selfInterrupt()，将中断补上。
![独享锁获取锁过程](https://images2015.cnblogs.com/blog/721070/201511/721070-20151102145743461-623794326.png '独享锁获取锁过程')

释放锁：
1. tryRelease()释放资源；
2. 如果释放后资源为0，从队列中唤醒下一个等待的线程。

**重点2 对于共享锁，如何获取和释放锁**  
与获取锁类似。差别在于共享锁获取成功时，可能会通知其他线程获取锁(因为是共享的)。什么时候会通知？插队获取到共享锁的，不会通知。在队列中被其他线程唤醒的，会通知后继线程。

**问题1 队列的头节点表示获取到锁的线程吗？**  
主要是aquire()源码  
不一定是。  
第一个获取到锁的线程会直接执行，队列为空。以后线程再获取锁，才加入到队列。当第二个线程获取锁会创建队列，这时第一个节点是空节点(但不是null)，不表示任何一个线程。 
当某个线程释放锁时，如果唤醒了队列中的线程，队列中的线程会将自己设置为头结点。这时头结点表示拥有锁的线程。  
当队列中只有一个线程在等待，这个线程被唤醒后，会将自己设置为头结点，这时队列中只有一个节点，这个线程执行释放锁之后不会唤醒其他线程，也不会清除自身，头结点表示某个过去拥有锁的线程。当下一个线程获取锁时，不会操作队列，再下一个线程获取锁时，会加到队列第二个位置。  
所以队列中第二个节点表示下一个被唤醒的线程，但如果这个线程放弃了锁(状态为CANCELED)，会被忽略继续查找。  

**问题2 AQS是公平的吗**  
AQS队列是FIFO的，保证了一定的公平性。在共享锁中，如果当前剩余锁比当前需要唤醒线程的需求要小时，不会唤醒后面的线程，尽管剩余的锁可能能够满足后面线程的需要。但AQS不是公平锁。  
问题在于锁的释放和获取。获取时，先查看能不能获取到锁，不能的话再入队列。释放时，先将锁释放，再从队列中去唤醒下一个线程。如果在锁释放之后，唤醒下一个线程之前，有一个线程获取锁，是会获取成功的。也就是，AQS允许插队。  

**问题3 自己想要实现一种锁的逻辑，要怎么做？**  
ReentransLock和ReentrantReadWriteLock都是通过AQS实现的。要实现一种锁的逻辑，需要实现tryAcquire和tryRelease两个方法。 

## ReentrantLock
实现了公平锁和非公平锁，需要实现tryAcquire和tryRelease。核心内容就是要支持重入，和是否允许插队，释放锁的逻辑都一样。  
**获取锁**  
公平锁：实际上就是不允许tryAcquire时插队。在tryAcquire时，会检查队列是否为空，为空的话，才会直接获取，否则会入队。  
如何支持重入？就是获取锁的时候，判断下当前线程和拥有锁的线程是不是同一个。

## ReentrantReadWriteLock
state高16位表示读锁，低16位表示写锁。可以是公平的也可以是非公平的，内部的sync继承AQS。
公平读写锁和非公平读写锁差别还是在于是否允许插队，队列中的线程能够保证获取锁的顺序。

### 公平读写锁
就是不插队  
**写锁的获取**  
首先是否有线程在占用锁。如果有线程占用锁，在检查自身是否占用写锁，如果占用写锁直接获取，否则获取失败。  
如果没有线程占用锁，判断队列中是否有等待，有的话入队列，没有线程等待则抢占。  
**读锁的获取**  
这里面有点内容，大框架类似  
首先看是否有线程占用写锁，如果占用写锁的不是自己的话，则获取锁失败，会如队列。  
如果没有线程占用写锁，或者或者占用写锁的是自己，会看队列是否为空，为空则竞争锁，竞争成功则获取到锁。  
如果没竞争到锁，或者队列不为空就要入队列吗？ 还需要进一步判断读锁是不是重入的读锁。是重入的读锁，也要获得锁成功。  

释放锁没啥可说
### 非公平读写锁
就是获取的时候可以插队，其他跟公平锁一样。有一点：如果队列中第一个等待的是写锁的话，获取读锁时也会进入队列，保证写锁不会过度饥饿。    

## CAS
CPU硬件上依赖于MESI协议实现，是本地调用方法。封装的java代码不看了。  
