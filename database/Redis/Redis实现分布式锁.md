[TOC]

分布式锁，是控制分布式系统之间同步访问共享资源的一种方式。在分布式系统中，往往需要互斥来防止共享资源的并发访问，从而保证数据一致性。

**分布式锁需要具备的特性**：

- **互斥性**：任意时刻都只有一个客户端获得锁，其他申请锁的客户端必须等待
- **安全性**：锁只能被持有的客户端删除，其他客户端不能删除
- **避免死锁**：获取锁的客户端，一定也要释放锁，否则如果客户端宕机了，资源就被永远锁住了。
- **容错**：当客户端出现了宕机，可以实现高可用

## 使用setnx expire del实现

由于Redis的操作都是原子性的，所以Redis可以使用setnx expire del命令实现：

- setnx（set if not exits）命令：**命令格式**：SETNX key value   只有当key不存在时，才存入这个key-value值，并返回true；否则返回false
- expire命令：**命令格式**：EXPIRE key seconds  为key设置为生存时间，若超时，则会被自动删除。

逻辑如下：

客户端执行setnx方法，若设置成功了(返回1),则获取锁成功，设置过期时间（这个时间可以当作是最长持有时间），在这个事件内执行任务，执行完，执行del删除（因为可能提前执行完了，还需要等待过期？很明显不合理）

若设置失败了(返回0),说明此时有其他客户端占有锁资源，此时需要等待

![img](https://img-blog.csdnimg.cn/20190303135212531.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L2Rhem91MQ==,size_16,color_FFFFFF,t_70)

缺陷：虽然setnx expire del都是原子操作，但是它们组合在一起就是不是原子操作了。有可能执行完setnx成功后，客户端宕机了，没有设置超时，此时就会发生死锁。

问题的本质还是，这一系列操作不是原子性的。

## 使用set实现

SET key value [EX seconds] [PX milliseconds] [NX|XX]

如：set key value ex 10 nx

这样就使得加锁，设置过期这一系列操作是一个原子操作了。

## 如果大量key同时过期怎么办？

清除大量的key很耗时，有可能会出现卡顿，我们在设置过期时间的时候可能在一个范围内选一个随机的时间来作为过期时间
