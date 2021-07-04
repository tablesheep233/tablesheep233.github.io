---
layout:     post
title:      "Spring Bean Lifecycle"
subtitle:   " \"Spring Bean Lifecycle笔记\""
author:     "tablesheep"
date:       2021-07-02 21:10:00
header-style: text
catalog: true
tags:
    - Java
    - Spring
---

> Spring Bean 生命周期记录
>
> version：Spring Framework  5.3.x



## Overview

### BeanDefinition 

`BeanDefinition`（[see](https://github.com/spring-projects/spring-framework/blob/main/spring-beans/src/main/java/org/springframework/beans/factory/config/BeanDefinition.java)）封装了Bean的配置信息（提供一系列Getter、Setter方法），包括Class、Scope、FactoryBean信息、构造器参数、是否懒加载、初始化方法、销毁方法等，Spring是根据`BeanDefinition`的信息创建Bean。

https://docs.spring.io/spring-framework/docs/current/reference/html/core.html#beans-definition

### BeanFactory

`Spring IoC`底层容器，`Spring IoC`的基础。有很多的子类（大体上可以根据命名看出功能），基本上都用`DefaultListableBeanFactory`。（[see](https://github.com/spring-projects/spring-framework/blob/main/spring-beans/src/main/java/org/springframework/beans/factory/support/DefaultListableBeanFactory.java)）

https://docs.spring.io/spring-framework/docs/current/reference/html/core.html#beans-beanfactory

### ApplicationContext

更高级的`Ioc`容器，内部其实还是使用了`BeanFactory`做Bean的管理，只不过`ApplicationContext`提供了更多功能（比如`BeanFactoryPostProcessor`（`BeanFactoryPostProcessor`在`BeanFactory`级别是没有的）&`BeanPostProcessor`自动注入、Spring 消息、Spring 事件等）（[see](https://github.com/spring-projects/spring-framework/blob/main/spring-context/src/main/java/org/springframework/context/ApplicationContext.java)）

https://docs.spring.io/spring-framework/docs/current/reference/html/core.html#context-introduction-ctx-vs-beanfactory



### BeanPostProcessor

Bean回调接口。基于它和一系列子类实现了生命周期的各种操作。根据注释，非代理类通常通过`postProcessBeforeInitialization`回调，而代理类则通过`postProcessAfterInitialization`（`BeanFactory`创建的Bean只会调这个方法）。另外`BeanPostProcessor`的顺序很重要，它仅支持`PriorityOrdered`、`Ordered`接口排序，不支持`@Order`注解。

```java
/**
 * Factory hook that allows for custom modification of new bean instances &mdash;
 * for example, checking for marker interfaces or wrapping beans with proxies.
 *
 * <p>Typically, post-processors that populate beans via marker interfaces
 * or the like will implement {@link #postProcessBeforeInitialization},
 * while post-processors that wrap beans with proxies will normally
 * implement {@link #postProcessAfterInitialization}.
 *
 * <h3>Registration</h3>
 * <p>An {@code ApplicationContext} can autodetect {@code BeanPostProcessor} beans
 * in its bean definitions and apply those post-processors to any beans subsequently
 * created. A plain {@code BeanFactory} allows for programmatic registration of
 * post-processors, applying them to all beans created through the bean factory.
 *
 * <h3>Ordering</h3>
 * <p>{@code BeanPostProcessor} beans that are autodetected in an
 * {@code ApplicationContext} will be ordered according to
 * {@link org.springframework.core.PriorityOrdered} and
 * {@link org.springframework.core.Ordered} semantics. In contrast,
 * {@code BeanPostProcessor} beans that are registered programmatically with a
 * {@code BeanFactory} will be applied in the order of registration; any ordering
 * semantics expressed through implementing the
 * {@code PriorityOrdered} or {@code Ordered} interface will be ignored for
 * programmatically registered post-processors. Furthermore, the
 * {@link org.springframework.core.annotation.Order @Order} annotation is not
 * taken into account for {@code BeanPostProcessor} beans.
 *
 * @author Juergen Hoeller
 * @author Sam Brannen
 * @since 10.10.2003
 * @see InstantiationAwareBeanPostProcessor
 * @see DestructionAwareBeanPostProcessor
 * @see ConfigurableBeanFactory#addBeanPostProcessor
 * @see BeanFactoryPostProcessor
 */
public interface BeanPostProcessor {

	/**
	 * Apply this {@code BeanPostProcessor} to the given new bean instance <i>before</i> any bean
	 * initialization callbacks (like InitializingBean's {@code afterPropertiesSet}
	 * or a custom init-method). The bean will already be populated with property values.
	 * The returned bean instance may be a wrapper around the original.
	 * <p>The default implementation returns the given {@code bean} as-is.
	 * @param bean the new bean instance
	 * @param beanName the name of the bean
	 * @return the bean instance to use, either the original or a wrapped one;
	 * if {@code null}, no subsequent BeanPostProcessors will be invoked
	 * @throws org.springframework.beans.BeansException in case of errors
	 * @see org.springframework.beans.factory.InitializingBean#afterPropertiesSet
	 */
    //初始化前回调方法
	@Nullable
	default Object postProcessBeforeInitialization(Object bean, String beanName) throws BeansException {
		return bean;
	}

	/**
	 * Apply this {@code BeanPostProcessor} to the given new bean instance <i>after</i> any bean
	 * initialization callbacks (like InitializingBean's {@code afterPropertiesSet}
	 * or a custom init-method). The bean will already be populated with property values.
	 * The returned bean instance may be a wrapper around the original.
	 * <p>In case of a FactoryBean, this callback will be invoked for both the FactoryBean
	 * instance and the objects created by the FactoryBean (as of Spring 2.0). The
	 * post-processor can decide whether to apply to either the FactoryBean or created
	 * objects or both through corresponding {@code bean instanceof FactoryBean} checks.
	 * <p>This callback will also be invoked after a short-circuiting triggered by a
	 * {@link InstantiationAwareBeanPostProcessor#postProcessBeforeInstantiation} method,
	 * in contrast to all other {@code BeanPostProcessor} callbacks.
	 * <p>The default implementation returns the given {@code bean} as-is.
	 * @param bean the new bean instance
	 * @param beanName the name of the bean
	 * @return the bean instance to use, either the original or a wrapped one;
	 * if {@code null}, no subsequent BeanPostProcessors will be invoked
	 * @throws org.springframework.beans.BeansException in case of errors
	 * @see org.springframework.beans.factory.InitializingBean#afterPropertiesSet
	 * @see org.springframework.beans.factory.FactoryBean
	 */
    //初始化后回调方法
	@Nullable
	default Object postProcessAfterInitialization(Object bean, String beanName) throws BeansException {
		return bean;
	}

}
```

### InstantiationAwareBeanPostProcessor

在`BeanPostProcessor`基础上增加了几个方法，以名字而言应该是跟初始化有关。很多类都基于它实现，比如`@Autowired`、`@Value`注解能力的`AutowiredAnnotationBeanPostProcessor`，提供`JSR-250`注解能力的`CommonAnnotationBeanPostProcessor`等。

```java
public interface InstantiationAwareBeanPostProcessor extends BeanPostProcessor {

//实例化前回调方法
   @Nullable
   default Object postProcessBeforeInstantiation(Class<?> beanClass, String beanName) throws BeansException {
      return null;
   }

//实例化后回调方法
   default boolean postProcessAfterInstantiation(Object bean, String beanName) throws BeansException {
      return true;
   }

//PS:个人感觉以下两个方法跟实例化没啥关系跟初始化有点关系，不过此类又是继承BeanPostProcessor，其实还是有点关系，只不过名字感觉怪怪的。（仁者见仁智者见智吧。）
//用于Bean的属性填充，返回null时跳过填充
   @Nullable
   default PropertyValues postProcessProperties(PropertyValues pvs, Object bean, String beanName)
         throws BeansException {

      return null;
   }

//用于Bean的属性填充，返回null时跳过填充（虽然标记为过时，但Spring其实会在postProcessProperties返回null时调用，只有当这两个方法都为null时才跳过属性填充（未来应该会删除））
	@Deprecated
	@Nullable
	default PropertyValues postProcessPropertyValues(
			PropertyValues pvs, PropertyDescriptor[] pds, Object bean, String beanName) throws BeansException {

		return pvs;
	}

}
```



### BeanWrapper

`BeanWrapper`是Spring底层JavaBeans基础的中央接口（基于javabeans规范，包装了Bean对象，使用java.beans API进行属性相关操作），通常用于数据绑定和`BeanFactory`中。

https://docs.spring.io/spring-framework/docs/current/reference/html/core.html#beans-beans



## Lifecycle

以`ApplicationContext`作为`IoC`容器记录一下`Bean`的生命周期。之所以不用`BeanFactory`是因为有部分环节只在`ApplicationContext`为容器才能体现（比如`CommonAnnotationBeanPostProcessor`、`ApplicationContext`级别的`Aware`接口等），而且大部分项目都是使用`ApplicationContext`，很少直接使用`BeanFactory`。不过虽说使用了`ApplicationContext`，但其实大部分过程都是在`DefaultListableBeanFactory`中。

>  使用下面类跟踪，具体调用链可以从`AbstractBeanFactory#getBean(String name)`方法进入

```java
import org.springframework.beans.BeansException;
import org.springframework.beans.PropertyValue;
import org.springframework.beans.PropertyValues;
import org.springframework.beans.factory.*;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.beans.factory.annotation.Qualifier;
import org.springframework.beans.factory.config.InstantiationAwareBeanPostProcessor;
import org.springframework.beans.factory.support.MergedBeanDefinitionPostProcessor;
import org.springframework.beans.factory.support.RootBeanDefinition;
import org.springframework.context.ApplicationContext;
import org.springframework.context.ApplicationContextAware;
import org.springframework.context.annotation.AnnotationConfigApplicationContext;
import org.springframework.context.annotation.Bean;
import org.springframework.core.Ordered;
import org.springframework.core.PriorityOrdered;

import javax.annotation.PostConstruct;
import javax.annotation.PreDestroy;
import javax.annotation.Resource;
import java.beans.PropertyDescriptor;
import java.util.function.Supplier;

public class LifecycleMain {

    private static int COUNT = 1;

    public static void main(String[] args) {
        AnnotationConfigApplicationContext applicationContext = new AnnotationConfigApplicationContext();
        applicationContext.register(LifecycleMain.class);
//        applicationContext.getBeanFactory().addBeanPostProcessor(new CustomBeanPostProcessor()); //若通过直接注册优先级最高,使用不当会引起一些注解注入失效等问题
//        applicationContext.registerBean(BeanDomain.class, BeanDomain::new, (bd) -> bd.setInitMethodName("initMethod"));
        applicationContext.refresh();
        BeanDomain beanDomain = applicationContext.getBean(BeanDomain.class);
        String[] beanDefinitionNames = applicationContext.getBeanDefinitionNames();
        printlnFormatMessage(beanDomain.stringFiled);
        applicationContext.close();
    }

    @Bean
    public CustomBeanPostProcessor customInstantiationAwareBeanPostProcessor() {
        return new CustomBeanPostProcessor();
    }


    public static class CustomBeanPostProcessor implements InstantiationAwareBeanPostProcessor,
            MergedBeanDefinitionPostProcessor, //若不实现它，则属性注入以及@PostConstruct会失效，当然注意实现的方法的返回也可以规避这些问题
            PriorityOrdered {

        @Override
        public Object postProcessBeforeInstantiation(Class<?> beanClass, String beanName) throws BeansException {
            if (isBeanDomain(beanClass)) {
                printlnFormatMessage("InstantiationAwareBeanPostProcessor#postProcessBeforeInstantiation");
//                return null; //返回null会终止实例化并进入初始化后阶段
            }
            return InstantiationAwareBeanPostProcessor.super.postProcessBeforeInstantiation(beanClass, beanName);
        }

//        @Override
        public void postProcessMergedBeanDefinition(RootBeanDefinition beanDefinition, Class<?> beanType, String beanName) {
            if (isBeanDomain(beanType)) {
                printlnFormatMessage("MergedBeanDefinitionPostProcessor#postProcessMergedBeanDefinition");
            }
        }

        @Override
        public PropertyValues postProcessProperties(PropertyValues pvs, Object bean, String beanName) throws BeansException {
            if (isBeanDomain(bean.getClass())) {
                printlnFormatMessage("InstantiationAwareBeanPostProcessor#postProcessProperties");
                return null; //返回null终止属性注入，若此InstantiationAwareBeanPostProcessor优先级较高，会出现一些注解注入失效问题
            }
            return pvs;
        }

        @Override
        public PropertyValues postProcessPropertyValues(PropertyValues pvs, PropertyDescriptor[] pds, Object bean, String beanName) throws BeansException {
            if (isBeanDomain(bean.getClass())) {
                return null; //返回null终止属性注入，若此InstantiationAwareBeanPostProcessor优先级较高，会出现一些注解注入失效问题
            }
            return pvs;
        }

        @Override
        public Object postProcessBeforeInitialization(Object bean, String beanName) throws BeansException {
            if (isBeanDomain(bean.getClass())) {
                printlnFormatMessage("BeanPostProcessor#postProcessBeforeInitialization");
                return null; //返回null 后续postProcessBeforeInitialization终止
            }
            return InstantiationAwareBeanPostProcessor.super.postProcessBeforeInitialization(bean, beanName);
        }

        @Override
        public Object postProcessAfterInitialization(Object bean, String beanName) throws BeansException {
            if (isBeanDomain(bean.getClass())) {
                printlnFormatMessage("BeanPostProcessor#postProcessAfterInitialization");
                return null; //返回null 后续postProcessAfterInitialization终止
            }
            return InstantiationAwareBeanPostProcessor.super.postProcessAfterInitialization(bean, beanName);
        }

        @Override
        public boolean postProcessAfterInstantiation(Object bean, String beanName) throws BeansException {
            if (isBeanDomain(bean.getClass())) {
                printlnFormatMessage("InstantiationAwareBeanPostProcessor#postProcessAfterInstantiation");
            }
            return InstantiationAwareBeanPostProcessor.super.postProcessAfterInstantiation(bean, beanName);
        }



        @Override
        public int getOrder() {
            return Ordered.LOWEST_PRECEDENCE;
        }
    }

    @Bean
    public String stringFiled() {
        return "stringFiled";
    }


    @Bean(initMethod = "customInitMethod", destroyMethod = "customDestroyMethod")
//    @Scope(SCOPE_PROTOTYPE) //不会有完成阶段以及销毁阶段
//    @Lazy //不会有完成阶段
    public BeanDomain beanDomain() {
        return new BeanDomain();
    }

    public static class BeanDomain implements InitializingBean, DisposableBean,
            BeanFactoryAware, ApplicationContextAware, SmartInitializingSingleton {

        //@Resource
        @Autowired
        @Qualifier("stringFiled")
        private String stringFiled = "lueluelue";

        public BeanDomain() {
        }

        @PostConstruct
        public void initPostConstruct() {
            printlnFormatMessage("PostConstruct");
        }

        @PreDestroy
        public void preDestroy() {
            printlnFormatMessage("preDestroy");
        }

        @Override
        public void afterPropertiesSet() throws Exception {
            printlnFormatMessage("InitializingBean#afterPropertiesSet");
        }

        @Override
        public void destroy() throws Exception {
            printlnFormatMessage("DisposableBean#destroy");
        }

        public void customInitMethod() {
            printlnFormatMessage("customInitMethod");
        }

        public void customDestroyMethod() {
            printlnFormatMessage("customDestroyMethod");
        }

        @Override
        public void setBeanFactory(BeanFactory beanFactory) throws BeansException {
            printlnFormatMessage("BeanFactoryAware#setBeanFactory");
        }

        @Override
        public void setApplicationContext(ApplicationContext applicationContext) throws BeansException {
            printlnFormatMessage("ApplicationContextAware#setApplicationContext");
        }

        @Override
        public void afterSingletonsInstantiated() {
            printlnFormatMessage("SmartInitializingSingleton#afterSingletonsInstantiated");
        }
    }

//    @Bean
//    public BeanDomainFactoryBean beanDomainFactoryBean() {
//        return new BeanDomainFactoryBean();
//    }
//    
//    //通过FactoryBean创建的Bean只会调用BeanPostProcessor#postProcessAfterInitialization
//    public static class BeanDomainFactoryBean implements FactoryBean<BeanDomain> {
//
//        @Override
//        public BeanDomain getObject() throws Exception {
//            return new BeanDomain();
//        }
//
//        @Override
//        public Class<?> getObjectType() {
//            return BeanDomain.class;
//        }
//    }

    private static boolean isBeanDomain(Class clazz) {
        return BeanDomain.class.equals(clazz);
    }

    private static void printlnFormatMessage(String message) {
        System.out.println(String.format("%s. %s", COUNT++, message));
    }
}
```



### BeanDefinition 注册阶段

方法：`BeanDefinitionRegistry#registerBeanDefinition`（具体实现看`DefaultListableBeanFactory`）

将`BeanDefinition`信息注册到`Ioc`容器中，供后面Bean创建使用。（至于`BeanDefinition`信息的来源，主要是Spring内置、注解、XML、配置文件等）



### BeanDefinition 合并阶段

方法：`AbstractBeanFactory#getMergedLocalBeanDefinition`（先从缓存获取，没有才去调`getMergedBeanDefinition`） 

在Bean创建前会经过merge，将子Bean与父Bean合并，最终所有的`BeanDefinition`都会转换成`RootBeanDefinition`。（通过createBean调用，而createBean的调用有几种情况，如果是一些内置的组件（比如内置的`BeanFactoryPostProcessor`），会由`ApplicationContext`在`invokeBeanFactoryPostProcessors`时调用，而如果是我们普通的Bean（**非懒加载、单例**）会由`AbstractApplicationContext#finishBeanFactoryInitialization`通过`DefaultListableBeanFactory.preInstantiateSingletons`调用）



### Bean 实例化前阶段

方法：`AbstractAutowireCapableBeanFactory#resolveBeforeInstantiation`

会调用所有的`InstantiationAwareBeanPostProcessor#postProcessBeforeInstantiation`方法，当返回的对象不为null时，会终止Bean的创建并直接返回（可以用于生成代理对象）。同时在触发终止Bean创建后，会进入Bean初始化后阶段，即调用`AbstractAutowireCapableBeanFactory#applyBeanPostProcessorsAfterInitialization`方法。



### Bean 实例化阶段

方法：`AbstractAutowireCapableBeanFactory#createBeanInstance`

根据`BeanDefinition`中的信息创建Bean实例，最终会返回`BeanWrapper`。（此时创建的对象并未填充属性）

优先级分别是Supplier方式（`obtainFromSupplier`）、FactoryMethod方式（`instantiateUsingFactoryMethod`，`@Bean`是根据这个方式）、匹配构造器方式（`autowireConstructor`）、无参构造器方式（`instantiateBean`）

构造器方式具体的实例化是根据`InstantiationStrategy`完成的，默认使用`CglibSubclassingInstantiationStrategy`

```java
/**
 * Simple object instantiation strategy for use in a BeanFactory.
 *
 * <p>Does not support Method Injection, although it provides hooks for subclasses
 * to override to add Method Injection support, for example by overriding methods.
 *
 */
public class SimpleInstantiationStrategy implements InstantiationStrategy {
......
}


/**
 * Default object instantiation strategy for use in BeanFactories.
 *
 * <p>Uses CGLIB to generate subclasses dynamically if methods need to be
 * overridden by the container to implement <em>Method Injection</em>.
 * 提供了方法注入的支持
 */
public class CglibSubclassingInstantiationStrategy extends SimpleInstantiationStrategy {
......
}
```



### BeanDefinition 修改阶段

方法：`AbstractAutowireCapableBeanFactory#applyMergedBeanDefinitionPostProcessors`

通过`MergedBeanDefinitionPostProcessor#postProcessMergedBeanDefinition`方法，允许对`RootBeanDefinition`进行修改。

例如：在`CommonAnnotationBeanPostProcessor`中会获取`@Resource`、`@PostConstruct`、`@PreDestroy`属性或方法信息并写入到`BeanDefinition`中，类似的在`AutowiredAnnotationBeanPostProcessor`中会获取`@Autowired`、`@Value`、`@Inject`属性信息并写入到`BeanDefinition`中（`InjectionMetadata#checkConfigMembers` -> `BeanDefinition#registerExternallyManagedConfigMember`）。



### Bean 属性填充阶段

方法：`AbstractAutowireCapableBeanFactory#populateBean`

大致流程如下：

1. 实例化后处理

   方法：`InstantiationAwareBeanPostProcessor#postProcessAfterInstantiation`

   调用所有`InstantiationAwareBeanPostProcessor#postProcessAfterInstantiation`，让InstantiationAwareBeanPostProcessors 有能力在Bean属性注入前修改Bean状态。返回true代表要进行属性填充，false则不需要。

   

2. postProcessProperties（真不知道怎么翻译，直译怪怪的...）

   方法：`InstantiationAwareBeanPostProcessor#postProcessProperties`和`InstantiationAwareBeanPostProcessor#postProcessPropertyValues`（过时）

   处理所有的`postProcessProperties`和`postProcessPropertyValues`方法。

   先调用一个`InstantiationAwareBeanPostProcessor`的`postProcessProperties`方法，如果返回为null，会调用过时的`postProcessPropertyValues`方法（为了兼容一些旧版本吧，可能在未来会删除这个方法）。当有一个`InstantiationAwareBeanPostProcessor`的处理结果为null时，会停止属性填充，所以`InstantiationAwareBeanPostProcessor`的顺序很重要，这里有坑。

   为什么说有坑呢，因为`@Resource`、`@Autowired`、`@Value`、`@Inject`等注解都是在这一步进行填充，如果我们自定义的`InstantiationAwareBeanPostProcessor`先于`CommonAnnotationBeanPostProcessor`（`@Resource`支持）、`AutowiredAnnotationBeanPostProcessor`（`@Autowired`、`@Value`、`@Inject`支持）返回null，那么它们的属性填充就会失败了。

   **重点类** ：`CommonAnnotationBeanPostProcessor`、`AutowiredAnnotationBeanPostProcessor`

   以`AutowiredAnnotationBeanPostProcessor`为例子

   ```java
   @Override
   public PropertyValues postProcessProperties(PropertyValues pvs, Object bean, String beanName) {
   //获取所有的@Autowired、@Value、@Inject InjectionMetadata信息
      InjectionMetadata metadata = findAutowiringMetadata(beanName, bean.getClass(), pvs);
      try {
      //根据InjectionMetadata进行注入
         metadata.inject(bean, beanName, pvs);
      }
      catch (BeanCreationException ex) {
         throw ex;
      }
      catch (Throwable ex) {
         throw new BeanCreationException(beanName, "Injection of autowired dependencies failed", ex);
      }
      return pvs;
   }
   ```

   

3. 属性填充

   方法：`AbstractAutowireCapableBeanFactory#applyPropertyValues`

   将`PropertyValues`填充到Bean中，XML方式更加直观。



### Bean 初始化阶段

方法：`AbstractAutowireCapableBeanFactory#initializeBean`

也包含了很多方法的调用，大致如下：

1. `BeanFactory`级别的`Aware`接口回调

   方法：`AbstractAutowireCapableBeanFactory#invokeAwareMethods`

   ```java
   private void invokeAwareMethods(String beanName, Object bean) {
      if (bean instanceof Aware) {
         if (bean instanceof BeanNameAware) {
            ((BeanNameAware) bean).setBeanName(beanName);
         }
         if (bean instanceof BeanClassLoaderAware) {
            ClassLoader bcl = getBeanClassLoader();
            if (bcl != null) {
               ((BeanClassLoaderAware) bean).setBeanClassLoader(bcl);
            }
         }
         if (bean instanceof BeanFactoryAware) {
            ((BeanFactoryAware) bean).setBeanFactory(AbstractAutowireCapableBeanFactory.this);
         }
      }
   }
   ```

   

2. 初始化前回调

   方法：`AbstractAutowireCapableBeanFactory#applyBeanPostProcessorsBeforeInitialization`

   循环调用所有的`BeanPostProcessor#postProcessBeforeInitialization`方法，当返回为null停止调用。

   `ApplicationContext`级别的`Aware`接口就是在这个阶段实现的，具体见`ApplicationContextAwareProcessor`（还要`Servlet`级别的`ServletContextAwareProcessor`）。

   此外还有`CommonAnnotationBeanPostProcessor`支持的`@PostConstruct`初始化方法也是在这里通过`postProcessBeforeInitialization`方法进行的。

   

3. 初始化

   方法：`AbstractAutowireCapableBeanFactory#invokeInitMethods`

   包含了`InitializingBean`的调用以及自定义初始化方法的调用。

   1. 先判断Bean是否`InitializingBean`，若是则会调用`afterPropertiesSet`初始化。
   2. 调用`BeanDefinition`中配置的初始化方法（@Bean配置的，或者XML中配置的）。



### Bean 初始化后阶段

方法：`AbstractAutowireCapableBeanFactory#applyBeanPostProcessorsAfterInitialization`

与初始前回调类似，循环调用`BeanPostProcessor#postProcessAfterInitialization`，当返回为null停止调用。在实例化前阶段终止实例化的Bean也会进入此阶段。



### Bean 销毁回调注册

方法：`AbstractBeanFactory#registerDisposableBeanIfNecessary`

针对**非`Prototype`**的（因为`Prototype`作用域的Bean创建完可以说就不归Spring管理了，每次创建都是新的），以及需要销毁的`Bean`会调用此方法注册一个`DisposableBean`（使用`DisposableBeanAdapter`适配）供`IoC`容器关闭是销毁调用。

关于需要销毁的Bean主要有：

- 实现了`DisposableBean`的类
- 实现了`AutoCloseable`的类
- `BeanDefinition`指定了销毁方法（`@Bean`配置、XML配置等）
- 存在`DestructionAwareBeanPostProcessor`并且该Bean适用于`DestructionAwareBeanPostProcessor`（举个例子就是`BeanFactory`的`BeanPostProcessor`列表存在`CommonAnnotationBeanPostProcessor`，并且我们的Bean使用了`@PreDestroy`指定销毁方法）

具体看`AbstractBeanFactory#requiresDestruction`方法。



### Bean 完成阶段

方法：`SmartInitializingSingleton#afterSingletonsInstantiated`

在`DefaultListableBeanFactory.preInstantiateSingletons`完成所有单例、非懒加载的的创建后，会回调这些Bean中所有实现了`SmartInitializingSingleton`接口的方法`afterSingletonsInstantiated`。通过它的方法调用能保证所有的单例Bean已经创建，在很多时候非常有用。



### Bean 销毁阶段

方法：`DisposableBeanAdapter#destroy`

有很多入口，主要由`ApplicationContext`容器关闭时通过`DefaultListableBeanFactory#destroySingletons`调用。

销毁流程：

1. 销毁前阶段

   方法：`DestructionAwareBeanPostProcessor#postProcessBeforeDestruction`

   主要就是`CommonAnnotationBeanPostProcessor`的`@PreDestroy`处理逻辑。

2. `DisposableBean`销毁

   方法：`DisposableBean#destroy`

3. 自定义销毁方法

   调用注册时的销毁方法（`@Bean`配置、XML配置等）



## 问题记录

### InstantiationAwareBeanPostProcessor顺序问题

**问题复现：**在`ApplicationContext`中，实现`InstantiationAwareBeanPostProcessor#postProcessProperties`（和`postProcessPropertyValues`）方法且返回null，若没有同时实现`MergedBeanDefinitionPostProcessor`，会导致`@Resource`、`@Autowired`、`@Value`、`@Inject`失效，就算实现了`Order`、`PriorityOrdered`接口，指定最低优先级也没用。按道理Spring会根据Order顺序执行，可是当这种情况下会发现自定义的`InstantiationAwareBeanPostProcessor`会在`CommonAnnotationBeanPostProcessor`和`AutowiredAnnotationBeanPostProcessor`之前。

**原因分析：**`ApplicationContext#registerBeanPostProcessors`是通过`PostProcessorRegistrationDelegate.registerBeanPostProcessors`注册`BeanPostProcessor`的，一开始会按照`PriorityOrdered`、`Order`接口进行顺序注册，但在最后会把内部的`BeanPostProcessor`重新注册一遍，而判断的方法就是是否`MergedBeanDefinitionPostProcessor`，而`BeanPostProcessor`注册时是先把原有的删除在添加到队列尾部。

```java
public static void registerBeanPostProcessors(
      ConfigurableListableBeanFactory beanFactory, AbstractApplicationContext applicationContext) {
......

   // Separate between BeanPostProcessors that implement PriorityOrdered,
   // Ordered, and the rest.
   List<BeanPostProcessor> priorityOrderedPostProcessors = new ArrayList<>();
   List<BeanPostProcessor> internalPostProcessors = new ArrayList<>();
   List<String> orderedPostProcessorNames = new ArrayList<>();
   List<String> nonOrderedPostProcessorNames = new ArrayList<>();
   for (String ppName : postProcessorNames) {
      if (beanFactory.isTypeMatch(ppName, PriorityOrdered.class)) {
         BeanPostProcessor pp = beanFactory.getBean(ppName, BeanPostProcessor.class);
         priorityOrderedPostProcessors.add(pp);
         //内部BeanPostProcessor判断
         if (pp instanceof MergedBeanDefinitionPostProcessor) {
            internalPostProcessors.add(pp);
         }
      }
      else if (beanFactory.isTypeMatch(ppName, Ordered.class)) {
         orderedPostProcessorNames.add(ppName);
      }
      else {
         nonOrderedPostProcessorNames.add(ppName);
      }
   }

   // First, register the BeanPostProcessors that implement PriorityOrdered.
   sortPostProcessors(priorityOrderedPostProcessors, beanFactory);
   registerBeanPostProcessors(beanFactory, priorityOrderedPostProcessors);

   // Next, register the BeanPostProcessors that implement Ordered.
   List<BeanPostProcessor> orderedPostProcessors = new ArrayList<>(orderedPostProcessorNames.size());
   for (String ppName : orderedPostProcessorNames) {
      BeanPostProcessor pp = beanFactory.getBean(ppName, BeanPostProcessor.class);
      orderedPostProcessors.add(pp);
      if (pp instanceof MergedBeanDefinitionPostProcessor) {
         internalPostProcessors.add(pp);
      }
   }
   sortPostProcessors(orderedPostProcessors, beanFactory);
   registerBeanPostProcessors(beanFactory, orderedPostProcessors);

   // Now, register all regular BeanPostProcessors.
   List<BeanPostProcessor> nonOrderedPostProcessors = new ArrayList<>(nonOrderedPostProcessorNames.size());
   for (String ppName : nonOrderedPostProcessorNames) {
      BeanPostProcessor pp = beanFactory.getBean(ppName, BeanPostProcessor.class);
      nonOrderedPostProcessors.add(pp);
      if (pp instanceof MergedBeanDefinitionPostProcessor) {
         internalPostProcessors.add(pp);
      }
   }
   registerBeanPostProcessors(beanFactory, nonOrderedPostProcessors);

   // Finally, re-register all internal BeanPostProcessors.
   //会把内部的BeanPostProcessors重新注册
   sortPostProcessors(internalPostProcessors, beanFactory);
   registerBeanPostProcessors(beanFactory, internalPostProcessors);
......
}
```



### To be continued...






References
----------

- <https://docs.spring.io/spring-framework/docs/current/reference/html/core.html#spring-core>
- 小马哥讲 Spring 核心编程思想

