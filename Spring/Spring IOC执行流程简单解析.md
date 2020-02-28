@[toc]
# IOC的基本使用

废话不多说，我这里直接贴代码了：

applicationContext.xml

```xml
<beans xmlns="http://www.springframework.org/schema/beans"
       xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
       xmlns:aop="http://www.springframework.org/schema/aop"
       xmlns:context="http://www.springframework.org/schema/context"
       xsi:schemaLocation="http://www.springframework.org/schema/beans
    http://www.springframework.org/schema/beans/spring-beans.xsd
    http://www.springframework.org/schema/aop
    http://www.springframework.org/schema/aop/spring-aop.xsd
    http://www.springframework.org/schema/context
    http://www.springframework.org/schema/context/spring-context.xsd">

    <bean id="studentService" class="com.jay.service.impl.StudentServiceImpl">
        <constructor-arg name="id" value="1"></constructor-arg>
        <property name="name" value="jay"></property>
    </bean>


</beans>
```

StudentService:

```java
package com.jay.service;

public interface StudentService {
    void study();
}
```

StudentServiceImpl: 这里我加了两个属性，并且实现了一个BeanFactoryPostProcessor，在工厂类创建成功后会执行postProcessBeanFactory方法

```java
package com.jay.service.impl;

import com.jay.service.StudentService;
import org.springframework.beans.BeansException;
import org.springframework.beans.factory.config.BeanFactoryPostProcessor;
import org.springframework.beans.factory.config.ConfigurableListableBeanFactory;

public class StudentServiceImpl implements StudentService, BeanFactoryPostProcessor {

    int id;

    String name;

    public StudentServiceImpl(int id) {
        this.id = id;
    }

    public void setName(String name) {
        this.name = name;
    }

    @Override
    public void study() {
        System.out.println("好好学习");
    }

    @Override
    public void postProcessBeanFactory(ConfigurableListableBeanFactory beanFactory) throws BeansException {
        System.out.println("hello factory");
    }
}
```

main:

```java
package com.jay.test;


import com.jay.service.StudentService;
import org.springframework.context.ApplicationContext;
import org.springframework.context.support.ClassPathXmlApplicationContext;

public class StudentTest {
    public static void main(String[] args){
        ApplicationContext applicationContext=new ClassPathXmlApplicationContext("classpath:applicationContext.xml");
        StudentService studentService=(StudentService)applicationContext.getBean("studentService");
        studentService.study();
    }

}
```

# BeanFactory

BeanFactory就是创建Bean的工厂类，它是一个接口，而我们熟悉的ApplicationContext接口就是这个接口的子接口，看结构图吧~：

![在这里插入图片描述](https://img-blog.csdnimg.cn/20200229014628878.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L1NDVVRKQVk=,size_16,color_FFFFFF,t_70)



可以看到，ApplicationContext继承了好多个接口，但在这里主要讨论ListableBeanFactory，它才是爸爸！

# 初始化ApplicationContext

第一步，肯定是从ClassPathXmlApplicationContext的构造方法开始，这个创建ApplicationContext的过程其实就是创建BeanFactory的过程。

```java
public ClassPathXmlApplicationContext(
      String[] configLocations, boolean refresh, @Nullable ApplicationContext parent)
      throws BeansException {

   super(parent);
   //这一步其实是处理配置文件的命名格式
   setConfigLocations(configLocations);
   if (refresh) {
      //这是核心方法，在refresh里完成BeanFactory的创建
      refresh();
   }
}
```

那就看一下refresh方法（为啥取名叫refresh,而不是init啥的？这是因为我们可以调用这个方法对BeanFactory进行重建）

## refresh方法

```java
@Override
public void refresh() throws BeansException, IllegalStateException {
   //加锁，否则在refresh执行过程中，又有其他线程跑来重建怎么办？
   synchronized (this.startupShutdownMonitor) {
      // 这一步很简单，就是记录下容器的启动时间，将状态切换为转换状态什么的。
      prepareRefresh();

      // 这是关键步骤，它会将配置文件中的<bean>标签解析为BeanDefinition对象，然后注册到BeanFactory中
      ConfigurableListableBeanFactory beanFactory = obtainFreshBeanFactory();

      
      prepareBeanFactory(beanFactory);

      try {
         //还记得之前的实现类里面实现了BeanFactoryPostProcessor接口，会在工厂类创建完成后调用postProcessBeanFactory方法吗？没错，这里就是即将要进行后处理
         postProcessBeanFactory(beanFactory);

         // 真正地执行后处理
         invokeBeanFactoryPostProcessors(beanFactory);

         //注册BeanPostProcessor的实现类，里面有两个方法，分别是Bean前处理器和Bean后处理
         registerBeanPostProcessors(beanFactory);

         // Initialize message source for this context.
         initMessageSource();

         // Initialize event multicaster for this context.
         initApplicationEventMulticaster();

         // Initialize other special beans in specific context subclasses.
         onRefresh();

         // Check for listener beans and register them.
         registerListeners();

         //这是重点！前面只是将bean标签进行解析和注册，这里才是真正地创建实例的地方
         finishBeanFactoryInitialization(beanFactory);

         // Last step: publish corresponding event.
         finishRefresh();
      }

      catch (BeansException ex) {
         if (logger.isWarnEnabled()) {
            logger.warn("Exception encountered during context initialization - " +
                  "cancelling refresh attempt: " + ex);
         }

         // Destroy already created singletons to avoid dangling resources.
         destroyBeans();

         // Reset 'active' flag.
         cancelRefresh(ex);

         // Propagate exception to caller.
         throw ex;
      }

      finally {
         // Reset common introspection caches in Spring's core, since we
         // might not ever need metadata for singleton beans anymore...
         resetCommonCaches();
      }
   }
}
```

### obtainFreshBeanFactory方法

这个是全文中最重要的几个方法之一，做了这些事：初始化BeanFactory，加载bean标签，将bean注册到BeanFactory中：

```java
protected ConfigurableListableBeanFactory obtainFreshBeanFactory() {
   refreshBeanFactory();
   ConfigurableListableBeanFactory beanFactory = getBeanFactory();
   if (logger.isDebugEnabled()) {
      logger.debug("Bean factory for " + getDisplayName() + ": " + beanFactory);
   }
   return beanFactory;
}
```

首先是调用了refreshBeanFactory()：

```java
protected final void refreshBeanFactory() throws BeansException {
    //当前的ApplicationContext是否已经包含了BeanFactory，若有则进行销毁
   if (hasBeanFactory()) {
      destroyBeans();
      closeBeanFactory();
   }
   try {
       //初始化一个ListableBeanFactory对象。噢！所以说这个BeanFactory表面上是ApplicationContext，实际上是 DefaultListableBeanFactory
      DefaultListableBeanFactory beanFactory = createBeanFactory();
       //设置BeanFactory的两个属性，即是否允许覆盖和循环依赖
      beanFactory.setSerializationId(getId());
      customizeBeanFactory(beanFactory);
      //将bean加载到beanFactory中
      loadBeanDefinitions(beanFactory);
      synchronized (this.beanFactoryMonitor) {
         this.beanFactory = beanFactory;
      }
   }
   catch (IOException ex) {
      throw new ApplicationContextException("I/O error parsing bean definition source for " + getDisplayName(), ex);
   }
}
```

#### createBeanFactory方法

```java
protected DefaultListableBeanFactory createBeanFactory() {
   return new DefaultListableBeanFactory(getInternalParentBeanFactory());
}
```

这个方法很简单，就是直接new了一个DefaultListableBeanFactory对象而已

#### customizeBeanFactory方法

```java
protected void customizeBeanFactory(DefaultListableBeanFactory beanFactory) {
   if (this.allowBeanDefinitionOverriding != null) {
      beanFactory.setAllowBeanDefinitionOverriding(this.allowBeanDefinitionOverriding);
   }
   if (this.allowCircularReferences != null) {
      beanFactory.setAllowCircularReferences(this.allowCircularReferences);
   }
}
```

也很简单，就是set了一下

#### loadBeanDefinitions方法

这个方法就是很重要的了，这个方法根据配置文件对各个bean进行加载，然后注册到BeanFactory中：

```java
protected void loadBeanDefinitions(DefaultListableBeanFactory beanFactory) throws BeansException, IOException {
   //创建一个XmlBeanDefinitionReader对象
   XmlBeanDefinitionReader beanDefinitionReader = new XmlBeanDefinitionReader(beanFactory);

   // Configure the bean definition reader with this context's
   // resource loading environment.
   beanDefinitionReader.setEnvironment(this.getEnvironment());
   beanDefinitionReader.setResourceLoader(this);
   beanDefinitionReader.setEntityResolver(new ResourceEntityResolver(this));

   // Allow a subclass to provide custom initialization of the reader,
   // then proceed with actually loading the bean definitions.
   initBeanDefinitionReader(beanDefinitionReader);
    //调用了一个重载方法
   loadBeanDefinitions(beanDefinitionReader);
}
```

这个重载loadBeanDefinitions方法其实就是对XML文件一顿解析，得到一棵DOM树，然后解析结果作为参数，调用：

```java
 doRegisterBeanDefinitions(root);
```

```java
protected void doRegisterBeanDefinitions(Element root) {
   // Any nested <beans> elements will cause recursion in this method. In
   // order to propagate and preserve <beans> default-* attributes correctly,
   // keep track of the current (parent) delegate, which may be null. Create
   // the new (child) delegate with a reference to the parent for fallback purposes,
   // then ultimately reset this.delegate back to its original (parent) reference.
   // this behavior emulates a stack of delegates without actually necessitating one.
   BeanDefinitionParserDelegate parent = this.delegate;
   this.delegate = createDelegate(getReaderContext(), root, parent);

   if (this.delegate.isDefaultNamespace(root)) {
      String profileSpec = root.getAttribute(PROFILE_ATTRIBUTE);
      if (StringUtils.hasText(profileSpec)) {
         String[] specifiedProfiles = StringUtils.tokenizeToStringArray(
               profileSpec, BeanDefinitionParserDelegate.MULTI_VALUE_ATTRIBUTE_DELIMITERS);
         if (!getReaderContext().getEnvironment().acceptsProfiles(specifiedProfiles)) {
            if (logger.isInfoEnabled()) {
               logger.info("Skipped XML bean definition file due to specified profiles [" + profileSpec +
                     "] not matching: " + getReaderContext().getResource());
            }
            return;
         }
      }
   }

   preProcessXml(root);
    //主要看这，这是真正对标签进行转化的
   parseBeanDefinitions(root, this.delegate);
   postProcessXml(root);

   this.delegate = parent;
}
```

##### parseBeanDefinitions(root, this.delegate)方法

```java
protected void parseBeanDefinitions(Element root, BeanDefinitionParserDelegate delegate) {
   //如果是<import />、<alias />、<bean /> 和 <beans />标签，走这里
   if (delegate.isDefaultNamespace(root)) {
      NodeList nl = root.getChildNodes();
      for (int i = 0; i < nl.getLength(); i++) {
         Node node = nl.item(i);
         if (node instanceof Element) {
            Element ele = (Element) node;
            if (delegate.isDefaultNamespace(ele)) {
               parseDefaultElement(ele, delegate);
            }
            else {
               delegate.parseCustomElement(ele);
            }
         }
      }
   }
    //其他标签走这里
   else {
      delegate.parseCustomElement(root);
   }
}
```

反正接下来就是将解析到的标签转化为BeanDefinition对象，然后将对象注册到ApplicationContext的Map中，就不详细说了，等有时间再写一个相关的解析(逃)

弄完之后，bean的解析完成了，也已经注册到BeanFactory（也就是ApplicationContext中的beanFactory属性，也就是DefaultListableBeanFactory对象）中了，那接下来就是对bean进行实例化了，这下又回到refresh方法中了，再贴一下：

```java
@Override
public void refresh() throws BeansException, IllegalStateException {
   synchronized (this.startupShutdownMonitor) {
      // Prepare this context for refreshing.
      prepareRefresh();

      // Tell the subclass to refresh the internal bean factory.
      ConfigurableListableBeanFactory beanFactory = obtainFreshBeanFactory();

      // Prepare the bean factory for use in this context.
      prepareBeanFactory(beanFactory);

      try {
         // Allows post-processing of the bean factory in context subclasses.
         postProcessBeanFactory(beanFactory);

         // Invoke factory processors registered as beans in the context.
         invokeBeanFactoryPostProcessors(beanFactory);

         // Register bean processors that intercept bean creation.
         registerBeanPostProcessors(beanFactory);

         // Initialize message source for this context.
         initMessageSource();

         // Initialize event multicaster for this context.
         initApplicationEventMulticaster();

         // Initialize other special beans in specific context subclasses.
         onRefresh();

         // Check for listener beans and register them.
         registerListeners();

         // Instantiate all remaining (non-lazy-init) singletons.
         finishBeanFactoryInitialization(beanFactory);

         // Last step: publish corresponding event.
         finishRefresh();
      }

      catch (BeansException ex) {
         if (logger.isWarnEnabled()) {
            logger.warn("Exception encountered during context initialization - " +
                  "cancelling refresh attempt: " + ex);
         }

         // Destroy already created singletons to avoid dangling resources.
         destroyBeans();

         // Reset 'active' flag.
         cancelRefresh(ex);

         // Propagate exception to caller.
         throw ex;
      }

      finally {
         // Reset common introspection caches in Spring's core, since we
         // might not ever need metadata for singleton beans anymore...
         resetCommonCaches();
      }
   }
}
```

## finishBeanFactoryInitialization方法

refresh下一个要讲的方法就是finishBeanFactoryInitialization(beanFactory)了，这一步会初始化所有除了设置了懒加载的所有bean。初始化过程就是，如果没有使用动态代理，那么会使用反射来创建，如果使用了动态代理，那么久创建对应的Proxy代理类，实例化之后的bean会存进ApplicationContex里的beanFactory中，通过一个ConcurrentHashMap进行存储。

那这一步做完，整个ApplicationContext就算是初始化完成了，康一康整个ApplicationContext长什么样：

![在这里插入图片描述](https://img-blog.csdnimg.cn/20200229014714582.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L1NDVVRKQVk=,size_16,color_FFFFFF,t_70)

beanFactory下面还有：

![在这里插入图片描述](https://img-blog.csdnimg.cn/20200229014723935.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L1NDVVRKQVk=,size_16,color_FFFFFF,t_70)

可以看到，ApplicationContext实际上是ClassPathXmlApplicationContext对象，而其中有一个beanFactory属性，为DefaultListableBeanFactory对象，这个对象里，有一个ConcurrentHashMap，里面的key值为bean的名字，value为对应的实例对象。而后面的getBean()方法实际上就是从这里取的。

# 获取bean对象

## getBean方法

也就是调用ClassPathXmlApplicationContext的getBean方法():

这是AbstractApplicationContext类中的方法，看一下结构图：

![在这里插入图片描述](https://img-blog.csdnimg.cn/20200229014736793.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L1NDVVRKQVk=,size_16,color_FFFFFF,t_70)

可以看到，ClassPathXmlApplicationContext是AbstractApplicationContext的子类，那直接看这个方法吧：

```java
public Object getBean(String name) throws BeansException {
    //这一步啥也没做==
   assertBeanFactoryActive();
   return getBeanFactory().getBean(name);
}
```

### getBeanFactory方法

```java
public final ConfigurableListableBeanFactory getBeanFactory() {
   synchronized (this.beanFactoryMonitor) {
      if (this.beanFactory == null) {
         throw new IllegalStateException("BeanFactory not initialized or already closed - " +
               "call 'refresh' before accessing beans via the ApplicationContext");
      }
       //返回了this.beanFactory，不就是DefaultListableBeanFactory实例么？
      return this.beanFactory;
   }
}
```

![在这里插入图片描述](https://img-blog.csdnimg.cn/20200229014750621.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L1NDVVRKQVk=,size_16,color_FFFFFF,t_70)

从这里也可以看出来

### getBean方法

```java
public Object getBean(String name) throws BeansException {
   return doGetBean(name, null, null, false);
}
```

调用了doGetBean

#### doGetBean方法

```java
protected <T> T doGetBean(final String name, @Nullable final Class<T> requiredType,
      @Nullable final Object[] args, boolean typeCheckOnly) throws BeansException {

   final String beanName = transformedBeanName(name);
   Object bean;

   // 调用了这个方法
   Object sharedInstance = getSingleton(beanName);
   .....
}
```

调用了getSingleton，进去看看

##### getSingleton方法

最终会到这里

```java
protected Object getSingleton(String beanName, boolean allowEarlyReference) {
    //主要是这里！
   Object singletonObject = this.singletonObjects.get(beanName);
   if (singletonObject == null && isSingletonCurrentlyInCreation(beanName)) {
      synchronized (this.singletonObjects) {
         singletonObject = this.earlySingletonObjects.get(beanName);
         if (singletonObject == null && allowEarlyReference) {
            ObjectFactory<?> singletonFactory = this.singletonFactories.get(beanName);
            if (singletonFactory != null) {
               singletonObject = singletonFactory.getObject();
               this.earlySingletonObjects.put(beanName, singletonObject);
               this.singletonFactories.remove(beanName);
            }
         }
      }
   }
   return singletonObject;
}
```

看看this.singletonObjects是什么：

![在这里插入图片描述](https://img-blog.csdnimg.cn/20200229014807869.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L1NDVVRKQVk=,size_16,color_FFFFFF,t_70)

害！可不就是DefaultListableBeanFactory中这个ConcurrentHashMap嘛，至此就拿到实例了

那执行流程粗略就是这样了，有一些细节上的操作没有仔细看，但是总体方向还是清楚的

那此时面试官问，Spring IOC的执行流程是啥呀？

A：执行的第一件事，就是要加载ApplicationContext对象，这个ApplicationContext是一个接口，并且是BeanFactory的子接口，BeanFactory就是存放与bean相关数据的一个容器，所以说初始化ApplicationContext就是初始化BeanFactory，与Bean相关的操作都是基于ApplicationContext来执行的，或者准确来说并不是由ApplicationContext亲自执行的，它内部还有一个beanFactory属性，这个beanFactory最终会被赋值为一个DefaultListableBeanFactory对象，那其实所有的bean操作都是基于这个属性完成的。

在创建好BeanFactory后，之后就要会对配置文件进行解析，每一个bean标签都会解析成一个BeanDefinition对象，然后注册到BeanFactory中，注册的方式是基于一个ConcurrentHashMap，key值为bean标签的名，value就是这个BeanDefinition对象。

注册完了之后，接下来就是对bean进行实例化了，实例化的过程分两种情况，如果对应的类使用了动态代理，那么就要实例化它对应的Proxy代理类，若没有使用动态代理，则通过反射进行创建。创建好的实例同样是注册在beanFactory中的一个ConcurrentHashMap中。至此ApplicationContext的创建就算是完成了。

后续调用getBean方法，其实就是从ApplicationContext的beanFactory属性中的ConcurrentHashMap进行获取。

这就是IOC执行的一个流程。
