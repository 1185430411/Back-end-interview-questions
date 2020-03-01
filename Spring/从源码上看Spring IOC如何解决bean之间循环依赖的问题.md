@[toc]
我们来探讨一下Spring是如何解决循环依赖问题的。

## 什么是循环依赖

先看一个示例图吧：

![在这里插入图片描述](https://img-blog.csdnimg.cn/20200301210108669.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L1NDVVRKQVk=,size_16,color_FFFFFF,t_70)

好像很抽象的样子，没事，直接看代码就很清晰了：

```java
class A{
    private B b;
}
class B{
    private C c;
}
class C{
    private A a;
}
```

大概就是这么一种依赖结构，对应的配置文件如下：

```xml
<bean id="beanA" class="com.jay.bean.A">
    <property name="b" ref="beanB"/>
</bean>
<bean id="beanB" class="com.jay.bean.B">
    <property name="c" ref="beanC"/>
</bean>
<bean id="beanC" class="com.jay.bean.C>
    <property name="a" ref="beanA"/>
</bean>
```

也就是这样了。

我这样模拟了一个：

```java
public class StudentServiceImpl implements StudentService, BeanFactoryPostProcessor {

    int id;

    String name;

    Den den;

    public StudentServiceImpl() {
    }
}



//
public class Den {
    StudentService studentService;

    public StudentService getStudentService() {
        return studentService;
    }

    public Den() {
    }
}


public class StudentTest {
    public static void main(String[] args){
        ApplicationContext applicationContext=new ClassPathXmlApplicationContext("classpath:applicationContext.xml");
        StudentService studentService=(StudentService)applicationContext.getBean("studentService");
        studentService.study();
    }
}

```



我们都知道Spring IOC在创建完实例之后，需要对实例进行属性填充，而此时如果发现了循环依赖的关系，会怎么办呢？

IOC使用是一种 允许未完成初始化的对象提前发布 的一种思想来解决的，具体的实现呢，就是使用了三级缓存

## 三级缓存

```java
/** Cache of singleton objects: bean name --> bean instance */
private final Map<String, Object> singletonObjects = new ConcurrentHashMap<>(256);

/** Cache of singleton factories: bean name --> ObjectFactory */
private final Map<String, ObjectFactory<?>> singletonFactories = new HashMap<>(16);

/** Cache of early singleton objects: bean name --> bean instance */
private final Map<String, Object> earlySingletonObjects = new HashMap<>(16);
```

IOC中的三级缓存其实就是三个Map，并且这三个都是存放在DefaultListableBeanFactory这个工厂类对象中，DefaultListableBeanFactory应该不需要多解释了。

- singletonObjects：一级缓存是一个ConcurrentHashMap，存放着对象实例，我们调用getBean方法其实都是从这个map中直接进行获取的
- earlySingletonObjects：二级缓存是一个HashMap，也是存放着对象实例，但和一级缓存的区别就是，二级缓存存放的是尚未经过初始化的实例，也就是还未进行属性赋值的实例
- singletonFactories：三级缓存也是一个HashMap，存放着对应bean的工厂类对象

## 流程

上面的代码写了三个对象的相互依赖，这里为了简便起见，我们只讨论两个对象的的依赖情况。

经过一系列流程，此时容器完成了初始化，配置文件也被解析成了beanDefinition对象，并且注册到了beanFactory中，此时要做的，就是按照beanDefinition，开始进行bean的实例化了

对一个bean的实例化，从getBean方法开始，而getBean方法，会调用 doGetBean():

```java
protected <T> T doGetBean(final String name, @Nullable final Class<T> requiredType,
      @Nullable final Object[] args, boolean typeCheckOnly) throws BeansException {

   final String beanName = transformedBeanName(name);
   Object bean;

   // 调用getSingleton方法，这里最终会去一级缓存中获取，自然也只能获取到空值了。
   Object sharedInstance = getSingleton(beanName);
   if (sharedInstance != null && args == null) {
      if (logger.isDebugEnabled()) {
         if (isSingletonCurrentlyInCreation(beanName)) {
            logger.debug("Returning eagerly cached instance of singleton bean '" + beanName +
                  "' that is not fully initialized yet - a consequence of a circular reference");
         }
         else {
            logger.debug("Returning cached instance of singleton bean '" + beanName + "'");
         }
      }
      bean = getObjectForBeanInstance(sharedInstance, name, beanName, null);
   }
   //获取到空值，那么就要进行实例化了
   else {
      ...

      try {
         ...

         // 如果是单例
         if (mbd.isSingleton()) {
            sharedInstance = getSingleton(beanName, () -> {
               try {
                  //这里才是真正实例化的地方
                  return createBean(beanName, mbd, args);
               }
               catch (BeansException ex) {
                  // Explicitly remove instance from singleton cache: It might have been put there
                  // eagerly by the creation process, to allow for circular reference resolution.
                  // Also remove any beans that received a temporary reference to the bean.
                  destroySingleton(beanName);
                  throw ex;
               }
            });
            bean = getObjectForBeanInstance(sharedInstance, name, beanName, mbd);
         }

         else if (mbd.isPrototype()) {
            // It's a prototype -> create a new instance.
            Object prototypeInstance = null;
            try {
               beforePrototypeCreation(beanName);
               prototypeInstance = createBean(beanName, mbd, args);
            }
            finally {
               afterPrototypeCreation(beanName);
            }
            bean = getObjectForBeanInstance(prototypeInstance, name, beanName, mbd);
         }

         ...
      }
      catch (BeansException ex) {
         cleanupAfterBeanCreationFailure(beanName);
         throw ex;
      }
   }

   ...
   return (T) bean;
}
```

那就看一下createBean()吧：

```java
@Override
protected Object createBean(String beanName, RootBeanDefinition mbd, @Nullable Object[] args)
      throws BeanCreationException {

   if (logger.isDebugEnabled()) {
      logger.debug("Creating instance of bean '" + beanName + "'");
   }
    //获取该bean的模板，也就是beanDefinition实例
   RootBeanDefinition mbdToUse = mbd;

   // Make sure bean class is actually resolved at this point, and
   // clone the bean definition in case of a dynamically resolved Class
   // which cannot be stored in the shared merged bean definition.
   Class<?> resolvedClass = resolveBeanClass(mbd, beanName);
   if (resolvedClass != null && !mbd.hasBeanClass() && mbd.getBeanClassName() != null) {
      mbdToUse = new RootBeanDefinition(mbd);
      mbdToUse.setBeanClass(resolvedClass);
   }

   // Prepare method overrides.
   try {
      mbdToUse.prepareMethodOverrides();
   }
   catch (BeanDefinitionValidationException ex) {
      throw new BeanDefinitionStoreException(mbdToUse.getResourceDescription(),
            beanName, "Validation of method overrides failed", ex);
   }

   try {
      // Give BeanPostProcessors a chance to return a proxy instead of the target bean instance.
      Object bean = resolveBeforeInstantiation(beanName, mbdToUse);
      if (bean != null) {
         return bean;
      }
   }
   catch (Throwable ex) {
      throw new BeanCreationException(mbdToUse.getResourceDescription(), beanName,
            "BeanPostProcessor before instantiation of bean failed", ex);
   }

   try {
       //重点是这里
      Object beanInstance = doCreateBean(beanName, mbdToUse, args);
      if (logger.isDebugEnabled()) {
         logger.debug("Finished creating instance of bean '" + beanName + "'");
      }
      return beanInstance;
   }
   catch (BeanCreationException ex) {
      // A previously detected exception with proper bean creation context already...
      throw ex;
   }
   catch (ImplicitlyAppearedSingletonException ex) {
      // An IllegalStateException to be communicated up to DefaultSingletonBeanRegistry...
      throw ex;
   }
   catch (Throwable ex) {
      throw new BeanCreationException(
            mbdToUse.getResourceDescription(), beanName, "Unexpected exception during bean creation", ex);
   }
}
```

看看doGetBean里：

```java
protected Object doCreateBean(final String beanName, final RootBeanDefinition mbd, final @Nullable Object[] args)
      throws BeanCreationException {

   // Instantiate the bean.
   BeanWrapper instanceWrapper = null;
   if (mbd.isSingleton()) {
      instanceWrapper = this.factoryBeanInstanceCache.remove(beanName);
   }
   if (instanceWrapper == null) {
       //这一步才是真正地，将bean对象实例化了，并且存放在instanceWrapper这个包装类对象中
      instanceWrapper = createBeanInstance(beanName, mbd, args);
   }
    //从包装类对象中获取到bean实例
   final Object bean = instanceWrapper.getWrappedInstance();    //1
   Class<?> beanType = instanceWrapper.getWrappedClass();
   if (beanType != NullBean.class) {
      mbd.resolvedTargetType = beanType;
   }

   // Allow post-processors to modify the merged bean definition.
   synchronized (mbd.postProcessingLock) {
      if (!mbd.postProcessed) {
         try {
            applyMergedBeanDefinitionPostProcessors(mbd, beanType, beanName);
         }
         catch (Throwable ex) {
            throw new BeanCreationException(mbd.getResourceDescription(), beanName,
                  "Post-processing of merged bean definition failed", ex);
         }
         mbd.postProcessed = true;
      }
   }

   // Eagerly cache singletons to be able to resolve circular references
   // even when triggered by lifecycle interfaces like BeanFactoryAware.
   boolean earlySingletonExposure = (mbd.isSingleton() && this.allowCircularReferences &&
         isSingletonCurrentlyInCreation(beanName));
   if (earlySingletonExposure) {
      if (logger.isDebugEnabled()) {
         logger.debug("Eagerly caching bean '" + beanName +
               "' to allow for resolving potential circular references");
      }
       //写入三级缓存，这是实现循环依赖的关键！
      addSingletonFactory(beanName, () -> getEarlyBeanReference(beanName, mbd, bean));
   }

   // Initialize the bean instance.
   Object exposedObject = bean;
   try {
       //进行初始化，也就是属性填充
      populateBean(beanName, mbd, instanceWrapper);
      exposedObject = initializeBean(beanName, exposedObject, mbd);
   }
   catch (Throwable ex) {
      if (ex instanceof BeanCreationException && beanName.equals(((BeanCreationException) ex).getBeanName())) {
         throw (BeanCreationException) ex;
      }
      else {
         throw new BeanCreationException(
               mbd.getResourceDescription(), beanName, "Initialization of bean failed", ex);
      }
   }

   ...

   ...

   return exposedObject;
}
```

现在bean实例已经完成了，按理来说我们可以直接返回了，但事实上，我们还需要写缓存呢，因为要解决依赖问题，所以接下来就看看addSingletonFactory方法里面做了什么：

```java
protected void addSingletonFactory(String beanName, ObjectFactory<?> singletonFactory) {
   Assert.notNull(singletonFactory, "Singleton factory must not be null");
   synchronized (this.singletonObjects) {
      if (!this.singletonObjects.containsKey(beanName)) {
         this.singletonFactories.put(beanName, singletonFactory);
         this.earlySingletonObjects.remove(beanName);
         this.registeredSingletons.add(beanName);
      }
   }
}
```

做了这么几件事：

1.写三级缓存。（看看三级缓存里面都有什么）：

![在这里插入图片描述](https://img-blog.csdnimg.cn/20200301210001965.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L1NDVVRKQVk=,size_16,color_FFFFFF,t_70)

看singletonFactories属性中，已经存入了一个工厂类对象，里面存储着一个StudentServiceImpl@2067，正是我们刚刚实例化完成，但还没有进行赋值的那个对象

2.从二级缓存中移除(当然二级缓存中目前什么也没有)

接下来就是要进行属性注入了，其中也要注入den属性，这里就发生循环依赖了，populateBean方法就不仔细看了，里面就是会循环bean实例里面的参数进行赋值，当循环到den属性时，发现是一个对象，那么就要进行den的初始化了。在一级二级三级缓存都没有发现den对象，那么就要对den进行初始化了，那den的初始化和StudentServiceImpl初始化类似，都是调用：

getBean  --- > doGetBean ---> getSingleton这样子 ，获取到den的实例（还没有初始化噢！），同样也要注册到三级缓存中。同样，也要进行赋值，在赋值时，因为den也依赖了studentServiceImpl，所以它同样要给studentServiceImpl赋值，同样要去一级二级三级缓存中查找，而这次不同的是，它会在三级缓存中找到studentServiceImpl对象对应的工厂类对象，于是就能获取到了studentServiceImpl对象，而这个studentServiceImpl对象正是我们刚刚创建的那个，获取到它之后，赋值给den，那么den的实例化就算是完成了，虽然den锁依赖的studentServiceImpl对象还没有完成初始化，也就是它内部的属性都是一个原始值，但是没关系，den对象完成了实例化就足够了
去一级二级三级缓存查找的过程如下：


```java
protected Object getSingleton(String beanName, boolean allowEarlyReference) {
    //去一级缓存中查找
   Object singletonObject = this.singletonObjects.get(beanName);
   if (singletonObject == null && isSingletonCurrentlyInCreation(beanName)) {
      synchronized (this.singletonObjects) {
          //若空，则取二级缓存中查找
         singletonObject = this.earlySingletonObjects.get(beanName);
         if (singletonObject == null && allowEarlyReference) {
             //若空，则去三级缓存中查找
            ObjectFactory<?> singletonFactory = this.singletonFactories.get(beanName);
            if (singletonFactory != null) {
                //singletonFactories已经有一个值了，在创建studentServiceImpl实例（未初始化）时就已经经过一次
                //addSingletonFactory操作了，将“早期引用”存放在这里了，所以这里直接获取到了
                //取出该“早期引用”，里面是一个studentServiceImpl实例，但属性都是原始值
               singletonObject = singletonFactory.getObject();
               this.earlySingletonObjects.put(beanName, singletonObject);
               this.singletonFactories.remove(beanName);
            }
         }
      }
   }
    //返回
   return singletonObject;
}
```

！

den初始化完成了，然后就是回到studentServiceImpl的赋值阶段了呗，den已经有了，直接赋值，ok，至此，studentServiceImpl对象的创建就已经完成了，后续就是将该对象存到一级缓存中，使得后续调用getBean可以直接在一级缓存中获取该对象。ok，那么这样的话，对象的创建就大功告成啦！循环依赖的问题也解决了。

最后再以一道面试题来总结下博客的内容吧~

**面试官：IOC是如何解决循环依赖的问题的呢？**

A：IOC解决循环依赖的思想就是：将初始化还未完成的对象提前发布。

对应的实现呢，就是ioc中的beanFactory维护了三个级别的缓存

- 一级缓存叫做singletonObjects，它是一个ConcurrentHashMap，它存的value就是bean对象实例了，而后续调用getBean方法其实也是从这个map里获取
- 二级缓存叫做earlySingletonObjects，它是一个HashMap，而它存的也是bean对象实例，但和一级缓存的区别就是，二级缓存存的实例还未进行初始化，也就是所有属性值都只是数据类型的初始值。
- 三级缓存叫做singletonFactories，它也是一个HashMap，存的是bean的工厂类。

假设有两个类是A,B是互相依赖的，那么首先要对A类进行实例化。实例化的入口就是getBean这个方法，对应的操作就是以该bean的beanDefinition对象作为模板，进行实例化，实例化之后，将该对象包装成工厂类，注册到三级缓存之中，而这一步就是实现循环依赖的一个关键所在了。进行了实例化之后，再对对象进行初始化，也就是属性赋值，当赋值到B对象，却发现一级二级三级缓存中都没有B对象的值时，此时就要去创建B对象，同样需要实例化，然后初始化的过程，那B对象的初始化过程也需要给A赋值，然后就去查找，最终在三级缓存中获取到了对象A，然后赋值给对象B，并且将三级缓存的数据添加到二级缓存后进行清楚。至此B对象就已经实例化完毕了，虽然B对象中的A对象只是进行实例化，还未进行初始化，但B对象能够成功实例化，就已经足够了。B对象实例化完之后，就可以赋值给A对象了，然后A对象完成初始化，注册在一级缓存中，那么整个过程就完成了。也解决了互相依赖的问题。
