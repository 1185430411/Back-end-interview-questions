PV操作就是荷兰语**P**asseren（通过），**V**rijgeven（释放）的简称。对应的就是wait等待，signal释放操作。

P操作就是，将进程从运行态转化为阻塞态，直到它被另一个进程唤醒

V操作就是，将一个处于阻塞态的进程唤醒。

这两个操作是CPU原语，所以从操作系统层面能保证这两个操作是原子操作。

PV操作一般都和信号量所关联，来实现一些互斥或者同步的逻辑。比如说将信号量设为0，此时表示一个互斥资源；将信号量设为N，此时就表示最多可以有N个进程并发访问，多出来的进程要陷入阻塞态

在Java中也有所体现，JUC下的AQS就用了这样的思想；AQS也维护了一个state属性和一个等待队列，state属性就是信号量，等待队列就专门用来存放陷入阻塞状态的线程。JUC下的很多类都是基于AQS实现的，比如ReentrantLock就是一个state值默认为0的AQS，Semaphore就是一个state默认值为N的AQS

![在这里插入图片描述](https://img-blog.csdnimg.cn/20200307170927691.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L1NDVVRKQVk=,size_16,color_FFFFFF,t_70)



可以看到这里就维护了等待队列的头结点和尾节点，还有一个state值，就是信号量
