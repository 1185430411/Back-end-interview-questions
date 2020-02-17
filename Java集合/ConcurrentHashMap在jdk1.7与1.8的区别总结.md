ConcurrentHashMap 1.7与1.8的区别总结

这应该是个很经典的面试题了，写个帖子大概总结下：

### 1.锁结构不同

在JDK1.7中，ConcurrentHashMap基于Segment+HashEntry数组实现的。Segment是Reentrant的子类，而其内部也维护了一个Entry数组，这个Entry数组和HashMap中的Entry数组是一样的。所以说Segment其实是一个锁，可以锁住一段哈希表结构，而ConcurrentHashMap中维护了一个Segment数组，所以是基于分段锁实现的。 而JDK1.8中，ConcurrentHashMap摒弃了Segment，而是采用synchronized+CAS+红黑树来实现的。锁的粒度也从段锁缩小为结点锁

### 2.put()的执行流程有所不同

JDK1.7中，ConcurrentHashMap要进行两次定位，先对Segment进行定位，再对其内部的数组下标进行定位。定位之后会采用自旋锁+锁膨胀的机制进行加锁，也就是自旋获取锁，当自旋次数超过64时，会发生膨胀，直接陷入阻塞状态，等待唤醒。并且在整个put操作期间都持有锁。

而在JDK1.8中只需要一次定位，并且采用CAS+synchronized的机制。如果对应下标处没有结点，说明没有发生哈希冲突，此时直接通过CAS进行插入，若成功，直接返回。若失败，则使用synchronized进行加锁插入。

### 3.计算size的方法不一样

1.7：采用类似于乐观锁的机制，先是不加锁直接进行统计，最多执行三次，如果前后两次计算的结果一样，则直接返回。若超过了三次，则对每一个Segment进行加锁后再统计。

1.8：会维护一个baseCount属性用来记录结点数量，每次进行put操作之后都会CAS自增baseCount

### 4.引入了红黑树

这点和HashMap相同，引入了红黑树结构用降低哈希冲突严重的场景的时间复杂度。

