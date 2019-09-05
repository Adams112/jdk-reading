# Java常用锁
![java主流锁](https://github.com/Adams112/jdk-reading/blob/master/images/20190616200439354_.png 'java主流锁')

## volatile
volatile并不是锁，在这里一并写上。
volatile是java语言提供的一种同步方法。volatile可以保证**有序性**，即禁止JVM对指令进行重排序，可以保证**可见性**，即一个线程可以看到另一个线程对变量做的修改，但不能保证原子性 。




## synchronized

## ReentrantLock

## synchronized和ReentrantLock比较

## synchronized深度理解

## ReentrantReadWriteLock

## CAS
