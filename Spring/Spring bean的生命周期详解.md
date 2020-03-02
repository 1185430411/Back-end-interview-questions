@[toc]

# 生命周期流程图

![在这里插入图片描述](https://img-blog.csdnimg.cn/20200302164959270.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L1NDVVRKQVk=,size_16,color_FFFFFF,t_70)

图已经描述得比较详细了

# 代码实现

纸上得来终觉浅，绝知此事要躬行。

那就写代码来看看，执行结果是不是符合我们得预期：

StudentBean：

```java
package com.jay.service;

import org.springframework.beans.BeansException;
import org.springframework.beans.factory.*;

public class StudentBean implements InitializingBean, DisposableBean, BeanNameAware, BeanFactoryAware {
    int age;

    public StudentBean() {
        System.out.println("[bean构造方法]");
    }

    public int getAge() {
        return age;
    }

    public void setAge(int age) {
        System.out.println("[注入属性]：age");
        this.age = age;
    }

    public void myInit(){
        System.out.println("[init-metho]调用init-method属性配置的初始化方法");
    }

    public void myDestory(){
        System.out.println("[destroy-method]调用destroy-method属性配置的销毁方法");
    }

    @Override
    public void setBeanFactory(BeanFactory beanFactory) throws BeansException {
        System.out.println("[BeanFactoryAware接口的setBeanFactory方法，这里可以获取beanFactory，对里面的属性进行修改]");
    }

    @Override
    public void setBeanName(String name) {
        System.out.println("[BeanNameAware接口的setBeanName方法]");
    }

    @Override
    public void destroy() throws Exception {
        System.out.println("[DisposableBean接口]调用DisposableBean接口的destroy方法");
    }

    @Override
    public void afterPropertiesSet() throws Exception {
        System.out.println("[InitializingBean接口]调用InitializingBean接口的afterPropertiesSet方法");
    }
}
```

BeanFactoryPostProcessor：

```java
package com.jay.service;

import org.springframework.beans.BeansException;
import org.springframework.beans.factory.config.BeanDefinition;
import org.springframework.beans.factory.config.BeanFactoryPostProcessor;
import org.springframework.beans.factory.config.ConfigurableListableBeanFactory;


public class MyBeanFactoryPostProcessor implements BeanFactoryPostProcessor {

    public MyBeanFactoryPostProcessor() {
        System.out.println("【BeanFactoryPostProcessor接口】调用BeanFactoryPostProcessor实现类构造方法");
    }

    /**
     * 重写BeanFactoryPostProcessor接口的postProcessBeanFactory方法，可通过该方法对beanFactory进行设置
     */
    @Override
    public void postProcessBeanFactory(ConfigurableListableBeanFactory beanFactory)
            throws BeansException {
        System.out.println("【BeanFactoryPostProcessor接口】调用BeanFactoryPostProcessor接口的postProcessBeanFactory方法");
        BeanDefinition beanDefinition = beanFactory.getBeanDefinition("studentBean");
        beanDefinition.getPropertyValues().addPropertyValue("age", "21");
    }
}
```

InstantiationAwareBeanPostProcessor：

```java
package com.jay.service;

import org.springframework.beans.BeansException;
import org.springframework.beans.PropertyValues;
import org.springframework.beans.factory.config.InstantiationAwareBeanPostProcessorAdapter;

import java.beans.PropertyDescriptor;


public class MyInstantiationAwareBeanPostProcessor extends InstantiationAwareBeanPostProcessorAdapter {

    public MyInstantiationAwareBeanPostProcessor() {
        System.out.println("[InstantiationAwareBeanPostProcessor接口]调用InstantiationAwareBeanPostProcessor构造方法");
    }

    /**
     * 实例化Bean之前调用
     */
    @Override
    public Object postProcessBeforeInstantiation(Class beanClass, String beanName) throws BeansException {
        System.out.println("[InstantiationAwareBeanPostProcessor接口]调用InstantiationAwareBeanPostProcessor接口的postProcessBeforeInstantiation方法");
        return null;
    }

    /**
     * 实例化Bean之后调用
     */
    @Override
    public Object postProcessAfterInitialization(Object bean, String beanName) throws BeansException {
        System.out.println("[InstantiationAwareBeanPostProcessor接口]调用InstantiationAwareBeanPostProcessor接口的postProcessAfterInitialization方法");
        return bean;
    }

    /**
     * 设置某个属性时调用
     */
    @Override
    public PropertyValues postProcessPropertyValues(PropertyValues pvs, PropertyDescriptor[] pds, Object bean, String beanName)
            throws BeansException {
        System.out.println("[InstantiationAwareBeanPostProcessor接口]调用InstantiationAwareBeanPostProcessor接口的postProcessPropertyValues方法");
        return pvs;
    }
}
```

BeanPostProcessor：

```java
package com.jay.service;

import org.springframework.beans.BeansException;
import org.springframework.beans.factory.config.BeanPostProcessor;


public class MyBeanPostProcessor implements BeanPostProcessor {

    public MyBeanPostProcessor(){
        System.out.println("[BeanPostProcessor接口]调用BeanPostProcessor的构造方法");
    }

    @Override
    public Object postProcessBeforeInitialization(Object bean, String beanName) throws BeansException {
        System.out.println("[BeanPostProcessor接口]调用postProcessBeforeInitialization方法，这里可对"+beanName+"的属性进行更改。");
        return bean;
    }

    @Override
    public Object postProcessAfterInitialization(Object bean, String beanName) throws BeansException {
        System.out.println("[BeanPostProcessor接口]调用postProcessAfterInitialization方法，这里可对"+beanName+"的属性进行更改。");
        return bean;
    }
}
```

配置文件：

```xml
<bean id="studentBean" class="com.jay.service.StudentBean" init-method="myInit" destroy-method="myDestory">
    <property name="age" value="19"></property>
</bean>
<!--配置Bean的后置处理器-->
<bean id="beanPostProcessor" class="com.jay.service.MyBeanPostProcessor">
</bean>

<!--配置instantiationAwareBeanPostProcessor-->
<bean id="instantiationAwareBeanPostProcessor" class="com.jay.service.MyInstantiationAwareBeanPostProcessor">
</bean>

<!--配置BeanFactory的后置处理器-->
<bean id="beanFactoryPostProcessor" class="com.jay.service.MyBeanFactoryPostProcessor">
</bean>
```

测试：

```java
@Test
public void test(){
    ApplicationContext applicationContext=new ClassPathXmlApplicationContext("classpath:applicationContext.xml");
    StudentBean studentBean=(StudentBean)applicationContext.getBean("studentBean");
    ((ClassPathXmlApplicationContext) applicationContext).registerShutdownHook();
}
```

执行结果：

![在这里插入图片描述](https://img-blog.csdnimg.cn/20200302165013419.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L1NDVVRKQVk=,size_16,color_FFFFFF,t_70)

可以看到，执行结果还是符合我们刚才给出的流程图的~

（这玩意也太难记了uuu）
