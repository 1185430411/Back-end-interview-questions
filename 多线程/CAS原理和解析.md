[TOC]

# 概述

在JDK1.5之前。Java都是依靠synchronized关键字来保证同步操作，而synchronized是一个有锁操作，而有锁操作会导致一系列问题：

1.加锁，解锁操作会导致线程上下文切换以及调度延时，开销较大。

2.若一个线程请求锁资源失败，那么该线程会在操作系统层面被挂起，这涉及用户态到核心态的转化，开销较大。

总得来说呢，锁是一种重量级的操作，而且像synchronized，ReentrantLock这样的独占锁属于悲观锁。而另一种更加有效的锁就是乐观锁。乐观锁的核心思想都是基于**CAS（Compare And Swap）**。

# CAS简介

## 什么是CAS？

CAS操作包含三个操作数：内存位置V，期望值A，以及新值B。当且仅当内存位置上的值与期望值相等时，才会将该位置的值更新为新值B，否则，不做任何操作。

思想很简单，但这个思想是整个java.util.concurrent包的基石。

还有一个问题是："判断期望值与原值是否相等，相等则修改"，这不是典型的"基于一个可能失效的判断结果来决定是否执行操作"做法吗？不就是竞价条件吗？

事实上，CAS可并不是一个简单Java算法。它是一条CPU并发原语，原语在操作系统种的执行是不可被中断的，所以，整个CAS操作实际上是一个原子操作，自然也就不会出现竞价条件的情况了。所以使用CAS来进行数据更新，就可以保证更新操作的原子性。AtomicInteger等原子类全部都是基于CAS来进行更新的。

也就是说CAS本身就能保证操作的原子性，再配合上自旋，就可以保证线程安全。原子性保证+线程安全，这就具备了成为"锁"的基本条件了：

```java
    //自旋执行CAS
	do{
       执行CAS;  
     }while(CAS执行失败)
```

## CAS的目的？

利用CPU对CAS的支持来保证原子性，再加上自旋来保证线程安全。这样就可以避免全部使用锁来保证线程安全。因为研究表明，线程持有锁的时间是非常短暂的，也就是说，当前线程虽然此时获取锁失败，但是很可能在不久的将来就能获取成功，这样的话，此时获取锁失败就直接在操作系统层面被挂起是很不划算的，因为这涉及用户态到核心态的切换。所以合理地使用CAS能够减小系统开销。

## CAS存在的问题？

### 1.ABA问题

刚才提到过，如果判断内存里的值和期望值相等，我们就执行更新。但问题是，即使满足这个条件，也不能保证在这段时间里没有其他的线程对这个数据进行更新。因为内存里的值相等，有可能从来没有变化过，也有可能经过一系列变化之后再变回原来的值，这就是ABA问题。在很多情况下，ABA问题不会影响执行正确性，我们并不需要关心。但是也会有例外的情况。

**解决方法**：除了维护最原始的三个值之外，再维护一个版本号字段，在进行数据更新的时候，版本号也要更新。所以在进行比对的时候，除了比对期望值和内存值，也要比对期望版本号和内存里的版本号。JDK1.5可以利用**AtomicStampedReference类**来解决这个问题，其内部除了维护value之外，还维护了一个时间戳stamp。那进行更新的时候，除了更新value，还需要更新这个时间戳。

### 2.循环时间长开销大

CAS一般来说会配合自旋来实现锁功能。但在并发冲突比较严重的场景下，自旋CAS会有大量操作失效，也就是

```java
//自旋执行CAS
do{
   执行CAS;  
 }while(CAS执行失败)
```

这样一个操作。如果自旋次数过多，会造成较大的CPU开销。

这也是为什么在JDK1.6之后对synchronized的优化之中除了加入偏向锁，轻量锁等，还引入了锁膨胀的机制，就是当自旋次数超过设定的阈值时，轻量锁会放弃自旋，膨胀为重量锁，然后线程阻塞。

**解决方法**：JVM支持处理器提供的pause指令，这样的话，效率会有一定的提升，pause指令有两个作用：

- 延迟流水线执行指令,使CPU不会消耗过多的执行资源
- 避免在退出循环的时候因内存顺序冲突（memory order violation）而引起CPU流水线被清空（CPU pipeline flush），从而提高CPU的执行效率。

### 3.不能保证多个共享变量的原子操作

CAS可以保证一个共享变量的原子操作，但如果涉及多个变量，就无法保证操作的原子性。

**解决方法**：

1.使用互斥锁，对相关操作直接加锁，简单粗暴。

2.将多个共享变量存储在一个对象之中，然后对一个对象进行原子操作。从JDK1.5开始就有**AtomicReference**类保证引用对象操作的原子性，只需要把多个共享变量放在一个对象里进行CAS操作即可。

## CAS原理（简述）

讲述CAS执行原理之前，先从熟悉的AtomicInteger类入手，先看其中的getAndIncrement()方法，这是一个CAS操作

```java
    public final int getAndIncrement() {
        return U.getAndAddInt(this, VALUE, 1);
    }
```

会发现其实是调用了U的getAndAddInt方法，那U又是啥？查看该类的开头：

```java
private static final jdk.internal.misc.Unsafe U = jdk.internal.misc.Unsafe.getUnsafe();
```

U原来就是Unsafe这个类的实例，再点进去看，发现其中一大堆都是native方法，如：

```java
	@HotSpotIntrinsicCandidate
    public native void putObject(Object o, long offset, Object x);

    /** @see #getInt(Object, long) */
    @HotSpotIntrinsicCandidate
    public native boolean getBoolean(Object o, long offset);

    /** @see #putInt(Object, long, int) */
    @HotSpotIntrinsicCandidate
    public native void    putBoolean(Object o, long offset, boolean x);
```

native方法也就是本地方法，由jvm本地实现，所以再想点进去看的话就是jvm中的cpp代码了。

我们都知道，java是无法直接和底层系统进行交互的，那前面又说过，CAS是一个CPU原语，那怎么样执行这个原语，那么就是执行native方法，这个还是比较好理解的。

像AQS这样的同步容器框架，里面的更新操作底层基本都是调用Unsafe实例中的一些native方法来实现的。所以说CAS是整个并发包实现的基石
