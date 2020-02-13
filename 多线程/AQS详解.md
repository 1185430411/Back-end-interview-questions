@[toc]
AQS是AbstractQueuedSynchronizer的简称。AQS提供了一种能实现阻塞锁和以来FIFO等待队列的同步器框架。很多同步容器都是基于其实现的。例如ReentranLock和CountDownLatch，实际上都是基于AQS实现的。

AQS核心思想就是维护一个volatile的int型变量state，用来记录状态；还有一个等待队列，用来存储陷入阻塞态的线程。

## AQS基础属性

### state状态

AQS维护了一个volatile int类型的变量，可以保证操作的可见性。与state相关的方法其实只有三个：

```java
/**
     * The synchronization state.
     */
    private volatile int state;

    /**
     * Returns the current value of synchronization state.
     * This operation has memory semantics of a {@code volatile} read.
     * @return current state value
     */
    protected final int getState() {
        return state;
    }

    /**
     * Sets the value of synchronization state.
     * This operation has memory semantics of a {@code volatile} write.
     * @param newState the new state value
     */
    protected final void setState(int newState) {
        state = newState;
    }

    /**
     * Atomically sets synchronization state to the given updated
     * value if the current state value equals the expected value.
     * This operation has memory semantics of a {@code volatile} read
     * and write.
     *
     * @param expect the expected value
     * @param update the new value
     * @return {@code true} if successful. False return indicates that the actual
     *         value was not equal to the expected value.
     */
	//CAS设置state
    protected final boolean compareAndSetState(int expect, int update) {
        return STATE.compareAndSet(this, expect, update);
    }
```

### 资源共享方式

AQS中定义了两个资源共享方式：

- Exclusive:独占式，每次只有一个线程能运行，如ReentrantLock
- Share：共享式，每次可以又多个线程执行，如Semaphore/CountDownLatch

举例说明。如ReentrantLock中，state会被设置为0，此时是属于未锁定状态。当线程A执行lock()的时候，实际上是调用tryAcquire()独占该锁，并将state+1,此时如果有其他的线程想要执行tryAcquire()函数，就会被加入到阻塞队列中等待被唤醒了。

再如CountDownLatch。如果构建CountDownLatch实例时传入的对象是N，那么state也会被设置为N。在执行countDown()方法的时候，会执行tryReleaseShare()方法，让state-1.若其他线程B执行了await()方法，state却>0，那么线程B就会进入阻塞队列之中。

## 源码详解

### 队列中节点状态waitStatus

在AQS中，每一个线程在进入队列之前，都会被封装成一个Node结点。Node结点除了线程，还包含了五种状态：

- 0:结点刚入队时的默认值。
- CANCELLED（1）：结点已经取消调度。当发生超时或者被中断后，结点会变成该状态，该状态的转变是不可逆的。
- SIGNAL（-1）：表示后继结点正在等待当前结点的唤醒。在一个结点入队之后，它必须要确保把前驱结点状态改变为SIGNAL之后，它才会安心地被LockSupport调用park()。否则它会面临不被唤醒的风险。
- PROPAGATE(-3)：主要使用在共享模式下。当前结点不仅会唤醒后继结点，还会唤醒后继的后继结点。

### 方法详解

#### 1.acquire(int)

该方法一般用于独占模式。也正是lock()方法的语义。

```java
public final void acquire(int arg) {
    if (!tryAcquire(arg) &&  //尝试获取资源，若成功则直接返回(此方法需要内部类Sync中重写，如果需要使用的话)
       acquireQueued(addWaiter(Node.EXCLUSIVE), arg)) //addWaiter使线程加入队列 acquireQueued使线程阻塞
       selfInterrupt();
}
```

##### 1.1tryAcquire(int)

```java
protected boolean tryAcquire(int arg) {
    throw new UnsupportedOperationException();
}
```

该方法直接抛出异常，是因为需要子类进行重写。而之所以不声明为抽象方法，是为了不强制让子类实现，如果有需要的话才实现。

##### 1.2 addWaiter(Node)

该方法就是把线程添加到等待队列的末尾。

```java
    private Node addWaiter(Node mode) {
        Node node = new Node(mode);
        //自旋
        for (;;) {
            Node oldTail = tail;
            //若尾节点不为空
            if (oldTail != null) {
                //将尾节点设置为自己的前驱结点
                node.setPrevRelaxed(oldTail);
                //CAS将自己设置为新的尾结点
                if (compareAndSetTail(oldTail, node)) {
                    oldTail.next = node;
                    return node;
                }
                //若尾结点为空，则创建该一个新的队列
            } else {
                initializeSyncQueue();
            }
        }
    }

```

##### 1.3.acquireQueued(Node, int)

该方法主要就是将加入队列的线程阻塞。

```java
    final boolean acquireQueued(final Node node, int arg) {
        boolean interrupted = false;
        try {
            //自旋，直到该结点获取到资源
            for (;;) {
                //获取前驱结点
                final Node p = node.predecessor();
                //若前驱结点是头结点(说明下一个就要轮到自己了)，且获取资源成功
                if (p == head && tryAcquire(arg)) {
                    //将自己设置为头结点(所以意思就是说，头结点就是正在使用资源的结点)
                    setHead(node);
                    //有助于GC
                    p.next = null; // help GC
                    return interrupted;
                }
                //若前驱结点不是头结点，说明要真正地进行阻塞了
                if (shouldParkAfterFailedAcquire(p, node))
                    interrupted |= parkAndCheckInterrupt();
            }
        } catch (Throwable t) {
            cancelAcquire(node);
            if (interrupted)
                selfInterrupt();
            throw t;
        }
    }

```

##### 1.4 shouldParkAfterFailedAcquire(Node, Node)

该方法就是前面所说的：要保证前驱结点的状态是SIGNAL，否则会没有人来唤醒。

```java
   private static boolean shouldParkAfterFailedAcquire(Node pred, Node node) {
       //获取前驱结点
        int ws = pred.waitStatus;
       //若已经被设置为SIGNAL了，直接返回true
        if (ws == Node.SIGNAL)
            /*
             * This node has already set status asking a release
             * to signal it, so it can safely park.
             */
            return true;
       //若前驱结点已经取消了，说明要继续往前寻找下一个可用的
        if (ws > 0) {
            /*
             * Predecessor was cancelled. Skip over predecessors and
             * indicate retry.
             */
            do {
                node.prev = pred = pred.prev;
            } while (pred.waitStatus > 0);
            //找到了可用的结点，设置next指针
            pred.next = node;
        } else {
            /*
             * waitStatus must be 0 or PROPAGATE.  Indicate that we
             * need a signal, but don't park yet.  Caller will need to
             * retry to make sure it cannot acquire before parking.
             */
            //CAS
            pred.compareAndSetWaitStatus(ws, Node.SIGNAL);
        }
        return false;
    }

```

##### 1.5parkAndCheckInterrupt()

到了这一步，就是真正的调用LockSupport.pork()方法来使线程进入waiting状态了

```java
    private final boolean parkAndCheckInterrupt() {
        LockSupport.park(this); //使线程进入waiting状态
        return Thread.interrupted(); //返回是否被中断
    }
```

OK！走到这里，acquire相关操作就已经基本走完。其实总结下来也就两句话：

如果能够获取到资源，就直接返回，否则就加入队列，等待资源可用再被唤醒。

**说完了获取或阻塞。那接下里就要讲一下释放并唤醒了**！

#### 2.release()

此方法就是在独占模式下释放资源。若释放之后state==0。说明资源可用，就会到队列中唤醒结点。这也正是unlock()的语义

```java
    public final boolean release(int arg) {
        //如果释放资源成功
        if (tryRelease(arg)) {
            Node h = head;
            //如果头结点不为空且不是新建态
            if (h != null && h.waitStatus != 0)
                //唤醒头结点
                unparkSuccessor(h);
            return true;
        }
        //如果释放资源失败，返回false
        return false;
    }

```

##### 2.1.tryRelease(int)

```java
    protected boolean tryRelease(int arg) {
        throw new UnsupportedOperationException();
    }
```

同样的，该方法还是需要子类去实现。在实现的时候需要注意。如果state变成0了，要返回true，否则返回false。因为AQS中会基于tryRelease(int)的返回值来决定是否要进行线程唤醒。

##### 2.2.unparkSuccessor(Node)

```java
    private void unparkSuccessor(Node node) {
        /*
         * If status is negative (i.e., possibly needing signal) try
         * to clear in anticipation of signalling.  It is OK if this
         * fails or if status is changed by waiting thread.
         */
        int ws = node.waitStatus;
        if (ws < 0)
            node.compareAndSetWaitStatus(ws, 0);

        /*
         * Thread to unpark is held in successor, which is normally
         * just the next node.  But if cancelled or apparently null,
         * traverse backwards from tail to find the actual
         * non-cancelled successor.
         */
        //取该结点的后继结点，后继结点就是真正要唤醒的结点
        Node s = node.next;
        //若s为空或者已经被取消，则要重新遍历队列来寻找
        if (s == null || s.waitStatus > 0) {
            s = null;
            //找到最靠前的一个p，作为s
            for (Node p = tail; p != node && p != null; p = p.prev)
                if (p.waitStatus <= 0)
                    s = p;
        }
        if (s != null)
            //唤醒p
            LockSupport.unpark(s.thread);
    }

```

该方法主要是要用来唤醒线程

OK！release()相关的方法也已经说完了，其实也很简单。就是释放资源，如果释放之后state为0，要从队列中唤醒线程。

## 应用AQS来模拟实现一个ReentrantLock

```java
package com.jay.test;

import java.util.concurrent.locks.AbstractQueuedSynchronizer;

public class MyReentrantLock {

    private Sync sync=new Sync();

    private static class Sync extends AbstractQueuedSynchronizer{

        @Override
        protected boolean isHeldExclusively() {
            return getState()==1;
        }

        //重写tryAcquire()
        @Override
        protected boolean tryAcquire(int arg) {
            //如果CAS获取锁成功
            if (compareAndSetState(0,1)){
                //将持有线程设为自己
                setExclusiveOwnerThread(Thread.currentThread());
                return true;
            }
            //否则就返回false
            return false;
        }

        //重写tryRelease()
        @Override
        protected boolean tryRelease(int arg) {
            //如果没有上锁，无法解锁
            if(getState()==0)
                throw new IllegalMonitorStateException();
            //如果不是当前线程在持有锁，也报错
            if(getExclusiveOwnerThread()!=Thread.currentThread())
                throw new  IllegalThreadStateException();
            //初始化
            setExclusiveOwnerThread(null);
            setState(0);

            return true;
        }
    }


    public void lock(){
        sync.acquire(1);
    }

    public void unlock(){
        sync.release(1);
    }

    public void tryLock(){
        //需要马上返回结果，所以直接调用tryAcquire()，没有"线程进入队列并阻塞"这一操作
        sync.tryAcquire(1);
    }
    public boolean isLocked(){
        return sync.isHeldExclusively();
    }
}

```
