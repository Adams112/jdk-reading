# Java常用锁
![java主流锁](https://github.com/Adams112/jdk-reading/blob/master/images/20190616200439354_.png 'java主流锁')

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

## ReentrantLock

## synchronized和ReentrantLock比较

## synchronized深度理解

## ReentrantReadWriteLock

## CAS
