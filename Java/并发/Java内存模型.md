## Java内存模型基础

Java是通过共享内存来隐式进行线程间通信的，在Java中堆内存是在多个线程间共享的，不同的线程都可以访问堆内存的同一变量。

但是实际操作中，我们尝试使用变量在多个线程之间进行通讯时会遭遇很多问题，比如数据不同步，这些问题产生的主要原因在于缓存优化以及编译指令优化。

* 其中缓存优化又分为CPU高速缓存、写缓冲区和寄存器等

* 编译指令优化又分为编译器优化、指令集并行重排序和内存系统的重排序。

这些优化可以在极大程度上加快系统运行速度，但是在并发的问题上也带了很多问题，对于在并发编程上出现的问题一般的解决方案时在特定的场景禁止这些优化，相应的方法包括总线锁、缓存锁定、禁止指定的重排序

当然上述的这些优化对于普通Java开发者来说是不可见的，在JSR（Java Specification Requests，Java 规范提案）中定义了一些概念来封装底层的实现，而JVM则实现了这些概念的，我们需要了解的Java的一些特定关键字会带来什么样的影响。

## as-if-serial

编译和指令优化的前提是在单线程场景下，程序的执行结果不能发生变化。

这个语义通过禁止一些有**数据依赖性**的指令冲排序来完成

## happens-before

happens-before主要定义了两个操作的可见性，以及默认的执行序列（重排序之前的顺序），happens-before的概论有如下的特征

* 单线程的操作都是具有happens-before关系的
* 不同线程间的操作默认不具备happens-before关系的
* 可以通过锁、volatile等改变不同线程间操作的happens-before关系
* happens-before具有传递性
* happens-before不保证不会重排序，但是有数据依赖性的重排序会谨慎的进行（不会改变运行结果）

## 顺序一致性内存模型

顺序一致性模型是一种理想化的模型，它要求

* 单线程内的所有操作必须按照程序的顺序执行
* 所有线程的的操作都是原子的且对其他线程可见

