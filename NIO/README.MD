### 一.概述

JAVA NIO是JDK 1.4引进的，全称为New IO或者No-Blocking IO。解决了传统IO操作同步阻塞的缺点。
其中的核心组件有：

###### Selector，Channel，Buffer


## Selector：

Selector选择器，是NIO中最核心的组件。
在传统的IO模型中，一个IO操作对应一个线程，这导致每出现一个IO请求，都需要建立一个新的线程，而线程的创建与销毁这部分的开销会导致资源的浪费。即使使用线程池，并发数量也有一定的限制，还可能会出现的问题就是比如说，一个容量为200的线程池内有200个线程在进行大型IO操作，而第201个线程即使只有很少的任务需要处理，也需要等待线程池的空闲。
而Selector的出现就使得一个线程可以处理多个Channel，也就是说一个线程可以和一个Selector绑定，来管理多个IO操作。如图所示：

![图源见水印](https://img-blog.csdn.net/20171129113740705?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvU29taHU=/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/SouthEast)

这样的优点是：能使用更少的线程来管理多个通道，甚至用一个线程来管理所有的通道，这样就减少了线程创建和销毁的开销。并且可以实时监听管道上是否有事件发生，继而来进行处理。

Selector的创建：直接调用Selector类中的静态open方法，就可以返回一个Selector对象了。如下：

```java
Selector selector=Selector.open();
```

而Selector创建完之后，上面没有绑定任何的Channel呀，Selector怎么知道要监听什么东西？

所以说我们在获得Channel对象后，要调用其register()方法来绑定到该Selector中（后面还会写Channel），如图：

```java
	Channel channel=.....;//通过某种方式获得了一个Channel
	channel.register(selector,SelectionKey.OP_READ);
```

注意register的参数，要传入一个selector对象实例，然后一个属性是SelectionKey.OP_READ（另外，选择器只能绑定非阻塞模式的Channel，也就是说FileChannel无法绑定选择器，因为FileChannel无法是非阻塞模式）。这表明了Selector对什么事件感兴趣，是读操作发生的时候我受理呢？还是写操作发生的时候我受理呢？还是都受理呢？所以SelectionKey中的静态属性给我们准备了四种事件：

- **SelectionKey.OP_CONNECT**
- **SelectionKey.OP_ACCEPT**
- **SelectionKey.OP_READ**
- **SelectionKey.OP_WRITE**

分别对应连接连接，接收，读，写操作的监听。当然，如果我同时对读和写都感兴趣呢？可以这样：

```java
int myInterest=SelectionKey.OP_READ | SelectionKey.OP_WRITE;
channel.register(selector,myInterest);
```

使用位或操作来实现。

另外register会返回一个SelectionKey对象，你可以把它理解成一个连接Channel和Selector的连接物。此刻返回的该对象，就是与该Channel对应的一个SelectionKey，如果日后该Channel发生了事件，那么选择器就会获得这个SelectionKey



OK，到目前为止一切都还不错，Selector创建完成了，也朝里面注册了Channel，接下来就是监听Channel是否有事件发生了。那这就要使用Selector中的select()方法。

selector的select()方法是有两个重载方法：

select()  int

select(long time) int

这不就是阻塞方法的一个典型模型嘛~就好像多线程中Future类的get()方法一样。

没错，select()方法是一个阻塞方法，它表示一直监听着所有的Channel，观察是否有事件发生，如果没有，则会一直阻塞下去，而带参数的重载方法也指明了它最大的阻塞事件，若超时还没有监听到结果，那么直接返回-1.

当然想要完全不阻塞，还有一个selectNow()方法，无论是否监听到事件，该方法都会马上返回。

OK，我们假设Selector调用select方法进行监听，突然！监听的所有Channel中，有两个Channel发生了事件，这下可把Selector给高兴坏了，直接把这个两个发生事件的Channel对应的SelectionKey加入到自己内部的一个Set集合中，同时select方法返回2。这下客户端便得知了，有事件发生了，我要去处理了。



```java
	int num=selector.select();  //一直阻塞
	if(num!=-1){
        //开始去处理
    }
```

ok。那怎么样处理呢？刚才说了Selector在监听到事件之后，会把发生事件的Channel对应的SelectionKey加入到自己内部的一个Set集合中，那我们应该可以获得这个集合才对。没错，调用Selector中的selectedKeys()方法就可以获得该集合。

```java
Set<SelectionKey> set=selector.selectedKeys();
```

那接下里，毋庸置疑的是要遍历一下该set对吧。set里面存放的是SelectionKey，那接下里就要小讲一下SelectionKey了。

#### SelectionKey

之前提过，SelectionKey是类似于连接Selector和Channel的一个连接器，在Channel调用register注册到Selector的时候会第一次返回该对象。那SelectionKey里面到底有什么呢？

- interest集合：还记得一开始我们使用register的时候，除了传入selector实例，还传入了一个参数嘛？（SelectionKey.OP_READ）没错，我们此时在SelectionKey内部获得了它，此时我们也可以对其进行修改了，比如说我刚开始是监听读操作的，但是现在返回了，想监听写操作：

```java
//如果是读感兴趣，就改成写感兴趣。
if(selectionKey.interestOps==SelectionKey.OP_READ)
    selectionKey.interesOps=SelectionKey.OP_WRITE;
```

- ready集合：这个也好说。由于感兴趣的事件可能不止一个，所以需要询问到底是发生了哪个事件？

```java
if(selectionKey.isReadable()){
    //read
}
if(selectionkey.isWritable()){
    //write
}
if(selectionKey.isAcceptable()){
    //accept
}
if(selectionKey.isConnectable()){
    //connect
}
....
```

- Channel和Selector：这是最关键的一个点了。因为我们对事件的处理实际上是对Channel进行处理，那么获取了SelectionKey还不行啊，我得拿到Channel才能进行处理吧。所以通过SelectionKey就可以获取到Channel和Selector，实际上虽然很关键，但是很简单

  ```java
  Channel channel=selectionKey.channel();
  //获取到channel，然后开始真正的处理事件啦。
  Selector selector=selectionKey.selector();
  ```

- 附加对象：在注册的时候，实际上还有一个重载方法，可以给SelectionKey绑定一个对象，当然也可以通过SelectionKey获取到该对象。这在实际场景中还是比较好用的，比如Channel和某个对象有关联，在处理Channel的时候也要处理它（比如说像缓存区）

  ```java
  selectionKey.attach(object);
  Object getObject = selectionKey.attachment();
  ```

OK！那现在绑定过了，监听到了，获得了发生事件的Channel。那就要开始说Channel了：绑定到Selector上的Channel怎么来的？监听拿到Channel之后要做什么？

别急，因为使用Channel免不了也要使用Buffer缓存区，所以在说Channel之前，插播一下Buffer缓存区吧~



## Buffer

之所以说要先说Buffer，是因为在NIO上，数据的读取与书写并不发生在Channel上，而是发生在Buffer缓存区上，Channel管道负责将在缓存区上的操作结果进行传输（比如说传递到文件上，或者传递到服务器上）.

Buffer是一个抽象类，JAVA中的每种基本数据类型都有对应的Buffer子类，如IntBuffer，LongBuffer，BooleanBuffer等。

而最常用的就是ByteBuffer了。

ByteBuffer的底层实现是一个byte数组（其他也类似，例如IntBuffer底层就是int数组）。对数据的读写操作就是在这个数组上进行。所以为了记录数据的读写位置，除了数组之外，Buffer还有四个指针，分别是：

- Capacity：最大容量，不可改变，在声明ByteBuffer的时候就已经进行声明。如想要创建一个Capacity为1024的ByteBuffer,是通过静态方法来创建的：

  ```java
  ByteBuffer buf = ByteBuffer.allocate(1024);
  ```

- position：这是数据读写的当前位置，初始的位置为0，每当有一个数据被写进数组，position的位置就+1
- limit：在写模式下：limit指明能写入数据的最大下标，所以和capacity是相等的值。  而在读模式下，limit指向最后一个有效数据的下标。读操作不能超过limit，否则会读到错误的数据。
- mark：标记当前position的位置。常用的手段就是标记当前position的位置，然后向前进行读或者写（同时position向前移动），突然间想回到原来的位置重新写，就可以让position回到mark,也就是原来的地方进行写了。

再介绍一下ByteBuffer的几个通用API：

分配固定长度的Buffer（前面已经写过了）：

```javascript
ByteBuffer buf = ByteBuffer.allocate(1024);
```

写模式，向对象中写数据

```javascript
buf.put("Hello,World!".getBytes());
```

读模式：注意，因为position指针是往前走的，所以要读，需要这样做：

```javascript
/*这个操作是指从写模式转化为读模式。其实是这样实现的：
limit去到position的位置，而position归0，所以说写入的数据就是position到limit的这一段，所以才说limit在读模式下指向最后一个有效数据。
这样操作之后，才可以进行读操作，否则会读到错误内容
*/
buf.flip();
while(buf.hasRemaining()){
    System.out.println((char)buf.get()); //get方法的调用也会使position不断向前。
}
```

或者这样读：

```javascript
System.out.println(new String(buf.array()));
```

在读完一次之后，如果想重新读，就调用rewind()方法。该方法只是把position归0，也就是说读完一次，position本来和limit相等了，现在重置position，可以重复读了。

而clear()方法是直接使所有指针都重置，一般用来重新进行写操作，也就是position归0，limit又回到capacity。但是里面的数据实际上并没有被清除(也就是说并没有被一个个置为null值)，而只是把指针重置，下次写数据的时候直接进行覆盖。

equals方法和compareTo方法可以用来比较两个Buffer：

当满足下列所有条件时，表示两个Buffer相等： 
\- 有相同的类型（byte、int等）。 
\- Buffer中剩余数据个数相等。 
\- Buffer中所有剩余的byte、char等都一一相同。

所以，equals方法只是比较了Buffer中的剩余元素

**CompareTo()方法：** 
compareTo()方法也是比较两个Buffer，同样是原来比较剩余元素， 如果满足下列条件之一，则认为一个Buffer“小于”另一个Buffer： 
\- 第一个不相等的元素小于另一个Buffer中对应的元素 。 
\- 所有元素都相等，但第一个Buffer比另一个先耗尽(第一个Buffer的元素个数比另一个少)。 
（译注：剩余元素是从 position到limit之间的元素）



### Channel

好了，Buffer总算是讲了点了，原来说到哪里来着？遍历Set中的SelectionKey，获得了Channel对吧？OK，聊聊Channel

传统IO都是以流为单位进行数据传输的，一次只能从流中操作一个或者多个字节，并且流的读写都是单向的，输入流只能进行输入，输出流只能输出。

而Channel不一样，Channel是双向的，读和写都可以通过Channel来完成，并且可以异步进行，就是说两个操作可以同时进行。但Channel并不能直接读写数据，数据的读写都是在Buffer中进行的，Channel只是一个管道，可以理解为通向目的地的道路。

Channel有几种主要实现类：

- FileChannel 顾名思义从文件中读写数据的通道。
- SocketChannel 通过TCP读写网络中的数据，类似于Socket。
- ServerSocketChannel 监听新进来的TCP连接，并且每个新进来的连接都可以通过该类的accpet()方法获得SocketChannel对象。类似于ServerSocket也是这样子。
- DatagramChannel 能通过UDP读写网络中的数据

这几个实现类待会一个个说，先从FileChannel开始

- #### FileChannel：

FileChannel是一个连接到文件的通道，可以通过其对文件进行读写，怎么获取呢？Channel都不可以直接new出来，而是要通过InputStream、OutputStream或RandomAccessFile来获取，这里以RandomAccessFile为例：

```java
RandomAccessFile file = new RandomAccessFile("D:/abc.txt", "rw");//可读可写
FileChannel fileChannel = file.getChannel();

```

这下清楚了，原来FileChannel就是这么来的，这个FileChannel就是当时说的“获得Channel，将Channel绑定到Selector上”的那个Channel，同时也是通过SelectionKey获得的那个Channel，都是它。

那么最后一个问题，怎么样通过Channel来进行操作呢？

读数据：

```java
RandomAccessFile file=new RandomAccessFile("D:/abc.html","rw");
FileChannel fileChannel=file.getChannel();
ByteBuffer byteBuffer=ByteBuffer.allocate(1024);
int count;
while((count=fileChannel.read(byteBuffer))!=-1){
    System.out.print(new String(byteBuffer.array()));
    byteBuffer.clear();//万万记得要clear一下，重置指针，接着下次读
}
```

写数据：

```javascript
RandomAccessFile file=new RandomAccessFile("D:/edf.txt","rw");
FileChannel fileChannel=file.getChannel();
ByteBuffer byteBuffer=ByteBuffer.allocate(1024);
byteBuffer.put("kobe bryant".getBytes());
byteBuffer.flip();//切记切记。否则写完之后，position和limit之间是空数据
fileChannel.write(byteBuffer);
```

这下就知道，拿到Channel之后要干啥了吧哈哈

再介绍FileChannel的几个API：

- position()：有两个重载方法，无参数法用来获取当前position指针的位置，有参数法传入一个int参数，用来设置position的位置：

  ```java
  long position = channel.position();
  channel.position(position +123);
  ```

- size():返回关联文件的大小，所以也可以直接把缓存区直接设置为和文件一样的大小，然后就不需要循环进行读操作了，读一次就可以了。

```java
long size=fileChannel.size();
```



- #### SocketChannel

通过SocketChannel的读写功能可以实现服务端与客户端的通信。

SocketChannel的获取方法和FileChannel有所不同：

```java
SocketChannel socketChannel = SocketChannel.open();
socketChannel.connect(new InetSocketAddress("localhost", 8080));
```

首先要通过静态方法创建实例，然后把该实例连接到指定服务器的端口上。

设置非阻塞，这样才能绑定到选择器上，而FileChannel不能设置为非阻塞。

```javascript
socketChannel.configureBlocking(false); //而设置为非阻塞之后，connect方法就算还没有连接成功，方法也会直接返回。所以还需要循环判断是否连接完成。
socketChannel.connect(new InetSocketAddress("localhost", 8080));
while(! socketChannel.finishConnect() ){
    //如果还没有连接成功，则do something else....
}
```

至于SocketChannel的读写方法，和FileChannel基本一样，就不演示了，直接参考上面FileChannel的就可以了。

- #### ServerSocketChannel

ServerSocketChannel可以监听新进来的TCP连接，获得SocketChannel，这点和标准IO中的ServerSocket一样。

创建方法：

```javascript
//创建实例
ServerSocketChannel serverSocketChannel = ServerSocketChannel.open();
//绑定端口
serverSocketChannel.socket().bind(new InetSocketAddress(8080));
//设置非阻塞
serverSocketChannel.configureBlocking(false);

//循环接收TCP请求
while(true){
    SocketChannel socketChannel = serverSocketChannel.accept();
    //do something with socketChannel...
}
```

ServerSocketChannel基本就是只起一个监听新连接的作用，主要是通过它来获取到SocketChannel，然后通过SocketChannel来进行客户端与服务端之间真正的交互。



### 那这就是NIO的几个核心组件了。再回想一下具体步骤：

- 获取Channel（FileChannel就通过getChannel()来获取；SocketChannel等要open()出实例，然后绑定地址和端口）
- 设置为非阻塞，才能注册到Selector中(FileChannel不可以设置为非阻塞)
- register()注册到Selector中
- Selector执行select()方法进行阻塞监听，直到有事件发生，函数才返回
- 调用Selector的SelectionKeys()返回一个装着SelectionKey的Set集合。
- 遍历该集合，对其中的SelectionKey可以获取到对应的Channel
- 对Channel进行操作(比如读写),来完成事件的处理
