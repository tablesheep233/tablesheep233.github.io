---
layout:     post
title:      "AopContext使用问题分析"
subtitle:   " \"AopContext使用问题分析\""
author:     "tablesheep"
date:       2021-11-11 21:22:00
header-style: text
catalog: true
tags:
- Java
- Spring
- Spring AOP
- 源码
---

> version：Spring Framework 5.3.x



在使用`Spring`开发时，偶尔会遇到在本类方法中调用自身被`AOP`增强的方法，这时候直接调用并不会达到预期的效果，大致的原因是在`Spring`生成的代理对象中调用自身方法并不会调用被增强的方法，而是会直接调用被代理类原本的方法（具体可通过`Spring`  `JDK`动态代理和`CGLIB`代理生成的字节码去看）。



那具体有什么方法能实现我们想要的效果呢？(🙄)

- 通过依赖查找获取`Bean`（简单粗暴）
- 不要写这种代码（废话，不过官方也是这么讲的，不过个人也是建议这样）
- 通过配置`exposeProxy=true`，使用`AopContext`获取当前代理对象调用（Spring官方提供的一种解决方案，但是在一些情况下有坑）

## AopContext使用

now，先来说说`AopContext`的用法，以及它碰到`@Async`时会遇到的问题。

**使用**

只需要通过`EnableAspectJAutoProxy#exposeProxy`开启，使用`AopContext#currentProxy`就可以获取到当前的代理对象

```java
//配置
@EnableAspectJAutoProxy(exposeProxy = true)

//使用
((YourClass)AopContext.currentProxy()).yourMethod();
```

**原理**

`AopContext`就是使用`ThreadLocal`在同一个线程中使用，而关键一点就是在何时调用`setCurrentProxy`将代理对象设置进去。

```java
public final class AopContext {
   private static final ThreadLocal<Object> currentProxy = new NamedThreadLocal<>("Current AOP proxy");
......

   @Nullable
   static Object setCurrentProxy(@Nullable Object proxy) {
      Object old = currentProxy.get();
      if (proxy != null) {
         currentProxy.set(proxy);
      }
      else {
         currentProxy.remove();
      }
      return old;
   }

}
```

而具体设置的时机便是在AOP方法调用时进行的，以`JdkDynamicAopProxy`为例，在`invoke`中会根据`advised.exposeProxy`（`AdvisedSupport`）的配置进行设置，而这个配置便是在`JdkDynamicAopProxy`构造时进行设置的。

```java
@Override
@Nullable
public Object invoke(Object proxy, Method method, Object[] args) throws Throwable {
   Object oldProxy = null;
   boolean setProxyContext = false;
......

   try {
     ......

      Object retVal;

      if (this.advised.exposeProxy) {
         // Make invocation available if necessary.
         oldProxy = AopContext.setCurrentProxy(proxy);
         setProxyContext = true;
      }
......

      // Get the interception chain for this method.
      List<Object> chain = this.advised.getInterceptorsAndDynamicInterceptionAdvice(method, targetClass);

      // Check whether we have any advice. If we don't, we can fallback on direct
      // reflective invocation of the target, and avoid creating a MethodInvocation.
      if (chain.isEmpty()) {
         // We can skip creating a MethodInvocation: just invoke the target directly
         // Note that the final invoker must be an InvokerInterceptor so we know it does
         // nothing but a reflective operation on the target, and no hot swapping or fancy proxying.
         Object[] argsToUse = AopProxyUtils.adaptArgumentsIfNecessary(method, args);
         retVal = AopUtils.invokeJoinpointUsingReflection(target, method, argsToUse);
      }
      else {
         // We need to create a method invocation...
         MethodInvocation invocation =
               new ReflectiveMethodInvocation(proxy, target, method, args, targetClass, chain);
         // Proceed to the joinpoint through the interceptor chain.
         retVal = invocation.proceed();
      }

      // Massage return value if necessary.
      Class<?> returnType = method.getReturnType();
      if (retVal != null && retVal == target &&
            returnType != Object.class && returnType.isInstance(proxy) &&
            !RawTargetAccess.class.isAssignableFrom(method.getDeclaringClass())) {
         // Special case: it returned "this" and the return type of the method
         // is type-compatible. Note that we can't help if the target sets
         // a reference to itself in another returned object.
         retVal = proxy;
      }
      else if (retVal == null && returnType != Void.TYPE && returnType.isPrimitive()) {
         throw new AopInvocationException(
               "Null return value from advice does not match primitive return type for: " + method);
      }
      return retVal;
   }
   finally {
......
      if (setProxyContext) {
         // Restore old proxy.
         AopContext.setCurrentProxy(oldProxy);
      }
   }
}
```





## 问题

### 问题一

```java
//配置
@EnableAspectJAutoProxy(exposeProxy = true)
```



```java
@Service
public class ServiceImpl implements IService {

    @Override
    public void handle() {
        //do something ......
        ((IService)AopContext.currentProxy()).asyncHandle();
    }

    @Async
    @Override
    public void asyncHandle() {
        //do something ......
    }
}
```

当`Bean`中只存在`@Async`时，在别的方法使用`AopContext`试图调用`@Async`增强的方法时会出现如下错误：

```
Cannot find current proxy: Set 'exposeProxy' property on Advised to 'true' to make it available, and ensure that AopContext.currentProxy() is invoked in the same thread as the AOP invocation context.
```



#### 原因分析

根据上面对`AopContext`的分析，结合`@Async`的创建（使用`AsyncAnnotationBeanPostProcessor`）过程，在`AbstractAdvisingBeanPostProcessor#postProcessAfterInitialization`打断点，会使用`prepareProxyFactory`创建`ProxyFactory`，而`ProxyFactory`的配置来自自身，而`exposeProxy`是为`false`。

```java
public abstract class AbstractAdvisingBeanPostProcessor extends ProxyProcessorSupport implements BeanPostProcessor {
......

   @Override
   public Object postProcessAfterInitialization(Object bean, String beanName) {
      if (this.advisor == null || bean instanceof AopInfrastructureBean) {
         // Ignore AOP infrastructure such as scoped proxies.
         return bean;
      }

      if (bean instanceof Advised) {
          //若对象已经被代理过了，则直接将advisor添加进去
         Advised advised = (Advised) bean;
         if (!advised.isFrozen() && isEligible(AopUtils.getTargetClass(bean))) {
            // Add our local Advisor to the existing proxy's Advisor chain...
            if (this.beforeExistingAdvisors) {
               advised.addAdvisor(0, this.advisor);
            }
            else {
               advised.addAdvisor(this.advisor);
            }
            return bean;
         }
      }

      if (isEligible(bean, beanName)) {
          //创建ProxyFactory
         ProxyFactory proxyFactory = prepareProxyFactory(bean, beanName);
         if (!proxyFactory.isProxyTargetClass()) {
            evaluateProxyInterfaces(bean.getClass(), proxyFactory);
         }
         proxyFactory.addAdvisor(this.advisor);
         customizeProxyFactory(proxyFactory);

         // Use original ClassLoader if bean class not locally loaded in overriding class loader
         ClassLoader classLoader = getProxyClassLoader();
         if (classLoader instanceof SmartClassLoader && classLoader != bean.getClass().getClassLoader()) {
            classLoader = ((SmartClassLoader) classLoader).getOriginalClassLoader();
         }
          //创建代理对象
         return proxyFactory.getProxy(classLoader);
      }

      // No proxy needed.
      return bean;
   }

......
   protected ProxyFactory prepareProxyFactory(Object bean, String beanName) {
   //创建ProxyFactory，将本身配置赋予创建的ProxyFactory
      ProxyFactory proxyFactory = new ProxyFactory();
      proxyFactory.copyFrom(this);
      proxyFactory.setTarget(bean);
      return proxyFactory;
   }
......
}
```

而我们已经配置了`@EnableAspectJAutoProxy(exposeProxy = true)`，这个时候就要看看这个配置是如何工作的了。在`@EnableAspectJAutoProxy`中引入了`AspectJAutoProxyRegistrar`，这个类实现了`ImportBeanDefinitionRegistrar`，会在`BeanDefinition`注册时进行工作。

```java
class AspectJAutoProxyRegistrar implements ImportBeanDefinitionRegistrar {

   /**
    * Register, escalate, and configure the AspectJ auto proxy creator based on the value
    * of the @{@link EnableAspectJAutoProxy#proxyTargetClass()} attribute on the importing
    * {@code @Configuration} class.
    */
   @Override
   public void registerBeanDefinitions(
         AnnotationMetadata importingClassMetadata, BeanDefinitionRegistry registry) {

      AopConfigUtils.registerAspectJAnnotationAutoProxyCreatorIfNecessary(registry);

      AnnotationAttributes enableAspectJAutoProxy =
            AnnotationConfigUtils.attributesFor(importingClassMetadata, EnableAspectJAutoProxy.class);
      if (enableAspectJAutoProxy != null) {
         if (enableAspectJAutoProxy.getBoolean("proxyTargetClass")) {
            AopConfigUtils.forceAutoProxyCreatorToUseClassProxying(registry);
         }
         if (enableAspectJAutoProxy.getBoolean("exposeProxy")) {
             //若exposeProxy为true，调用AopConfigUtils#forceAutoProxyCreatorToExposeProxy进行配置
            AopConfigUtils.forceAutoProxyCreatorToExposeProxy(registry);
         }
      }
   }

}
```

```java
public static final String AUTO_PROXY_CREATOR_BEAN_NAME =
			"org.springframework.aop.config.internalAutoProxyCreator";

public static void forceAutoProxyCreatorToExposeProxy(BeanDefinitionRegistry registry) {
   if (registry.containsBeanDefinition(AUTO_PROXY_CREATOR_BEAN_NAME)) {
       //可以看出，它配置的只有名为AUTO_PROXY_CREATOR_BEAN_NAME的BeanDefinition
      BeanDefinition definition = registry.getBeanDefinition(AUTO_PROXY_CREATOR_BEAN_NAME);
      definition.getPropertyValues().add("exposeProxy", Boolean.TRUE);
   }
}
```

由于配置的只有名为`AUTO_PROXY_CREATOR_BEAN_NAME`变量的`BeanDefinition`，而`AsyncAnnotationBeanPostProcessor`配置时并不是这个名字，所以`@EnableAspectJAutoProxy(exposeProxy = true)`并不会对它生效。

```java
@Configuration(proxyBeanMethods = false)
@Role(BeanDefinition.ROLE_INFRASTRUCTURE)
public class ProxyAsyncConfiguration extends AbstractAsyncConfiguration {

   @Bean(name = TaskManagementConfigUtils.ASYNC_ANNOTATION_PROCESSOR_BEAN_NAME)
   @Role(BeanDefinition.ROLE_INFRASTRUCTURE)
   public AsyncAnnotationBeanPostProcessor asyncAdvisor() {
      Assert.notNull(this.enableAsync, "@EnableAsync annotation metadata was not injected");
      AsyncAnnotationBeanPostProcessor bpp = new AsyncAnnotationBeanPostProcessor();
      bpp.configure(this.executor, this.exceptionHandler);
      Class<? extends Annotation> customAsyncAnnotation = this.enableAsync.getClass("annotation");
      if (customAsyncAnnotation != AnnotationUtils.getDefaultValue(EnableAsync.class, "annotation")) {
         bpp.setAsyncAnnotationType(customAsyncAnnotation);
      }
      bpp.setProxyTargetClass(this.enableAsync.getBoolean("proxyTargetClass"));
      bpp.setOrder(this.enableAsync.<Integer>getNumber("order"));
      return bpp;
   }

}
```



#### 解决

参照`AopConfigUtils#orceAutoProxyCreatorToExposeProxy`只需要实现`BeanFactoryPostProcessor`，将`AsyncAnnotationBeanPostProcessor`的`BeanDefinition`中添加`exposeProxy=true`即可。

```java
@Configuration
public class AsyncBeanFactoryPostProcessor implements BeanFactoryPostProcessor {
    @Override
    public void postProcessBeanFactory(ConfigurableListableBeanFactory beanFactory) throws BeansException {
        if (beanFactory.containsBeanDefinition(TaskManagementConfigUtils.ASYNC_ANNOTATION_PROCESSOR_BEAN_NAME)) {
            BeanDefinition beanDefinition = beanFactory.getBeanDefinition(TaskManagementConfigUtils.ASYNC_ANNOTATION_PROCESSOR_BEAN_NAME);
            beanDefinition.getPropertyValues().add("exposeProxy", true);
        }
    }
}
```

#### 总结

`@EnableAspectJAutoProxy(exposeProxy = true)` 只适用于名为`AUTO_PROXY_CREATOR_BEAN_NAME`的`BeanDefinition`。




### 问题二

```java
//配置
@EnableAspectJAutoProxy(exposeProxy = true)
```



```java
@Service
public class ServiceImpl implements IService {

    @Override
    public void handle() {
        //do something ......
        ((IService)AopContext.currentProxy()).asyncHandle();
    }

    @Async
    @Override
    public void asyncHandle() {
        //do something ......
    }
    
    @Transactional(rollbackFor = Exception.class)
   // @Cacheable(cacheNames="cachename", key="#a0.id")
    @Override
    public void save(A a) {
        //do something ......
    }
}
```

当`Bean`中存在`@Async`还有别的`Spring AOP`注解（`@Transactional`、`@Cacheable......`）时，在别的方法使用`AopContext`试图调用`@Async`增强的方法时却不会出现问题，诡异，吗？



#### 现象分析

这次依旧在`AbstractAdvisingBeanPostProcessor#postProcessAfterInitialization`打断点，会发现`exposeProxy`是依旧为`false`，但是与之前不同的是并不会进入创建代理对象的阶段，而是会直接将`advisor`添加进`Advised`中，这表示这个`bean`已经被代理过了，再看`Advised`中的配置，会发现`exposeProxy`却是`true`，再通过看`Advised`中的`advisors`，可以看出这个`Bean`是被事务增强（或者缓存增强）过的`Bean`。

```java
public abstract class AbstractAdvisingBeanPostProcessor extends ProxyProcessorSupport implements BeanPostProcessor {
......

   @Override
   public Object postProcessAfterInitialization(Object bean, String beanName) {
......
      if (bean instanceof Advised) {
          //若对象已经被代理过了，则直接将advisor添加进去
         Advised advised = (Advised) bean;
         if (!advised.isFrozen() && isEligible(AopUtils.getTargetClass(bean))) {
            // Add our local Advisor to the existing proxy's Advisor chain...
            if (this.beforeExistingAdvisors) {
               advised.addAdvisor(0, this.advisor);
            }
            else {
               advised.addAdvisor(this.advisor);
            }
            return bean;
         }
      }
......
   }
}
```

出现这个现象的原因是由于在`AbstractAdvisingBeanPostProcessor`中会直接向已代理的`Bean`中直接添加`Advisor`，同时`AsyncAnnotationBeanPostProcessor`会位于`BeanPostProcessors`最后（`@EnableAsync`中指定了order，默认在最后），而`AopConfigUtils`中注入的名为`AUTO_PROXY_CREATOR_BEAN_NAME`的`BeanPostProcessor`会在最前面。

```java
public abstract class AopConfigUtils {

......

   @Nullable
   private static BeanDefinition registerOrEscalateApcAsRequired(
         Class<?> cls, BeanDefinitionRegistry registry, @Nullable Object source) {

      Assert.notNull(registry, "BeanDefinitionRegistry must not be null");

      if (registry.containsBeanDefinition(AUTO_PROXY_CREATOR_BEAN_NAME)) {
         BeanDefinition apcDefinition = registry.getBeanDefinition(AUTO_PROXY_CREATOR_BEAN_NAME);
         if (!cls.getName().equals(apcDefinition.getBeanClassName())) {
            int currentPriority = findPriorityForClass(apcDefinition.getBeanClassName());
            int requiredPriority = findPriorityForClass(cls);
            if (currentPriority < requiredPriority) {
               apcDefinition.setBeanClassName(cls.getName());
            }
         }
         return null;
      }

      RootBeanDefinition beanDefinition = new RootBeanDefinition(cls);
      beanDefinition.setSource(source);
    //指定order顺序，默认最高优先级
      beanDefinition.getPropertyValues().add("order", Ordered.HIGHEST_PRECEDENCE);
      beanDefinition.setRole(BeanDefinition.ROLE_INFRASTRUCTURE);
      registry.registerBeanDefinition(AUTO_PROXY_CREATOR_BEAN_NAME, beanDefinition);
      return beanDefinition;
   }
......

}
```



#### 总结

在创建`Bean`时，如果Bean中有`@Cacheable、@Transactional、@Async`注解，会先经由名为`AUTO_PROXY_CREATOR_BEAN_NAME`（自动代理创建器）的`BeanPostProcessor`代理，接着才会经过`AsyncBeanFactoryPostProcessor`的代理往代理对象中添加异步的`Advisor`。而此时的代理对象的配置是来自`AUTO_PROXY_CREATOR_BEAN_NAME`代理而来，故如果配置了`exposeProxy=true`会发现没有问题。
