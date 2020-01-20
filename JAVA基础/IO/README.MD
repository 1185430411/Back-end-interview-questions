### 概述：JAVA IO流可以按两种方法进行分类：
- 按流向分类可分为：
	* 输入流（InputStream,Reader）
	* 输出流	(OutputStream,Writer)
- 按操作类型进行分类：
	* 字节流	(InputStream,OutputStream)：字节流可以操作任何数据，因为计算机中的数据都是以字节的形式存储的。
	* 字符流	(Redaer,Writer):字符流只能操作字符数据，是为了防止乱码。



### 一.使用FileIntputStream和FileOutputStream对文件进行读写操作。
- **FileIntputStream**：
```java
@Test
    public void testFileInputStream(){
        FileInputStream fileOutputStream=null;
        try {
            fileOutputStream=new FileInputStream("D:/abc.txt");
            int temp;
            while((temp=fileOutputStream.read())!=-1)
                System.out.print((char)temp);
        } catch (FileNotFoundException e) {
            e.printStackTrace();
        } catch (IOException e) {
            e.printStackTrace();
        }finally {
            try {
                fileOutputStream.close(); //关闭资源
            } catch (IOException e) {
                e.printStackTrace();
            }
        }
    }
```
这样每次输出一个字符temp
但这样一次只读一个字符实在是太慢了，所以还可以使用byte数组，一次进行多此读取：
```javascript
@Test
    public void testFileInputStream2(){
        FileInputStream fileOutputStream=null;
        try {
            fileOutputStream=new FileInputStream("D:/abc.txt");
            byte[]temp=new byte[3];
            int num; //num记录一次读取了多少个字节
            while((num=fileOutputStream.read(temp))!=-1)
                System.out.print(new String(temp,0,num)); //从0到num就是本次读取的长度
        } catch (FileNotFoundException e) {
            e.printStackTrace();
        } catch (IOException e) {
            e.printStackTrace();
        }finally {
            try {
                fileOutputStream.close();
            } catch (IOException e) {
                e.printStackTrace();
            }
        }
    }
```

- **FileOutputStream**
```javascript
@Test
    public void testFileOutPutStream(){
        FileOutputStream fileOutputStream=null;
        try {
            fileOutputStream=new FileOutputStream("D://def.txt",true);  //true待会要追加写入
            String msg="jayice";
            fileOutputStream.write(msg.getBytes());
            fileOutputStream.flush();

        } catch (FileNotFoundException e) {
            e.printStackTrace();
        } catch (IOException e) {
            e.printStackTrace();
        }finally {
            try {
                fileOutputStream.close();
            } catch (IOException e) {
                e.printStackTrace();
            }
        }
    }
```

- 练习：**使用FileInputStream和FileOutputStream进行文件拷贝**

```javascript
@Test
    public void copyFile(){
        FileInputStream fileInputStream=null;
        FileOutputStream fileOutputStream=null;
        try {
            fileInputStream = new FileInputStream("D://abc.txt");  //源文件
            fileOutputStream = new FileOutputStream("D://target.txt");  //目标文件
            byte[]arr=new byte[1024];
            int temp;
            while((temp=fileInputStream.read(arr))!=-1){
                fileOutputStream.write(arr,0,temp);
            }
        } catch (FileNotFoundException e) {
            e.printStackTrace();
        } catch (IOException e) {
            e.printStackTrace();
        }finally {
            try {
                fileInputStream.close();
                fileOutputStream.close();
            } catch (IOException e) {
                e.printStackTrace();
            }
        }
    }
```

### 二.Buffered缓冲流。
Java中提供了BufferedInputStream和BufferedOutputStream缓冲流用来读取和写出（装饰者模式）。前者会创建一个长度为8192的byte数组，所以我们在读操作时无需再定义byte数组。
其实本质上使用缓冲流与直接使用FileInputStream等没有太大差别，底层都是使用了byte数组，只不过缓冲流帮我们装饰起来了
此时我们试着用Buffered缓冲流来进行文件拷贝：
```javascript
@Test
    public void testBuffered(){
        BufferedInputStream bufferedInputStream=null;
        BufferedOutputStream bufferedOutputStream=null;
        try {
            bufferedInputStream=new BufferedInputStream(new FileInputStream("D://abc.txt"));  //装饰
            bufferedOutputStream = new BufferedOutputStream(new FileOutputStream("D://buffered.txt"));
            int temp;
            while((temp=bufferedInputStream.read())!=-1){
                bufferedOutputStream.write(temp); //直接写
            }
        } catch (FileNotFoundException e) {
            e.printStackTrace();
        } catch (IOException e) {
            e.printStackTrace();
        }finally {
            try {
                bufferedInputStream.close();
                bufferedOutputStream.close();
            } catch (IOException e) {
                e.printStackTrace();
            }
        }
    }
```
### 三.Reader与Wirter
在概述中我们已经说过，这两个是字节流的，用来解决中文乱码问题。要搞明白它的作用，我们首先得还原一下问题。
之前我们在文件上进行的数据操作都是英文，那如果换成中文呢？就会出现乱码问题（就不演示了）。
所以此时要使用字符流来处理中文数据

- **Reader**
```javascript
@Test
    public void testReader(){
        FileReader fileReader=null;
        try {
            fileReader=new FileReader("D://chinese.txt");
            int temp;
            while((temp=fileReader.read())!=-1){
                System.out.println((char)temp);
            }
        } catch (FileNotFoundException e) {
            e.printStackTrace();
        } catch (IOException e) {
            e.printStackTrace();
        }finally {
            try {
                fileReader.close();
            } catch (IOException e) {
                e.printStackTrace();
            }
        }
    }
```

- **Writer**
 ```javascript
@Test
    public void testWriter(){
        FileWriter fileWriter=null;
        try {
            fileWriter=new FileWriter("qwe.txt");
            fileWriter.write("hello jayice");
            fileWriter.write(102);
        } catch (IOException e) {
            e.printStackTrace();
        }finally {
            try {
                fileWriter.close();
            } catch (IOException e) {
                e.printStackTrace();
            }
        }
    }
```

### 四.Buffered缓冲流装饰Reader和Writer
BufferedReader可以实现一次读一行，废话不多说，直接上代码。
```javascript
@Test
    public void testBufferedReader(){
        BufferedReader bufferedReader=null;
        try {
            bufferedReader=new BufferedReader(new FileReader("D://abc.txt"));
            String s;
            while((s=bufferedReader.readLine())!=null){
                System.out.println(s);
            }
        } catch (FileNotFoundException e) {
            e.printStackTrace();
        } catch (IOException e) {
            e.printStackTrace();
        }
    }
```
而BufferedWriter自然也可以写文件

```javascript
@Test
    public void testBufferedWriter(){
        BufferedWriter bufferedWriter=null;
        try {
            bufferedWriter=new BufferedWriter(new FileWriter("D://asd.txt"));
            bufferedWriter.write("你好世界");
            bufferedWriter.write("dqwdq");
            bufferedWriter.newLine();
            bufferedWriter.write("我喜欢打篮球");
            bufferedWriter.flush();
        } catch (IOException e) {
            e.printStackTrace();
        }
    }
```
这下再用BufferedReader与BufferedWriter拷贝文件
```javascript
@Test
    public void testCopy(){
        BufferedWriter bufferedWriter=null;
        BufferedReader bufferedReader=null;
        try {
            bufferedReader=new BufferedReader(new FileReader("D://asd.txt"));
            bufferedWriter=new BufferedWriter(new FileWriter("D://target.txt"));
            String s;
            while((s=bufferedReader.readLine())!=null){
                bufferedWriter.write(s);
                bufferedWriter.newLine();
            }
            bufferedWriter.flush();
        } catch (FileNotFoundException e) {
            e.printStackTrace();
        } catch (IOException e) {
            e.printStackTrace();
        }
    }
```

## 面试题
#### 1.java中有几种类型的流？
A：一般来说有两种区分法
- 按字节流和字符流分成两种：字节流是基于二进制数据操作的，都继承于InputStream或者OutputStream这两个抽象类。而字符流是基于字符操作的，一般用来解决非英文数据的乱码问题，字符流类都继承于Reader和Reader这两个类。
- 按照输入输出流也可分成两种：输入流是指从内存或者数据通道上读取数据，比如说FileInputStream和FileReader；而输出流是指向内存或者数据通道中写入数据，比如说FileOutputStream和FileWriter。


#### 2.字节流和字符流的区别
A：字节流是最基本的，计算机的所有数据都采用二进制数据的方法进行传输，而字节流进行操作的最基本单位就是单个字节，所以字节流通常被用来处理对二进制数据。Java中最基本的两个字节流类就是InputStream和OutputStream，所以继承它们的子类都是基于字节流操作的。
而字符流进行操作的最基本单位是Unicode码元（大小为两个字节），字符流经常被用来处理文本数据。Java中最基本的两个字符流父类就是Reader和Writer。所有继承它们两个的子类都是基于字符流操作的。

#### 3.缓冲流与非缓冲流的区别。
A：缓冲流的底层实现了一个缓冲区，其实就是一个长度为8192的byte类型数组，这样可以解决非缓冲流一次只能读取一个字符或者字节导致的低效率问题。另外使用缓冲流还需要注意的是，缓冲区中的数据不会像非缓冲流一样被自动提交，只有当流资源被close()，或者显式地调用flush()，缓冲区中的数据才会被写到目标资源中。
另外如果读写数据非常大的话，也应该使用缓冲流提高效率，还可以降低对硬盘的损伤。

#### 4.什么叫对象序列化，什么叫反序列化，如何实现？
A：序列化就是将对象以二级制数据的形式保存在硬盘中。反序列化就是将二进制文件读取之后转化为对象。
实现方法是让需要进行序列化的类实现Serializable接口，如果不希望类中的某个属性参与序列化，就用transient关键字来修饰该属性。

#### 5.在实现序列化接口的时候一定要生成一个serialVersionUID字段吗？有什么用？
A：不一定。
serialVersionUID是版本号，为了保持版本号的一致来进行序列化。如果没有显式声明该字段的话，JVM会给该类分配一个默认的版本号，每次类中的属性被修改的时候，该版本号都会被修改。而将二进制数据文件进行反序列化并用该类对象去进行接收的时候，如果双方版本号不一致，那么就会报错。而我们显式地声明版本号，就不会出现这个问题。
再配合使用ObjectInputStream和ObjectOutputStream对对象进行转化后的读写就可以完成序列化与反序列化

#### 6.BufferedReader是属于哪种流，有啥用？里面有啥经典的方法？
A：BufferedReader被缓冲流修饰的字符输入流。
首先它是一个输入流，所以能把数据从内存或者数据通道中读取出来；然后它是一个字符流，它的最小处理单位是一个unicode单元，所以经常被用在处理文本数据的场景下；最后它是一个缓冲流，底层实现了缓冲区，把数据写在缓冲区，可以减少客户端和内存的交互次数，提高性能。另外还需要提的是需要手动调用其flush()方法才可以将缓冲区中的数据输出。
经典方法就是readLine()方法，一次可以读取文件中的一行数据，返回一个String类型的对象。
