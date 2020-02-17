CopyOnWriteArrayList是一个线程安全的ArrayList，它的特点是读写分离，并且读操作时不加锁，写操作进行加锁的一种同步容器。

CopyOnWriteArrayList的读操作无需加锁：

```java
public E get(int index) {
    return elementAt(getArray(), index);
}

static <E> E elementAt(Object[] a, int index) {
    return (E) a[index];
}
```

所以读操作的性能很好。

而写操作则需要进行加锁操作，举add()作例子：



```java
public boolean add(E e) {
    //首先进行加锁操作
    synchronized (lock) {
        //获取到旧数组
        Object[] es = getArray();
        int len = es.length;
        //es指向一个新数组，长度比原来大1
        es = Arrays.copyOf(es, len + 1);
        //进行更新
        es[len] = e;
        //新数组替换为旧数组
        setArray(es);
        return true;
    }
    //释放锁
}
```

操作很简单，总结来说就是：

1.加锁

2.建议一个旧数组的副本。

3.在新数组上进行写操作。

4.用新数组替换旧数组。

5.释放锁。

可以看到这个流程中，如果有其他线程并发地执行了读操作，这个读操作是不受影响的，因为读操作在旧数组上进行，而写操作则是在新数组上进行。从而实现读写分离。

但这样的机制也会出现一些问题：

1.读到的数据不能保证实时性。因为读操作始终是在旧数组上进行，所以不能确保读到的数据是最新的数据。

2.内存占用问题。在读操作时需要对旧数组进行拷贝，所以在内存中会出现两个数组，若数组长度过大，可能会出现内存问题。

