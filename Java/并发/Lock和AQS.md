## AQS

AQS，全称AbstractQueuedSynchronizer，抽象队列同步器。Java同步包下的几种锁都是基于AQS来实现的，AQS主要提供了如下的功能

* 提供了相应的模版方法以供实现不同类型的锁

* 一个锁等待队列，以及使用CAS实现的现场安全的入队和出队操作
* 一个Condition队列，Condition队列上的每一个节点都是一个队列，等待同一个Condition发生

其中等待队列和 `Condition` 的相关操作也是在模版方法中进行的。AQS是一个抽象类，子类需要实现的方法主要有 `tryAcquire` 、 `tryRelease` 、`tryAcquireShared` 、`tryReleaseShared` 和 `isHeldExclusively` 等五个方法，其中前两个是用于实现普通锁，接下来两个用于实现共享锁，最后一个在`Condition` 中会用到

