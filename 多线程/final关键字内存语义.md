[TOC]

对于final修饰的变量，其实有两个与指重排相关的规则

- **如果final变量在构造函数中赋值，那么就禁止这个赋值操作与构造函数的返回操作进行指令重排序**
- **如果一个对象中包含final变量，并且先访问这个对象，然后访问这个final变量，这两个操作不允许发生指令重排序**

我们所书写的代码顺序和JVM执行的指令顺序不一定是相同的，JVM会在不影响执行正确性的前提下对指令进行重排序，以达到提高执行效率的目的。当然，这个正确性指的是单线程下的正确性，如果是多线程的话，JVM是无法判断的。

## 写final

如果final变量在构造函数中进行赋值，那么在编译的时候，该赋值操作后面就会生成一个StoreStore内存屏障，用来保证构造函数返回之前，final变量必须赋值完成，以此来保证数据正确性。

而对于普通变量是没有这个机制的，所以，如果普通内部变量在对象构造函数中赋值，普通变量的赋值操作可能会发生逃逸。一个线程构造了这个对象，另一个线程访问这个对象，如果访问成功说明这个对象的构造函数已经执行完了，并且按理来说普通变量应该也已经赋值完成了，但是因为指令重排，普通变量的赋值操作可能逃逸出构造函数外了，导致普通变量的赋值有可能还未完成，此时访问这个普通变量就会出现问题，而final变量不会有这个问题：

![img](https://img-blog.csdn.net/20180520214123252?watermark/2/text/aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L2RhbTQ1NDQ1MDg3Mg==/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70)

## 读final

如果对象中包含final域，初次访问这个对象，和初次访问这个对象的final域，这两个操作不允许重排序。这是因为，如果访问对象，说明对象的构造方法已经执行完毕了，也说明final变量也进行赋值了。那么不会有问题；如果发生指令重排，先去访问final变量，此时就会读到还未赋值的final变量，就会出现问题。

具体的实现呢就是在编译的时候，在final变量读操作的前面加上一个LoadLoad内存屏障，来禁止指令重排
