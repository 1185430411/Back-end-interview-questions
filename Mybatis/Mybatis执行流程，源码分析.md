@[toc]
## 简单使用Mybatis

在看Mybatis的内部执行原理之前，先简单看一下我们要怎么样配置好然后进行使用：

先看一下整个结构：

![在这里插入图片描述](https://img-blog.csdnimg.cn/20200227002429689.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L1NDVVRKQVk=,size_16,color_FFFFFF,t_70)

mybatis.xml:

![在这里插入图片描述](https://img-blog.csdnimg.cn/20200227002439557.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L1NDVVRKQVk=,size_16,color_FFFFFF,t_70)

StudentMapper2.xml:

![在这里插入图片描述](https://img-blog.csdnimg.cn/20200227002448964.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L1NDVVRKQVk=,size_16,color_FFFFFF,t_70)

StudentDao2:

![在这里插入图片描述](https://img-blog.csdnimg.cn/20200227002457918.png)

基本的配置大概就是这样了，然后就是使用了：

![在这里插入图片描述](https://img-blog.csdnimg.cn/20200227002507314.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L1NDVVRKQVk=,size_16,color_FFFFFF,t_70)

顺序大概是这样的：

1.获取配置文件的输入流对象

1.以输入流对象作为参数，获取SqlSessionFactory对象

2.通过其工厂类对象，创建一个SqlSession

3.通过SqlSession对象获取接口对象

4.调用接口对象的方法，完成数据库操作。

## 源码解析

### 获取配置文件的输入流

OK，那一切都是从获取mybatis配置文件的输入流对象开始的，我们点进getResourceAsStream()方法里面康康到底发生了什么

```java
public static InputStream getResourceAsStream(String resource) throws IOException {
    //把ClassLoader作为参数作为null，直接调用了重载函数
    return getResourceAsStream((ClassLoader)null, resource);
}
```

点进这个重载函数看看：

```java
public static InputStream getResourceAsStream(ClassLoader loader, String resource) throws IOException {
    InputStream in = classLoaderWrapper.getResourceAsStream(resource, loader);
    if (in == null) {
        throw new IOException("Could not find resource " + resource);
    } else {
        return in;
    }
}
```

噢~原来需要返回的输入流对象就是在这里创建好的，点进去classLoaderWrapper.getResourceAsStream()看看是怎么创建的？它最终会调用这个方法：

```java
InputStream getResourceAsStream(String resource, ClassLoader[] classLoader) {
    ClassLoader[] var3 = classLoader;
    int var4 = classLoader.length;

    for(int var5 = 0; var5 < var4; ++var5) {
        ClassLoader cl = var3[var5];
        if (null != cl) {
            //用类加载器来加载资源
            InputStream returnValue = cl.getResourceAsStream(resource);
            if (null == returnValue) {
                returnValue = cl.getResourceAsStream("/" + resource);
            }

            if (null != returnValue) {
                return returnValue;
            }
        }
    }

    return null;
}
```

噢~这个输入流是通过类加载器来加载的啊，那看看cl.getResourceAsStream(resource)，最终会到这里：

```java
public URL getResource(String name) {
    Objects.requireNonNull(name);
    URL url;
    if (parent != null) {
        url = parent.getResource(name);
    } else {
        url = BootLoader.findResource(name);
    }
    if (url == null) {
        url = findResource(name);
    }
    return url;
}
```

这不就是一个双亲委派模型吗？没错，就是类加载器通过双亲委派模型加载出对应的URL类对象：

![在这里插入图片描述](https://img-blog.csdnimg.cn/20200227002524404.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L1NDVVRKQVk=,size_16,color_FFFFFF,t_70)

然后再由URL类对象创建配置文件的输入流对象，然后再返回，至此，**Resources.getResourceAsStream("mybatis.xml")**这个函数就执行完成啦！

### 获取SqlSessionFactory对象

获取到了配置文件的输入流对象，接下来就根据这个对象，获取SqlSessionFactory这个工厂类对象，也就是：

```java
SqlSessionFactory sessionFactory= new SqlSessionFactoryBuilder().build(Resources.getResourceAsStream("mybatis.xml"));
```

的build方法。build()方法最终就调用它的这个重载方法：

```java
public SqlSessionFactory build(InputStream inputStream, String environment, Properties properties) {
    SqlSessionFactory var5;
    try {
        //主要是这一行
        XMLConfigBuilder parser = new XMLConfigBuilder(inputStream, environment, properties);
        var5 = this.build(parser.parse());
    } catch (Exception var14) {
        throw ExceptionFactory.wrapException("Error building SqlSession.", var14);
    } finally {
        ErrorContext.instance().reset();

        try {
            inputStream.close();
        } catch (IOException var13) {
            ;
        }

    }

    return var5;
}
```

这一行执行起来其实很复杂。但总的来说，就是把配置文件的信息加载进来一个叫做Configuration对象实例中了。

parser.parse()其实就是获取了这个Configuration对象，也就是说，我们写的xml配置文件的内容，还有数据库相关信息，其实都储存在其中了，然后调用另一个重载方法，传入这个Configuration对象

```java
var5 = this.build(parser.parse());
```

该方法就是

```java
public SqlSessionFactory build(Configuration config) {
    return new DefaultSqlSessionFactory(config);
}
```

然后就获得一个DefaultSqlSessionFactory(SqlSessionFactory的子类)对象实例了。

至此，我们的SqlSessionFactory对象就创建完成了，其中就包括Configuratin对象，也就是配置文件的信息都存在里面啦~

### 创建SqlSession

```java
SqlSession sqlSession=sessionFactory.openSession();
```

也就是这句

先来看一下openSession()方法：

```java
public SqlSession openSession() {
    return this.openSessionFromDataSource(this.configuration.getDefaultExecutorType(), (TransactionIsolationLevel)null, false);
}
```

调用了openSessionFromDataSource方法，那再看看这个方法吧：

```java
private SqlSession openSessionFromDataSource(ExecutorType execType, TransactionIsolationLevel level, boolean autoCommit) {
    Transaction tx = null;

    DefaultSqlSession var8;
    try {
        Environment environment = this.configuration.getEnvironment();
        TransactionFactory transactionFactory = this.getTransactionFactoryFromEnvironment(environment);
        tx = transactionFactory.newTransaction(environment.getDataSource(), level, autoCommit);
        Executor executor = this.configuration.newExecutor(tx, execType);
        var8 = new DefaultSqlSession(this.configuration, executor, autoCommit);
    } catch (Exception var12) {
        this.closeTransaction(tx);
        throw ExceptionFactory.wrapException("Error opening session.  Cause: " + var12, var12);
    } finally {
        ErrorContext.instance().reset();
    }

    return var8;
}
```

其实就是对Configuration里面的东西一顿加载，然后逐个赋值给DefaultSqlSession对象，然后进行返回。

那看看返回的SqlSession对象里面有什么东西：

![在这里插入图片描述](https://img-blog.csdnimg.cn/20200227002548135.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L1NDVVRKQVk=,size_16,color_FFFFFF,t_70)

里面同样有一个Configuration对面，里面同样存有数据库相关信息。

那SqlSession也成功获得了。

### 获取Mapper接口对象

获取了SqlSession之后，

```java
StudentDao2 studentDao2=sqlSession.getMapper(StudentDao2.class);
```

就可以获取对应的接口对象（也就是和Mapper文件对应的那些接口），之后便可以调用接口中的方法了。

这个地方很神奇，为什么能直接调用接口的方法？**其实是调用了动态代理的设计模式来做到的**，具体的代理模式介绍呢，可以看一下这篇文章：

 [设计模式（十二）代理模式](https://blog.csdn.net/niunai112/article/details/79810490). 

接下来继续看代码吧：

getMapper()函数最终会调用这个方法：

```java
public <T> T getMapper(Class<T> type, SqlSession sqlSession) {
    //这里是一个map，根据传入的Class对象获取对应的代理类
    MapperProxyFactory<T> mapperProxyFactory = (MapperProxyFactory)this.knownMappers.get(type);
    if (mapperProxyFactory == null) {
        throw new BindingException("Type " + type + " is not known to the MapperRegistry.");
    } else {
        try {
            //这个工厂类去创建代理类
            return mapperProxyFactory.newInstance(sqlSession);
        } catch (Exception var5) {
            throw new BindingException("Error getting mapper instance. Cause: " + var5, var5);
        }
    }
}
```

这里的关键就是这个knownMappers了，这里面存的是啥呢？请看：

![在这里插入图片描述](https://img-blog.csdnimg.cn/20200227002559354.png)

其实这就是一个HashMap，以接口的Class对象为key，对应代理类的工厂类作为value，这样在sqlSession中传入Class对象，其实就能直接获取到代理类的工厂类了。然后再调用其newInstance()方法

```java
public T newInstance(SqlSession sqlSession) {
    MapperProxy<T> mapperProxy = new MapperProxy(sqlSession, this.mapperInterface, this.methodCache);
    return this.newInstance(mapperProxy);
}
```

this.newInstance(mapperProxy)其实就是：

```java
protected T newInstance(MapperProxy<T> mapperProxy) {
    return Proxy.newProxyInstance(this.mapperInterface.getClassLoader(), new Class[]{this.mapperInterface}, mapperProxy);
}
```

mapperInterface的内容： interface com.jay.dao.StudentDao2。这就是一个很经典的动态代理模式了！

可以看到，我们获得的接口对象，其实准确来说并不是接口对象，而是Proxy代理类对象：

![在这里插入图片描述](https://img-blog.csdnimg.cn/20200227002610170.png)

那么至此，就很明朗了，我们获得了接口的代理类对象，接下来的方法的调用，其实都只是告诉代理对象：我用了xxx方法，你去调用对应的jdbc吧！

### 接口方法的调用

接下里就调用接口的方法了，以getAll2()为例，当前是会进入到代理类的invoke()方法中：

```java
public Object invoke(Object proxy, Method method, Object[] args) throws Throwable {
    try {
        //如果它是Object中的方法，比如hashCode(),equals()
        if (Object.class.equals(method.getDeclaringClass())) {
            return method.invoke(this, args);
        }
        //如果它是实现的方法，而不是抽象方法
        if (this.isDefaultMethod(method)) {
            return this.invokeDefaultMethod(proxy, method, args);
        }
    } catch (Throwable var5) {
        throw ExceptionUtil.unwrapThrowable(var5);
    }
    //我们调用的是接口方法，所以就会走到这里
    
    //这个就是查缓存
    MapperMethod mapperMethod = this.cachedMapperMethod(method);
    //这个就是真正执行sql语句，也就是调用jdbc
    return mapperMethod.execute(this.sqlSession, args);
}
```

调用完jdbc之后，把结果封装成需要的数据结构，如果有需要的话，还要写缓存，然后再进行返回。

这样，Mybatis的一整个操作就完成啦！

所以当面试官问：你知道Mybatis的执行流程吗？

要聊Mybatis的内部执行流程，我觉得应该要先从我们开发人员调用的Mybatis API开始说。

当一些基础配置完成之后，我们想要使用Mybatis，以最初始的方法为例，我们要先创建配置文件的InputStream输入流对象，然后作为参数，获得SqlSessionFactory这个工厂类对象，之后调用其openSession()方法获取一个SqlSession实例，sqlSession实例就是关键，我们调用它的getMapper()方法，传入接口的Class对象作为参数，就能获得对应的接口对象了，这里说的接口就是Dao层的那些和Mapper文件对应的接口。然后调用接口的方法就可以完成对应的数据库操作。

那就先从获取配置文件的输入流对象开始说，这个过程概括起来其实很简单，就是根据我们传入的字符串，应用类加载器加载出一个对应URL类对象，使用的自然也是双亲委派模型。然后调用该url类的openStream()方法就可以返回对应的InputStream对象。

获得对象之后，下一步就是要获得SqlSessionFactory这个工厂类对象。其实这一步逻辑很简单但是很重要，这一步做的事情就是对Mybatis配置文件，也就是那个xml文件进行解析，将解析得到的结果放在一个叫Configuration的类对象中，Configuration很重要，是Mybatis的核心组件，我们写的配置文件的设置都存放在其中，包括使用哪个数据库？是线上环境的还是测试环境的？数据库账号密码；还有数据库连接池的一些相关参数。然后将这个Configuration对象作为参数赋值给DefaultSqlSessionFactory的构造函数中，生成工厂类对象然后返回。

工厂类对象获得，下一个就是要获得SqlSession对象，其实这一步也很简单，其实就是创建出一个DefaultSqlSession实例，然后按照Configuration里面设置的属性对其进行初始化之后，将这个DefaultSqlSession对象返回。

获得了SqlSession对象之后，最关键的一步来了，那就是调用它的getMapper()方法，传入接口的Class对象，就会返回对应的接口实例了。那这时候很奇怪的一点是，为什么我们后面可以直接调用这个接口的方法呢？其实这里用了动态代理的设计模式，准确来说，我们得到的接口对象并不是接口对象，而是Proxy这个代理类实例。所以接口的方法调用最终都会去到代理类对象的invoke()方法上。具体源码上的实现上，就是在这个方法里面，会去一个叫做knownMappers的结构中，这个结构其实就是一个HashMap，它的key为接口的Class对象，而value就是对应代理类的工厂类，所以说，我们传入Class对象，实际上就是来这个HashMap中获取了对应的代理类的工厂类，然后用这个工程类创建出对应的代理类，然后进行返回。所以才说为什么我们获得的其实是一个Proxy类对象。

那获得代理对象之后，一切就明朗了，接口方法的调用其实都是去到了代理对象的invoke方法中，那invoke方法当然就是访问数据库的地方了，它会先去查询缓存，如果缓存没有或者，才会去执行jdbc来访问数据库，将结果封装为需要返回的对象，有需要的话还要写入缓存再返回结果。然后再执行结束。

Mybatis的一次完整执行的大概流程就是这样！
