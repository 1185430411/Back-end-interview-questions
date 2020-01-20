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
