[TOC]

# HashMap中储存数据的单位Node

HashMap是一种基于散列数组与链表实现的哈希表，在JDK1.8之后，变成了散列数组+链表+红黑树的结构来实现。

HashMap内部实际上是维护了一个Node数组

```java
static class Node<K,V> implements Map.Entry<K,V> {
    final int hash;
    final K key;
    V value;
    Node<K,V> next;

    Node(int hash, K key, V value, Node<K,V> next) {
        this.hash = hash;
        this.key = key;
        this.value = value;
        this.next = next;
    }
    ...
 }
```

可以看到，Node结点里储存了key-value对，一个哈希值，一个next指针，也就是说，其基本实现是一个链表的结点。因为HashMap解决哈希冲突的方法是链地址法，所以在发生哈希冲突的时候，会在发生冲突的位置形成一条链表。我们都知道，链表不支持随机访问，只支持顺序访问。所以如果发生了哈希冲突，那么我们查询操作的时间复杂度就不再是O(1)了，而是还要加上遍历链表的时间。

当然，仅仅是链表结点显然不够，因为在JDK1.8之后还加入了红黑树，所以还需要有树结点才行：



```java
static final class TreeNode<K,V> extends LinkedHashMap.Entry<K,V> {
    TreeNode<K,V> parent;  // red-black tree links
    TreeNode<K,V> left;
    TreeNode<K,V> right;
    TreeNode<K,V> prev;    // needed to unlink next upon deletion
    boolean red;
    ....
}
```

可以看到，TreeNode结点继承了LinkedHashMap.Entry，而LinkedHashMap.Entry则是继承了上面所说的Node结点，所以，TreeNode实际上是Node结点的子类，并加上了parent，left，right，pre等树指针。

# HashMap中的一些重要属性

## DEFAULT_INITIAL_CAPACITY 

```java
/**
 * The default initial capacity - MUST be a power of two.
 */
static final int DEFAULT_INITIAL_CAPACITY = 1 << 4; // aka 16
```

HashMap中的默认初始容量，为16，这个值一定要是2的幂次数。也可以通过构造函数传入，但如果传入的值不是2的幂次数，HashMap会自动将其转化为大于它且最接近它的2幂次数。

## MAXIMUM_CAPACITY

```java
/**
 * The maximum capacity, used if a higher value is implicitly specified
 * by either of the constructors with arguments.
 * MUST be a power of two <= 1<<30.
 */
static final int MAXIMUM_CAPACITY = 1 << 30;
```

最大容量，为2的30次方，当容量超过该值时，就不会进行扩容了。基本不会达到这个值

## DEFAULT_LOAD_FACTOR

```java
/**
 * The load factor used when none specified in constructor.
 */
static final float DEFAULT_LOAD_FACTOR = 0.75f;
```

负载因子，默认为0.75。用来控制HashMap进行扩容的阈值，也就是说当HashMap当前容量大于当前数组长度的0.75倍时，就要进行一次扩容。那为什么要设定为0.75呢?而不是像ArrayList那样满了再进行扩容呢？因为HashMap发生哈希冲突的时候，查询性能是会被影响的，所以HashMap要尽量避免哈希冲突。而负载因子设置得太低，会使扩容太频繁，若设置得太高，会导致哈希冲突严重。0.75应该是一个概率统计的结果，当数据数量达到数组的容量的0.75时，普遍来说，发生哈希冲突增加的开销会大于扩容带来的开销，此时进行扩容是比较合适的。所以设置为0.75

## TREEIFY_THRESHOLD

```java
/**
 * The bin count threshold for using a tree rather than list for a
 * bin.  Bins are converted to trees when adding an element to a
 * bin with at least this many nodes. The value must be greater
 * than 2 and should be at least 8 to mesh with assumptions in
 * tree removal about conversion back to plain bins upon
 * shrinkage.
 */
static final int TREEIFY_THRESHOLD = 8;
```

链表转化为红黑树的阈值。若链表长度超过8，则满足了转化为红黑树的一个条件

## MIN_TREEIFY_CAPACITY 

```java
/**
 * The smallest table capacity for which bins may be treeified.
 * (Otherwise the table is resized if too many nodes in a bin.)
 * Should be at least 4 * TREEIFY_THRESHOLD to avoid conflicts
 * between resizing and treeification thresholds.
 */
static final int MIN_TREEIFY_CAPACITY = 64;
```

链表转红黑树的最小容量。只有当链表长度达到了8，且数组容量达到了64，才会进行转化。只满足前者，只会进行一次扩容

## UNTREEIFY_THRESHOLD

```java
/**
 * The bin count threshold for untreeifying a (split) bin during a
 * resize operation. Should be less than TREEIFY_THRESHOLD, and at
 * most 6 to mesh with shrinkage detection under removal.
 */
static final int UNTREEIFY_THRESHOLD = 6;
```

红黑树转化为链表的阈值，若删除元素，使红黑树结点小于6，则转化为链表



# HashMap部分源码

从我们最熟悉的方法入手，HashMap的put()方法，相信大家都有使用过：

## put()

```java
public V put(K key, V value) {
    return putVal(hash(key), key, value, false, true);
}
```

可以看到，其内部实际上是调用了putVal()方法，并且里面调用有hash()方法，再来看看hash方法是什么妖魔鬼怪：

### hash()

JDK1.7：

```java
static int hash(int h) {
    return h ^ (h >>> 7) ^ (h >>> 4);
}
```

JDK1.8：

```java
static final int hash(Object key) {
    int h;
    return (key == null) ? 0 : (h = key.hashCode()) ^ (h >>> 16);
}
```

相比之下，JDK1.7的实现实在是太鬼畜了。JDK1.8的实现就简单多了，其实就是取key的hashCode与自己的高16为进行亦或操作嘛~这样就获得了这个key的hash值

## putVal()

```java
/**
 * Implements Map.put and related methods.
 *
 * @param hash hash for key
 * @param key the key
 * @param value the value to put
 * @param onlyIfAbsent if true, don't change existing value
 * @param evict if false, the table is in creation mode.
 * @return previous value, or null if none
 */
final V putVal(int hash, K key, V value, boolean onlyIfAbsent,
               boolean evict) {
    Node<K,V>[] tab; Node<K,V> p; int n, i;
    //若数组还未被初始化，则进行初始化
    if ((tab = table) == null || (n = tab.length) == 0)
        n = (tab = resize()).length;
    //注意，这里取了(n - 1) & hash作为下标。若该下标下没有Node，则直接放进去就可以
    if ((p = tab[i = (n - 1) & hash]) == null)
        tab[i] = newNode(hash, key, value, null);
    //若有Node，说明发生了哈希冲突
    else {
        Node<K,V> e; K k;
        //若第一个结点的key就相等，说明直接覆盖就可以
        if (p.hash == hash &&
            ((k = p.key) == key || (key != null && key.equals(k))))
            e = p;
        //否则，若该结点是一个红黑树结点，说明要对红黑树进行插入操作
        else if (p instanceof TreeNode)
            e = ((TreeNode<K,V>)p).putTreeVal(this, tab, hash, key, value);
        //若是链表结点，则需要对链表进行插入
        else {
            //binCount用来记录链表长度
            for (int binCount = 0; ; ++binCount) {
                if ((e = p.next) == null) {
                    p.next = newNode(hash, key, value, null);
                    //为什么-1？因为是从next开始算起。若达到了转换阈值，则进行转换
                    if (binCount >= TREEIFY_THRESHOLD - 1) // -1 for 1st
                        //在这个方法里面还会进行一次判断，若数据数量还未达到64，只会进行一次扩容，而不会转化
                        treeifyBin(tab, hash);
                    break;
                }
                //如果找到了，则break
                if (e.hash == hash &&
                    ((k = e.key) == key || (key != null && key.equals(k))))
                    break;
                p = e;
            }
        }
        //说明找到了该key值，那么直接覆盖，同时返回旧值
        if (e != null) { // existing mapping for key
            V oldValue = e.value;
            if (!onlyIfAbsent || oldValue == null)
                e.value = value;
            afterNodeAccess(e);
            return oldValue;
        }
    }
    //这是一个快失败版本号，其实就是CAS操作的版本号，虽然HashMap不支持并发，但也提供了一些相关方法
    ++modCount;
    //若大于阈值，进行扩容
    if (++size > threshold)
        resize();
    afterNodeInsertion(evict);
    return null;
}
```

这里还是比较好理解的。

主要讲一下取了(n - 1) & hash作为下标这个点。在看前面的内容可能会有疑问，为什么最大容量一定要是2的幂次数呢？其实和这里取下标是有关的。

首先看2的幂次数的二进制是怎么样的？

| Num  | Binary |
| :--: | :----: |
|  2   |   10   |
|  4   |  100   |
|  8   |  1000  |
|  16  | 10000  |

这里是取n-1对hash进行运算的，那它们的n-1是什么形式？

| Num-1 | Binary |
| :---: | :----: |
|   1   |   1    |
|   3   |   11   |
|   7   |  111   |
|  15   |  1111  |

发现所有的低位都是1，那&运算的规则就是，只有当两个数字同时为1的时候才是1，否则为0，如

1011

1111   =

1011

只有2次幂数-1的二进制才是这种形式，那如果不是，那么n-1的二进制就会有0出现，也就是说在进行与操作时，该位的结果肯定是0，所以这就使得数组中至少有一个下标是永远不会被取到的，这就增加哈希冲突的概率

总结一下这里的几个点：

1.在JDK1.8中，是插入，再扩容。而在JDK1.7中是先扩容再插入。

2.在JDK1.8中，扩容触发条件多了一个，也就是当链表长度大于8，但数据总数量小于64时，也会触发一次扩容。为了解决哈希冲突

# HashMap面试题

## 1.说一下HashMap的扩容机制？

HashMap是基于数组实现的，所以也需要扩容机制。

HashMap和扩容机制相关的几个属性有：

1.当前最大容量，必须是2的幂次数，默认初始值是16

2.负载因子，默认值为0.75

3.阈值，为当前最大容量*负载因子，默认为12

4(jdk1.8).转化为红黑树的阈值，默认为8

5.(jdk1.8).转化为红黑树的最小容量，默认为64.

触发扩容的条件有：

1.插入一个新值，且当前容量已经达到了阈值，此时要触发一次扩容。

2(jdk1.8).插入一个新值，链表长度超过了8，但当前容量还未达到64，也需要进行一次扩容。

扩容过程比较简单：建议一个大小为原来容量2倍的新数组，并将旧数组中的元素经过rehash计算之后插入到新的下标，在jdk1.7中，这是一个头插入的操作，所以在多线程环境下会产生死循环。在jdk1.8中，是一个尾插入的操作，解决了死循环的问题。

## 2.HashMap的碰撞检测？解决的方法？

HashMap的碰撞检测实现很简单，就是通过哈希值计算出下标之后，判断下标处是否已经有Node结点，如果有就说明发生了哈希碰撞。

HashMap中解决哈希碰撞的方法是使用链地址法，也就是在同一个位置发生冲突的Node结点会形成一条链表，在检索的时候也需要遍历该链表。

## 3.了解HashMap存在的问题嘛？

存在死循环问题。HashMap进行扩容的时候，需要对旧数组中的元素进行rehash计算然后插入到新下标处，而且插入的方式是头插入，也就是说最后的顺序和原来的顺序是相反的。那么在多线程的环境下，这样的机制很可能会导致链表成环，然后在执行get()方法的时候，就会发生死循环。

另外还有一个在Tomcat中的问题。因为Tomcat会采用HashTable用来储存相关的HTTP请求对象。且解决哈希冲突的方法同样是链地址法，而链表只支持顺序访问，不支持随机访问。如果有黑客精心地设计一大堆hash值相同的请求，那么在Tomcat中会形成一个很长的链表或者是红黑树，会影响服务器性能。

## 4.HashMap在jdk1.7和1.8中的区别：

1.在jdk1.7中采用数组+链表的数据结构实现，而在jdk1.8中采用数组+链表+红黑树的数据结构实现，这样是为了降低哈希冲突严重的场景之下，查询操作的时间复杂度，从O(N)降低为O(logN)

2.在jdk1.7中，当哈希表为空时，会调用inflateTable()初始化一个数组，而在jdk1.8中会直接使用resize()方法对数组进行扩容。

3.在jdk1.7中，链地址法采用的是头插法，而在jdk1.8中采用尾插法。而头插法会导致在进行扩容的时候，原来的顺序和新插入的顺序相反，在多线程的场景下会出现链表成环，继而导致死循环的问题

4.计算hash值的算法也不同。在jdk1.7中，对key计算hash值是采用了三次位运算和两次亦或操作，比较复杂。而在jdk1.8中的实现就比较简单，直接取key的hashCode与自己的高16位进行亦或运算。

5.同样是插入操作，在jdk1.7中，如果有需要的话，会先进行扩容再进行插入，而在jdk1.8中，会先进行插入再进行扩容

6.扩容时机有所不同。在jdk1.7中，插入一个新值时，如果当前数据数量超过了阈值，就会触发一次扩容。而在jdk1.8中，除此之外，还有一个触发时间：当链表长度超过8，但当前数据数量还没超过64时，也会触发一次扩容
