[TOC]

## ThreadLocal简单使用

ThreadLocal在JDK1.2之后引入，用于实现线程间的数据隔离。ThreadLocal可以在线程内设置独立的数据副本，线程可以通过调用ThreadLocal实例的get()方法获取到数据。并且这个数据只能由本线程访问，其他线程无法获取。

多说无益，直接看个小Demo吧~

```java
    private ThreadLocal<String> threadLocal=new ThreadLocal();
    private ThreadLocal<Integer>threadLocal2=new ThreadLocal<>();

    @Test
    public void testLocal() throws InterruptedException {
        Thread t1=new Thread(new Runnable() {
            @Override
            public void run() {
                threadLocal.set("thread 1");
                threadLocal2.set(1);
                try {
                    Thread.sleep(100); //为了等另一个线程也执行完set，说明不是覆盖
                } catch (InterruptedException e) {
                    e.printStackTrace();
                }
                System.out.println("i am t1:"+threadLocal.get());
                System.out.println("i am t1:"+threadLocal2.get());
            }
        });
        Thread t2=new Thread(new Runnable() {
            @Override
            public void run() {
                threadLocal.set("thread 2");
                threadLocal2.set(2);
                try {
                    Thread.sleep(100);
                } catch (InterruptedException e) {
                    e.printStackTrace();
                }
                System.out.println("i am t2:"+threadLocal.get());
                System.out.println("i am t2:"+threadLocal2.get());
            }
        });
        t1.start();  //i am t1:thread 1   //i am t1:1
        t2.start();  //i am t2:thread 2   //i am t2:2
        Thread.sleep(100);
    }
```

可以看到效果：在每个线程内设置数据副本，且线程可以通过同一个ThreadLocal实例获取到不同的数据副本。是不是觉得很神奇？那接下来就看一下是如何实现的。

## ThreadLocal实现原理

先从ThreadLocal的set()方法看起，看看它是如何实现的

```java
    public void set(T value) {
        Thread t = Thread.currentThread();  //获取当前线程实例
        ThreadLocalMap map = getMap(t);  //获取到了一个map
        if (map != null) {//如果map不为空，则把ThreadLocal实例作为key，设定的值作为value，加入到map中
            map.set(this, value);
        } else {
            createMap(t, value);//如果map还没被初始化，那就先初始化。这里属于懒加载
        }
    }
```

可以看到getMap()方法传入了当前线程实例作为参数，那是不是也就是说，每个Thread都有一个对应的map？是这样的，看下去就知道了

getMap()方法：

```java
    ThreadLocalMap getMap(Thread t) {
        return t.threadLocals;
    }
```

发现实现很简单，直接返回了传入线程里面的一个属性**threadLocals**。那theadLocals是什么呢？那就要去到Thread类里面看了

```java
/* ThreadLocal values pertaining to this thread. This map is maintained
 * by the ThreadLocal class. */
ThreadLocal.ThreadLocalMap threadLocals = null;
```

可以看到Thread类的threadLocals对象，其实就是一个Map，这个map以ThreadLocal实例为key，以泛型T作为value储存在每一个Thread中。这样就清晰了。线程每调用一次ThreadLocal的set(T)方法，都会将当前ThreadLocal实例-T作为一个key-value值存在自己的threadLocals这个map中，所以调用一次get()方法，就是从map中取出value而已。

所以说为什么线程能通过同一个ThreadLocal获取到自己线程里面独立的数据副本？实际上只是将ThreadLocal实例作为key储存了起来而已。

ps：所以如果使用线程池搭配ThreadLocal进行使用的话，使用完之后记得对线程内的threadLocals属性进行清理，因为线程池中的线程是会被复用的，如果该属性没有被清理，后续很可能会影响业务逻辑，可以重写线程池中的afterExxcute方法：

```java
protected void afterExecute(Runnable r, Throwable t) { 
    // you need to set this field via reflection.
    Thread.currentThread().threadLocals = null;
}
```



## ThreadLocal的内存泄露问题

内存泄漏问题：由于疏忽或错误导致程序未能释放已经无用的内存空间。这就是内存泄漏问题。

之前说过了ThreadLocalMap是以ThreadLocal实例为key，T为value。其实这种说法是错误的！作为key值的并不是ThreadLocal实例，而是该实例的一个弱引用WeakReference

```java
static class Entry extends WeakReference<ThreadLocal<?>> {
    /** The value associated with this ThreadLocal. */
    Object value;

    Entry(ThreadLocal<?> k, Object v) {
        super(k);
        value = v;
    }
}
```

其实使用弱引用，就是能帮助我们解决内存泄漏的问题。如果key是ThreadLocal的强引用会怎么样呢？我们一步步来分析：

当我们认为该ThreadLocal实例在线程中已经不再会被使用到了，那么一般来说我们会直接删除该强引用。但是该对象会被GC回收吗？很明显不会，因为在map的key中还保存着该实例的强引用，地球人都知道，只要对象的强引用还存在，GC是不会对该对象进行回收的，所以也就造成了"程序未能释放已经无用的内存空间"这个问题，也就是内存泄漏。

那如果使用弱引用呢？当ThreadLocal实例，也就是强引用被我们删除之后，触发GC的时候也会回收弱引用，所以说map中的key就会被回收，变为null。而ThreadLocalMap也会清理key为null的Entry值，那么这就是一次正确地回收。

当然了，弱引用能被正确回收的前提是，我们要手动地清除强引用。所以我们在明确ThreadLocal在当前线程中不再被使用的时候，就要调用其remove()方法进行强引用的删除，这样就能防止内存泄漏。



**面试题：谈一下ThreadLocal的内存泄漏？**

ThreadLocal是在JDK1.2之后引入的一个类，存在于java.lang包下。它的作用就是实现线程间数据隔离，它可以在每个线程内设置一个单独的数据副本，并且只有当前线程可以访问到。也就是说，不同的线程可以通过同一个ThreadLocal实例访问到不同的变量副本。

而内存泄漏问题是指，由于开发者的疏忽或错误，导致程序未能释放已经无用的内存空间。

ThreadLocal如果没有被正确使用也会出现内存泄漏的问题，原因要从ThreadLocal的实现原理开始说：

ThreadLocal之所以能实现在不同线程内设置不同数据副本这一功能，是依靠Thread对象内的一个叫threadLocals的属性来实现的，该属性其实是ThreadLocalMap类的一个实例，顾名思义，这是一个map类，并且该map的数据对以ThreadLocal实例的弱引用作为key值，以泛型T作为value值。也就是说，每个线程内部都维护着一个map。

那ThreadLocal实现的原理，其实就是调用set()方法的时候，往当前线程的map里添加了一个数据对，这个数据对以ThreadLocal实例的弱引用作为key，以传入的泛型T作为value储存在当前线程的map中，而当调用get()的时候，就是从map中获取对应的value值。其实原理还是很简单的。

而刚才一直强调是以ThreadLocal实例的弱引用作为key，因为这是ThreadLocal中帮助我们解决内存泄漏问题的一个方法。如果使用强引用作为key，那么当开发者认为ThreadLocal实例在当前线程中不会再使用，然后删除该强引用的时候，map中对应的key-value对不会被删除。因为key中还存在着该ThreadLocal实例的强引用，而只要对象的强引用还存在，GC就不会回收该对象。所以这就会造成"没有释放已经无用的内存空间"这个问题，也就是内存泄漏。

而这里使用弱引用就能避免这个问题，当开发者决定删除ThreadLocal强引用的时候，下一次GC也会将map中弱引用也回收掉，然后map也会清理掉key值为null的数据对，那么这就完成了一次正确的回收操作。

但弱引用回收的前提是：开发者要手动删除强引用。所以在使用ThreadLocal的时候，如果确定线程已经不会再使用该ThreadLocal，就要调用remove()方法进行手动清除，这样就可以避免内存泄漏。

