[TOC]

##  重入锁的定义

重入锁的概念就是：一个线程在持有锁资源的过程中，可以重复多次地申请获得到这个锁资源，而不会陷入阻塞状态。

重入锁一般都维护着一个线程持有者和一个计数器，用来计算重入次数

synchronized和ReentrantLock都是重入锁，但在具体实现上稍微有点区别。

## synchronized

要介绍synchronized的重入锁机制，首先得从JVM的角度开始说。这块可以参考之前的一个博客：

https://blog.csdn.net/SCUTJAY/article/details/104298195

这里再大概简述下：

在Java虚拟机中，对象在内存中分为三个区域，其中有一个区域叫对象头。在对象头中又分为两个区域，其中一个叫Mark Word的区域，其中就储存着对象hashcode，GC标记等信息。

而其中有一个锁记录，上面标注着当前对象锁的状态(偏向锁？轻量锁？重量锁？)并且指向一个ObjectMonitor对象。ObjectMonitor对象是存在于JVM源码中的一个C++类的实例。点进这个C++类来看会发现有很多属性。但与重入锁相关的主要讲解两个：

- _count：计数器，记录锁重入次数。
- _owner：当前持有该锁对象的线程ID

所以看到这里就比较清晰了，那获取synchronized锁的流程就是：

1,线程在ObjectMonitor中进行判断，查看_count属性是否为0，若是，说明锁空闲，直接将 _owner改为本线程的线程ID，然后 _count值+1，进入同步区域执行任务

2.若 _count不是0，则检测当前对象的_owner属性是否指向自己，若不是，进入队列中(_EntryList)进行等待。

**3.若指向自己，说明当前线程已经持有锁，可以重入，则把 _count属性+1，继续执行。**

这就是基本的执行流程了，可以看到第三点就是实现了重入锁的机制。

## ReentrantLock

ReentrantLock是基于AQS实现的。重入锁的逻辑大部分都放在了AQS中实现，这点是和synchronized不太一样的。ReentrantLock类中的一系列方法实际上都是直接或者间接调用它里面的一个内部类叫Sync，这个Sync类是AQS的子类，看源码就会发现ReentrantLock中所有方法的实现基本都是调用了Sync重写的方法或者是AQS的方法。所以还需要到AQS中去看。

AQS中主要维护了三个属性：owner，state(状态值),等待队列。

怎么样？是不是和ObjectMonitor差不多？同样有一个owner，同样有计数器，同样有队列。所以重入锁实现逻辑基本大同小异，只是底层不太一样，synchronized是jvm级别的，ReentrantLock是jdk级别的。
