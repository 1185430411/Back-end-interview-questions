使用Redis实现任务队列首先想到的就应该是Redis的列表类型List，这是因为Redis中的列表类型是由双向链表实现的，符合队列的功能。

实现其实很简单：

- **非阻塞实现**：

生产者使用LPUSH 将任务加入队列，消费者使用RPOP将任务移除队列，一个先入先出的队列就实现了:

```sql
//生产者只需将数据LPUSH到队列中
127.0.0.1:6379> LPUSH queue task
(integer) 1
127.0.0.1:6379> LRANGE queue 0 -1
1) "task"
//消费者从队列中RPOP任务，如果为空则直接返回nil。
127.0.0.1:6379> RPOP queue
"task"
```

这个实现很简单，但是当然也有缺点，那就是当队列没有数据时，也会执行pop操作，此时可以使用阻塞式队列

- 阻塞式实现

```sql
127.0.0.1:6379> BLPOP queue 0   //0表示无限期等待，大于0的数表示阻塞时间
/若当队列为空则，则消费者的POP操作会处于阻塞状态。
//生产者将数据LPUSH到队列中
127.0.0.1:6379> LPUSH queue task
(integer) 1

//消费者立刻“消费”，取出task。
127.0.0.1:6379> BLPOP queue 0
1) "queue"
//此时生产者产生数据，消费者则获取到数据，退出阻塞
2) "task"
//阻塞时间，若超时，会直接返回nil
(13.38s)
```

这样的阻塞实现更像一个队列了

- 优先队列实现

如果有多个队列，消费者优先消费优先队列的数据：

```sql
127.0.0.1:6379> LPUSH queue1 first 
(integer) 1
127.0.0.1:6379> LPUSH queue2 second
(integer) 1

//当两个键都有元素时，按照从左到右的顺序取第一个键中的一个元素
127.0.0.1:6379> BRPOP queue1 queue2 0
1) "queue1"
2) "first"
```

- 订阅发布

前面三个队列的缺点就是：生产者往其中加入了一个任务，这个任务只能被一个消费者接受，能不能像MQ那样，发布一个，多个消费者都能收到呢？是可以的，这就要使用到**发布/订阅（pub/sub）**

```sql
127.0.0.1:6379> PUBLISH channel1.1 task  //往channel1.1这个频道发布一个task
(integer) 0 //有0个客户端收到消息，因为还没有绑定

127.0.0.1:6379> SUBSCRIBE channel1.1   //订阅channel1.1这个频道
Reading messages... (press Ctrl-C to quit)
1) "subscribe"  //"subscribe"表示订阅成功
2) "channel1.1" //订阅成功的频道
3) (integer) 1  //表示频道下，订阅客户端的数量，此时只有自己
//此时发布者发布一个信息task，订阅者会收到如下消息：
1) "message"    //接收到消息
2) "channel1.1" //产生消息的频道
3) "task"       //消息的内容
//当订阅者取消订阅时：
127.0.0.1:6379> UNSUBSCRIBE channel1.1
1) "unsubscribe"    //成功取消订阅
2) "channel1.1" //取消订阅的频道
3) (integer) 0  //当前订阅客户端的数量
```

另外，PSUBSCRIBE可以批量订阅指定的模式，其中？代表一个字符，*代表任意字符，包括0

```sql
127.0.0.1:6379> PSUBSCRIBE channel1.*  //订阅这个模式的频道，以channel1.开头的频道都会被订阅到
Reading messages... (press Ctrl-C to quit)
1) "psubscribe"
2) "channel1.*"
3) (integer) 1
//等待发布者发布消息
127.0.0.1:6379> PUBLISH channel1.1 test1.1
(integer) 1 //发布者在channel1.1上发布消息

//客户端中：
1) "pmessage"   //表示通过PSUBSCRIBE命令订阅而收到的
2) "channel1.*" //表示订阅时使用的通配符
3) "channel1.1" //表示收到消息的频道
4) "test1.1"        //表示消息内容

127.0.0.1:6379> PUBLISH channel1.2 test1.2
(integer) 1 //发布者在channel1.2发布消息

1) "pmessage"
2) "channel1.*"
3) "channel1.2"
4) "test1.2"

127.0.0.1:6379> PUBLISH channel1.3 test1.3
(integer) 1 //发布者在channel1.3发布消息

//客户端中
1) "pmessage"
2) "channel1.*"
3) "channel1.3"
4) "test1.3"

127.0.0.1:6379> PUNSUBSCRIBE channal1.*
1) "punsubscribe"   //退订成功
2) "channal1.*" //退订规则的通配符
3) (integer) 0  //表示当前订阅客户端的数量
```
