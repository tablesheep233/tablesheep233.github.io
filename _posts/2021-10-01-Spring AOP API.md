---
layout:     post
title:      "Spring AOP API 阅读"
subtitle:   " \"Spring AOP API 阅读\""
author:     "tablesheep"
date:       2021-10-01 09:00:00
header-style: text
catalog: true
tags:
    - Java
    - Spring
    - Spring AOP
    - 源码
---

> Spring AOP API 粗略浏览
>
> 实在是太多了(((φ(◎ロ◎;)φ)))



## AOP基础

### Joinpoint

`Joinpoint` （[see](https://github.com/spring-projects/spring-framework/blob/main/spring-aop/src/main/java/org/aopalliance/intercept/Joinpoint.java)）接口描述了程序运行时的一个连接点，笼统一点就是一个方法、构造方法、字段，AOP的方法增强就是基于连接点上实现的。

```java
public interface Joinpoint {

   /**
    * 执行拦截器链（Advice链）
    * Proceed to the next interceptor in the chain.
    * <p>The implementation and the semantics of this method depends
    * on the actual joinpoint type (see the children interfaces).
    * @return see the children interfaces' proceed definition
    * @throws Throwable if the joinpoint throws an exception
    */
   @Nullable
   Object proceed() throws Throwable;

   /**
    * 获取当前连接点对象
    * Return the object that holds the current joinpoint's static part.
    * <p>For instance, the target object for an invocation.
    * @return the object (can be null if the accessible object is static)
    */
   @Nullable
   Object getThis();

   /**
    * 获取当前连接点描述信息（就是Method对象）
    * Return the static part of this joinpoint.
    * <p>The static part is an accessible object on which a chain of
    * interceptors are installed.
    */
   @Nonnull
   AccessibleObject getStaticPart();

}
```

在Spring中只有方法，主要实现类有`ReflectiveMethodInvocation`和`CglibMethodInvocation`，分别用于JDK动态代理和CGLIB动态代理。

![joinpoint](/img/post/aop/aop-api-joinpoint.png)

### Pointcut

`Pointcut`主要是用来确定哪些`Joinpoint`是需要切入的，主要是通过`ClassFilter`确认类，以及`MethodMatcher`匹配方法。

```java
/**
 * Core Spring pointcut abstraction.
 *
 * <p>A pointcut is composed of a {@link ClassFilter} and a {@link MethodMatcher}.
 * Both these basic terms and a Pointcut itself can be combined to build up combinations
 * (e.g. through {@link org.springframework.aop.support.ComposablePointcut}).
 *
 * @author Rod Johnson
 * @see ClassFilter
 * @see MethodMatcher
 * @see org.springframework.aop.support.Pointcuts
 * @see org.springframework.aop.support.ClassFilters
 * @see org.springframework.aop.support.MethodMatchers
 */
public interface Pointcut {

   /**
    * Return the ClassFilter for this pointcut.
    * @return the ClassFilter (never {@code null})
    */
   ClassFilter getClassFilter();

   /**
    * Return the MethodMatcher for this pointcut.
    * @return the MethodMatcher (never {@code null})
    */
   MethodMatcher getMethodMatcher();

   /**
    * Canonical Pointcut instance that always matches.
    */
   Pointcut TRUE = TruePointcut.INSTANCE;
}
```

具体实现有很多，比如：

- `StaticMethodMatcherPointcut` ，继承`MethodMatcher`，不关心运行时参数（需要匹配参数的方法默认返回true，同时`final`化）（`@Cacheable`系列注解匹配）
- `DynamicMethodMatcherPointcut`，继承`DynamicMethodMatcher`，可以根据运行时参数动态匹配
- `AnnotationMatchingPointcut` ，注解匹配（`@Async`、`@Validated`，`@Repository`）
- `ExpressionPointcut`，表达式匹配（主要是AspectJ表达式，`AspectJExpressionPointcut` ）
- `ComposablePointcut`，通过对`ClassFilter`、`MethodMatcher`各自交集、并集实现组合匹配（`Pointcuts`基于它实现）



### Advice(Interceptor)

`Advice`标记接口，可以理解为是一个操作（增强、通知）。捋一捋，首先在程序运行时有很多的方法在执行（`Joinpoint`），而我们通过这些方法的信息筛选满足条件的方法（`Pointcut`），再然后在这些方法执行的时候加入一些操作（`Advice`）。

`Advice`有很多子接口以及很多的实现，接口如图：

![advice](/img/post/aop/aop-api-advice.png)

按简单理解，大体上跟AspectJ差不多，有Around、Before、After、AfterReturn、Throws这几种不同时机的`Advice`，其中Around方式可以使用`MethodInterceptor`实现。同时Spring也整合了AspectJ上述类型的注解，如下：

![advice-aspectj](/img/post/aop/aop-api-advice-aspectj.png)

其中一半是依赖`MethodInterceptor`，事实上除了部分`Advice`，其他的实现都是基于`MethodInterceptor`，比如异步的`AsyncExecutionInterceptor`、缓存的`AsyncExecutionInterceptor`、事务的`TransactionInterceptor`等。`MethodInterceptor`只有一个`invoke`方法，参数是我们的连接点`MethodInvocation`，可以使用`Joinpoint#proceed`方法执行连接点原本操作，同时也可以在它前后做操作。

PS：实际上`MethodBeforeAdvice`、`AfterReturningAdvice`、`ThrowsAdvice`都会通过适配器，适配为`MethodInterceptor`执行，所以也几乎可以将`Advice`与`Interceptor`划等号，具体看[AdvisorAdapter](#advisoradapter)、[AdvisorAdapterRegistry](#advisoradapterregistry)。

```java
/**
 * Intercepts calls on an interface on its way to the target. These
 * are nested "on top" of the target.
 *
 * <p>The user should implement the {@link #invoke(MethodInvocation)}
 * method to modify the original behavior. E.g. the following class
 * implements a tracing interceptor (traces all the calls on the
 * intercepted method(s)):
 *
 * <pre class=code>
 * class TracingInterceptor implements MethodInterceptor {
 *   Object invoke(MethodInvocation i) throws Throwable {
 *     System.out.println("method "+i.getMethod()+" is called on "+
 *                        i.getThis()+" with args "+i.getArguments());
 *     Object ret=i.proceed();
 *     System.out.println("method "+i.getMethod()+" returns "+ret);
 *     return ret;
 *   }
 * }
 * </pre>
 *
 * @author Rod Johnson
 */
@FunctionalInterface
public interface MethodInterceptor extends Interceptor {

   /**
   //获取到连接点（方法），使我们可以在它前后做操作
    * Implement this method to perform extra treatments before and
    * after the invocation. Polite implementations would certainly
    * like to invoke {@link Joinpoint#proceed()}.
    * @param invocation the method invocation joinpoint
    * @return the result of the call to {@link Joinpoint#proceed()};
    * might be intercepted by the interceptorjava
    * @throws Throwable if the interceptors or the target object
    * throws an exception
    */
   @Nullable
   Object invoke(@Nonnull MethodInvocation invocation) throws Throwable;

}
```



### Advisor

`Advisor`是`Advice`容器类，主要是用来存放`Advice`以及用来确定连接点是否能使用该`Advice`（通过`Pointcut`）。

```java
/**
 * Base interface holding AOP <b>advice</b> (action to take at a joinpoint)
 * and a filter determining the applicability of the advice (such as
 * a pointcut). <i>This interface is not for use by Spring users, but to
 * allow for commonality in support for different types of advice.</i>
 *
 * <p>Spring AOP is based around <b>around advice</b> delivered via method
 * <b>interception</b>, compliant with the AOP Alliance interception API.
 * The Advisor interface allows support for different types of advice,
 * such as <b>before</b> and <b>after</b> advice, which need not be
 * implemented using interception.
 *
 * @author Rod Johnson
 * @author Juergen Hoeller
 */
public interface Advisor {
......
   /**
    * Return the advice part of this aspect. An advice may be an
    * interceptor, a before advice, a throws advice, etc.
    * @return the advice that should apply if the pointcut matches
    * @see org.aopalliance.intercept.MethodInterceptor
    * @see BeforeAdvice
    * @see ThrowsAdvice
    * @see AfterReturningAdvice
    */
   Advice getAdvice();

......
}
```

主要有两大子类型，分别是`IntroductionAdvisor`和`PointcutAdvisor`，如下。

![advisor](/img/post/aop/aop-api-advisor.png)

其中`IntroductionAdvisor`为`Introduction`引入类型，主要用于为类引入新的接口功能（比如为A类引入B接口，具体要结合`IntroductionInterceptor`使用），具体有

- `DefaultIntroductionAdvisor`，内置可以参考Spring的`ExposeBeanNameIntroduction`，它实现了`NamedBean`接口，使得代理对象有`getBeanName`的能力（可以看看`DelegatingIntroductionInterceptor`的invoke方法，基于是否引入接口的方法进行执行选择）
- `DeclareParentsAdvisor`，基于AspectJ的`@DeclareParents`

而`PointcutAdvisor`有很多的实现，挑几个看看

- `DefaultPointcutAdvisor`默认`PointcutAdvisor`实现

- `InstantiationModelAwarePointcutAdvisor`，平时使用的AspectJ注解在创建`Advisor`时都是使用它，唯一实现类`InstantiationModelAwarePointcutAdvisorImpl`
- `AbstractBeanFactoryPointcutAdvisor`，主要作用是在`Advice`为null时，可以通过`adviceBeanName`从容器获取`Advice`，缓存（`BeanFactoryCacheOperationSourceAdvisor`）、事务（`BeanFactoryTransactionAttributeSourceAdvisor`）使用它（不过好像`Advice`都不为null）
- `AsyncAnnotationAdvisor`，异步相关

PS：关于`IntroductionAdvisor`和`PointcutAdvisor`有几个区别

- `IntroductionAdvisor`只有一个`ClassFilter`，只能匹配到类，而`PointcutAdvisor`包含了`Pointcut`可以用于更细粒度的匹配（方法匹配）
- `IntroductionAdvisor`只能应用于`Introduction`类型的接口引入（其实也可以在`Interceptor`的invoke方法做操作，但这样违背了设计）
- 最重要的一点`IntroductionAdvisor`的`Advice`（`Interceptor`）需要实现引入的接口，而`PointcutAdvisor`中的`Advice`没有限制



#### AdvisorAdapter

`AdvisorAdapter`适配器，主要是将`Advisor`中的`Advice`适配为`MethodInterceptor`，具体看图：

![AdvisorAdapter](/img/post/aop/aop-api-AdvisorAdapter.png)



```java
class MethodBeforeAdviceAdapter implements AdvisorAdapter, Serializable {

   @Override
   public boolean supportsAdvice(Advice advice) {
      return (advice instanceof MethodBeforeAdvice);
   }

   @Override
   public MethodInterceptor getInterceptor(Advisor advisor) {
      MethodBeforeAdvice advice = (MethodBeforeAdvice) advisor.getAdvice();
      return new MethodBeforeAdviceInterceptor(advice);
   }

}
```



#### AdvisorAdapterRegistry

适配器注册表，唯一实现`DefaultAdvisorAdapterRegistry`

```java
public class DefaultAdvisorAdapterRegistry implements AdvisorAdapterRegistry, Serializable {

   private final List<AdvisorAdapter> adapters = new ArrayList<>(3);


   /**
    * Create a new DefaultAdvisorAdapterRegistry, registering well-known adapters.
    */
   public DefaultAdvisorAdapterRegistry() {
       //将MethodBeforeAdvice、AfterReturningAdvice、ThrowsAdvice的适配器注册
      registerAdvisorAdapter(new MethodBeforeAdviceAdapter());
      registerAdvisorAdapter(new AfterReturningAdviceAdapter());
      registerAdvisorAdapter(new ThrowsAdviceAdapter());
   }


  //对Advice进行包装，包装为DefaultPointcutAdvisor
   @Override
   public Advisor wrap(Object adviceObject) throws UnknownAdviceTypeException {
      if (adviceObject instanceof Advisor) {
         return (Advisor) adviceObject;
      }
      if (!(adviceObject instanceof Advice)) {
         throw new UnknownAdviceTypeException(adviceObject);
      }
      Advice advice = (Advice) adviceObject;
      if (advice instanceof MethodInterceptor) {
         // So well-known it doesn't even need an adapter.
         return new DefaultPointcutAdvisor(advice);
      }
      for (AdvisorAdapter adapter : this.adapters) {
          //将MethodBeforeAdvice、AfterReturningAdvice、ThrowsAdvice包装为DefaultPointcutAdvisor
         // Check that it is supported.
         if (adapter.supportsAdvice(advice)) {
            return new DefaultPointcutAdvisor(advice);
         }
      }
      throw new UnknownAdviceTypeException(advice);
   }

    //将Advisor中的Advice统一转换为MethodInterceptor
   @Override
   public MethodInterceptor[] getInterceptors(Advisor advisor) throws UnknownAdviceTypeException {
      List<MethodInterceptor> interceptors = new ArrayList<>(3);
      Advice advice = advisor.getAdvice();
      if (advice instanceof MethodInterceptor) {
         interceptors.add((MethodInterceptor) advice);
      }
      for (AdvisorAdapter adapter : this.adapters) {
          //将MethodBeforeAdvice、AfterReturningAdvice、ThrowsAdvice适配为对应的MethodInterceptor
         if (adapter.supportsAdvice(advice)) {
            interceptors.add(adapter.getInterceptor(advisor));
         }
      }
      if (interceptors.isEmpty()) {
         throw new UnknownAdviceTypeException(advisor.getAdvice());
      }
      return interceptors.toArray(new MethodInterceptor[0]);
   }

   @Override
   public void registerAdvisorAdapter(AdvisorAdapter adapter) {
      this.adapters.add(adapter);
   }

}
```



## 代理相关

### 代理对象

![AdvisorAdapter](/img/post/aop/aop-api-AopProxy.png)

#### AopProxy

代理对象抽象接口，提供了获取代理对象的方法`getProxy`和`getProxy(ClassLoader classLoader)`，具体实现有`JDK`和`CGLIB`的实现。

```java
public interface AopProxy {

   /**
    * Create a new proxy object.
    * <p>Uses the AopProxy's default class loader (if necessary for proxy creation):
    * usually, the thread context class loader.
    * @return the new proxy object (never {@code null})
    * @see Thread#getContextClassLoader()
    */
   Object getProxy();

   /**
    * Create a new proxy object.
    * <p>Uses the given class loader (if necessary for proxy creation).
    * {@code null} will simply be passed down and thus lead to the low-level
    * proxy facility's default, which is usually different from the default chosen
    * by the AopProxy implementation's {@link #getProxy()} method.
    * @param classLoader the class loader to create the proxy with
    * (or {@code null} for the low-level proxy facility's default)
    * @return the new proxy object (never {@code null})
    */
   Object getProxy(@Nullable ClassLoader classLoader);

}
```



### 代理对象工厂

#### AopProxyFactory

代理工厂能根据代理配置`AdvisedSupport`生成代理对象`AopProxy`，同时生成的对象要符合一系列规定（看注释）。

```java
/**
 * Interface to be implemented by factories that are able to create
 * AOP proxies based on {@link AdvisedSupport} configuration objects.
 *
 * <p>Proxies should observe the following contract:
 * <ul>
 * <li>They should implement all interfaces that the configuration
 * indicates should be proxied.
 * <li>They should implement the {@link Advised} interface.
 * <li>They should implement the equals method to compare proxied
 * interfaces, advice, and target.
 * <li>They should be serializable if all advisors and target
 * are serializable.
 * <li>They should be thread-safe if advisors and target
 * are thread-safe.
 * </ul>
 *
 * <p>Proxies may or may not allow advice changes to be made.
 * If they do not permit advice changes (for example, because
 * the configuration was frozen) a proxy should throw an
 * {@link AopConfigException} on an attempted advice change.
 *
 * @author Rod Johnson
 * @author Juergen Hoeller
 */
public interface AopProxyFactory {

   /**
    * Create an {@link AopProxy} for the given AOP configuration.
    * @param config the AOP configuration in the form of an
    * AdvisedSupport object
    * @return the corresponding AOP proxy
    * @throws AopConfigException if the configuration is invalid
    */
   AopProxy createAopProxy(AdvisedSupport config) throws AopConfigException;

}
```



#### DefaultAopProxyFactory

代理工厂默认实现，根据配置选择生成`JDK`还是`CGLIB`的代理对象。

```java
public class DefaultAopProxyFactory implements AopProxyFactory, Serializable {


   @Override
   public AopProxy createAopProxy(AdvisedSupport config) throws AopConfigException {
       //判断是否GraalVM镜像环境是的话只能用jdk动态代理，否则根据是否优化，是否配置直接代理目标类，是否无接口或者是否只使用SpringProxy接口选择CGLIB
      if (!NativeDetector.inNativeImage() &&
            (config.isOptimize() || config.isProxyTargetClass() || hasNoUserSuppliedProxyInterfaces(config))) {
         Class<?> targetClass = config.getTargetClass();
         if (targetClass == null) {
            throw new AopConfigException("TargetSource cannot determine target class: " +
                  "Either an interface or a target is required for proxy creation.");
         }
         if (targetClass.isInterface() || Proxy.isProxyClass(targetClass)) {
            return new JdkDynamicAopProxy(config);
         }
         return new ObjenesisCglibAopProxy(config);
      }
      else {
         return new JdkDynamicAopProxy(config);
      }
   }

   /**
    * Determine whether the supplied {@link AdvisedSupport} has only the
    * {@link org.springframework.aop.SpringProxy} interface specified
    * (or no proxy interfaces specified at all).
    */
   private boolean hasNoUserSuppliedProxyInterfaces(AdvisedSupport config) {
      Class<?>[] ifcs = config.getProxiedInterfaces();
      return (ifcs.length == 0 || (ifcs.length == 1 && SpringProxy.class.isAssignableFrom(ifcs[0])));
   }

}
```



### 代理配置

#### Advised

[`Advised`](https://github.com/spring-projects/spring-framework/blob/main/spring-aop/src/main/java/org/springframework/aop/framework/Advised.java)是AOP 代理配置管理接口，通过它可以实现`Advisor`、`Advice`的增删，以及代理类型的获取等配置，主要由代理工厂实现；同时**Spring AOP生成的代理对象都可以转换为Advised接口类型**（具体在生成代理对象时通过`AopProxyUtils#completeProxiedInterfaces`获取代理接口时加入的）。

关于`Advised`、`Advisor`以及`Advice`：

- `Advice`代表一个操作（通知、增强），即切面织入的新功能
- `Advisor`是`Advice`的容器类，可以获取`Advice`以及`Advice`是否适用的过滤器（`Pointcut`、`ClassFilter`）
- `Advised`代表已经被代理的对象，包含了多个`Advisor`（一个代理对象可以被多次增强）



#### ProxyConfig

`ProxyConfig`代理配置的基类，主要是为了确保代理创建者属性的一致

```java
/**
 * Convenience superclass for configuration used in creating proxies,
 * to ensure that all proxy creators have consistent properties.
 *
 */
public class ProxyConfig implements Serializable {
   ......
   //确认代理的类型，是否代理目标类（CGLIB）
   private boolean proxyTargetClass = false;
   
   //代理是否积极的优化，此属性影响代理类型的选择（Spring折衷的认为CGLIB是一种优化）,为true时有可能使用CGLIB代理，无论是否有实现接口
   private boolean optimize = false;

   //是否不透明，代理配置信息是否不公开，具体作用是是否将Advised接口接入加入代理对象接口列表中中，默认为false，即Spring AOP生成的代理对象都可以转换为Advised接口类型
   boolean opaque = false;

   //是否暴露代理对象，具体在AopContext获取当前代理对象应用中
   boolean exposeProxy = false;

   //是否被冻结，主要用来控制Advisor等配置的修改，true时不允许修改
   private boolean frozen = false;
   ......
}
```



#### AdvisedSupport

`AdvisedSupport`继承了`Advised`和`ProxyConfig`，主要提供代理配置的能力，它的子类都是代理工厂，另外`AdvisedSupport`还提供方法适配的`Advisor`缓存等能力（[AdvisorChainFactory](#advisorchainfactory)），`Advised`类关系如图：

![AdvisorAdapter](/img/post/aop/aop-api-advised.png)



### 代理对象创建

#### ProxyCreatorSupport

代理工厂的父类，继承`AdvisedSupport`，因此有配置的功能，组合`AopProxyFactory`结合自身配置实现创建代理抽象类`AopProxy`的能力。

```java
public class ProxyCreatorSupport extends AdvisedSupport {

   private AopProxyFactory aopProxyFactory;

......  

   /**
    * Return the AopProxyFactory that this ProxyConfig uses.
    */
   public AopProxyFactory getAopProxyFactory() {
      return this.aopProxyFactory;
   }
......

   /**
    * Subclasses should call this to get a new AOP proxy. They should <b>not</b>
    * create an AOP proxy with {@code this} as an argument.
    */
   protected final synchronized AopProxy createAopProxy() {
      if (!this.active) {
         activate();
      }
      return getAopProxyFactory().createAopProxy(this);
   }
......

}
```



#### API编程创建

##### ProxyFactory

默认实现，基于编程式的方法创建代理对象。

##### ProxyFactoryBean

实现了`FactoryBean`接口，提供了基于`FactoryBean`构建代理对象的能力。可以通过将`ProxyFactoryBean`注入到`IoC`容器，`setTargetName`配置`Bean`的名称获取对应的代理（或者`setTargetSource`）。

##### AspectJProxyFactory

跟ProxyFactory类似，但是提供直接指定代理对象需要绑定的切面类的能力（`addAspect`方法绑定切面）



#### 自动创建

##### AbstractAutoProxyCreator

在Spring 容器中，主要是通过`AbstractAutoProxyCreator`实现`Bean`的代理自动创建，具体为实现`SmartInstantiationAwareBeanPostProcessor`在`Bean`的生命周期中（实例化前或初始化后），使用容器中注册的`Advisor`，根据`ProxyFactory`创建代理对象，同时本身通过`ProxyProcessorSupport`也继承了`ProxyConfig`，具有代理配置的功能。

```java
public abstract class AbstractAutoProxyCreator extends ProxyProcessorSupport
      implements SmartInstantiationAwareBeanPostProcessor, BeanFactoryAware {

......

    //实例化前有可能直接创建为代理
   @Override
   public Object postProcessBeforeInstantiation(Class<?> beanClass, String beanName) {
      Object cacheKey = getCacheKey(beanClass, beanName);

      if (!StringUtils.hasLength(beanName) || !this.targetSourcedBeans.contains(beanName)) {
         if (this.advisedBeans.containsKey(cacheKey)) {
            return null;
         }
         if (isInfrastructureClass(beanClass) || shouldSkip(beanClass, beanName)) {
            this.advisedBeans.put(cacheKey, Boolean.FALSE);
            return null;
         }
      }

      // Create proxy here if we have a custom TargetSource.
      // Suppresses unnecessary default instantiation of the target bean:
      // The TargetSource will handle target instances in a custom fashion.
      TargetSource targetSource = getCustomTargetSource(beanClass, beanName);
      if (targetSource != null) {
         if (StringUtils.hasLength(beanName)) {
            this.targetSourcedBeans.add(beanName);
         }
          //根据当前Bean获取容器中适用的Advices、Advisors
         Object[] specificInterceptors = getAdvicesAndAdvisorsForBean(beanClass, beanName, targetSource);
         Object proxy = createProxy(beanClass, beanName, specificInterceptors, targetSource);
         this.proxyTypes.put(cacheKey, proxy.getClass());
         return proxy;
      }

      return null;
   }
......

    //初始化后也有可能创建代理
   /**
    * Create a proxy with the configured interceptors if the bean is
    * identified as one to proxy by the subclass.
    * @see #getAdvicesAndAdvisorsForBean
    */
   @Override
   public Object postProcessAfterInitialization(@Nullable Object bean, String beanName) {
      if (bean != null) {
         Object cacheKey = getCacheKey(bean.getClass(), beanName);
         if (this.earlyProxyReferences.remove(cacheKey) != bean) {
            return wrapIfNecessary(bean, beanName, cacheKey);
         }
      }
      return bean;
   }
.....

   /**
    * Wrap the given bean if necessary, i.e. if it is eligible for being proxied.
    * @param bean the raw bean instance
    * @param beanName the name of the bean
    * @param cacheKey the cache key for metadata access
    * @return a proxy wrapping the bean, or the raw bean instance as-is
    */
   protected Object wrapIfNecessary(Object bean, String beanName, Object cacheKey) {
      if (StringUtils.hasLength(beanName) && this.targetSourcedBeans.contains(beanName)) {
         return bean;
      }
      if (Boolean.FALSE.equals(this.advisedBeans.get(cacheKey))) {
         return bean;
      }
      if (isInfrastructureClass(bean.getClass()) || shouldSkip(bean.getClass(), beanName)) {
         this.advisedBeans.put(cacheKey, Boolean.FALSE);
         return bean;
      }

      // Create proxy if we have advice.
    //根据当前Bean获取容器中适用的Advices、Advisors
      Object[] specificInterceptors = getAdvicesAndAdvisorsForBean(bean.getClass(), beanName, null);
      if (specificInterceptors != DO_NOT_PROXY) {
         this.advisedBeans.put(cacheKey, Boolean.TRUE);
         Object proxy = createProxy(
               bean.getClass(), beanName, specificInterceptors, new SingletonTargetSource(bean));
         this.proxyTypes.put(cacheKey, proxy.getClass());
         return proxy;
      }

      this.advisedBeans.put(cacheKey, Boolean.FALSE);
      return bean;
   }
......

   /**
    * Create an AOP proxy for the given bean.
    * @param beanClass the class of the bean
    * @param beanName the name of the bean
    * @param specificInterceptors the set of interceptors that is
    * specific to this bean (may be empty, but not null)
    * @param targetSource the TargetSource for the proxy,
    * already pre-configured to access the bean
    * @return the AOP proxy for the bean
    * @see #buildAdvisors
    */
   protected Object createProxy(Class<?> beanClass, @Nullable String beanName,
         @Nullable Object[] specificInterceptors, TargetSource targetSource) {

      if (this.beanFactory instanceof ConfigurableListableBeanFactory) {
         AutoProxyUtils.exposeTargetClass((ConfigurableListableBeanFactory) this.beanFactory, beanName, beanClass);
      }

    //创建ProxyFactory，并配置
      ProxyFactory proxyFactory = new ProxyFactory();
      proxyFactory.copyFrom(this);

      if (proxyFactory.isProxyTargetClass()) {
         // Explicit handling of JDK proxy targets (for introduction advice scenarios)
         if (Proxy.isProxyClass(beanClass)) {
            // Must allow for introductions; can't just set interfaces to the proxy's interfaces only.
            for (Class<?> ifc : beanClass.getInterfaces()) {
               proxyFactory.addInterface(ifc);
            }
         }
      }
      else {
         // No proxyTargetClass flag enforced, let's apply our default checks...
         if (shouldProxyTargetClass(beanClass, beanName)) {
            proxyFactory.setProxyTargetClass(true);
         }
         else {
            evaluateProxyInterfaces(beanClass, proxyFactory);
         }
      }

    //将获取到的Advices、Advisors统一成Advisor
      Advisor[] advisors = buildAdvisors(beanName, specificInterceptors);
      proxyFactory.addAdvisors(advisors);
      proxyFactory.setTargetSource(targetSource);
      customizeProxyFactory(proxyFactory);

      proxyFactory.setFrozen(this.freezeProxy);
      if (advisorsPreFiltered()) {
         proxyFactory.setPreFiltered(true);
      }

      // Use original ClassLoader if bean class not locally loaded in overriding class loader
      ClassLoader classLoader = getProxyClassLoader();
      if (classLoader instanceof SmartClassLoader && classLoader != beanClass.getClassLoader()) {
         classLoader = ((SmartClassLoader) classLoader).getOriginalClassLoader();
      }
    //使用proxyFactory创建代理对象
      return proxyFactory.getProxy(classLoader);
   }

......
 
	/**
	 * Determine the advisors for the given bean, including the specific interceptors
	 * as well as the common interceptor, all adapted to the Advisor interface.
	 * @param beanName the name of the bean
	 * @param specificInterceptors the set of interceptors that is
	 * specific to this bean (may be empty, but not null)
	 * @return the list of Advisors for the given bean
	 */
	protected Advisor[] buildAdvisors(@Nullable String beanName, @Nullable Object[] specificInterceptors) {
		// Handle prototypes correctly...
    //获取容器中的通用的Advisor，默认没有
		Advisor[] commonInterceptors = resolveInterceptorNames();

		List<Object> allInterceptors = new ArrayList<>();
		if (specificInterceptors != null) {
			if (specificInterceptors.length > 0) {
				// specificInterceptors may equal PROXY_WITHOUT_ADDITIONAL_INTERCEPTORS
				allInterceptors.addAll(Arrays.asList(specificInterceptors));
			}
			if (commonInterceptors.length > 0) {
				if (this.applyCommonInterceptorsFirst) {
					allInterceptors.addAll(0, Arrays.asList(commonInterceptors));
				}
				else {
					allInterceptors.addAll(Arrays.asList(commonInterceptors));
				}
			}
		}
		if (logger.isTraceEnabled()) {
			int nrOfCommonInterceptors = commonInterceptors.length;
			int nrOfSpecificInterceptors = (specificInterceptors != null ? specificInterceptors.length : 0);
			logger.trace("Creating implicit proxy for bean '" + beanName + "' with " + nrOfCommonInterceptors +
					" common interceptors and " + nrOfSpecificInterceptors + " specific interceptors");
		}

		Advisor[] advisors = new Advisor[allInterceptors.size()];
		for (int i = 0; i < allInterceptors.size(); i++) {
            //通过advisorAdapterRegistry对Advice进行适配，最终包装成DefaultPointcutAdvisor
			advisors[i] = this.advisorAdapterRegistry.wrap(allInterceptors.get(i));
		}
		return advisors;
	}

	/**
	 * Resolves the specified interceptor names to Advisor objects.
	 * @see #setInterceptorNames
	 */
	private Advisor[] resolveInterceptorNames() {
		BeanFactory bf = this.beanFactory;
		ConfigurableBeanFactory cbf = (bf instanceof ConfigurableBeanFactory ? (ConfigurableBeanFactory) bf : null);
		List<Advisor> advisors = new ArrayList<>();
		for (String beanName : this.interceptorNames) {
			if (cbf == null || !cbf.isCurrentlyInCreation(beanName)) {
				Assert.state(bf != null, "BeanFactory required for resolving interceptor names");
				Object next = bf.getBean(beanName);
				advisors.add(this.advisorAdapterRegistry.wrap(next));
			}
		}
		return advisors.toArray(new Advisor[0]);
	}
    ......
}
```



主要类图如下：

![AutoProxyCreator](/img/post/aop/aop-api-AutoProxyCreator.png)



##### DefaultAdvisorAutoProxyCreator

默认实现，根据容器中注册的`Advisor`创建代理，可以配置`Advisor`的`BeanName`前缀匹配

##### BeanNameAutoProxyCreator

根据配置的`BeanNames`（或者匹配规则）判断`Bean`是否需要被代理。

##### AnnotationAwareAspectJAutoProxyCreator

`@Aspect`注解的代理自动创建

##### InfrastructureAdvisorAutoProxyCreator

过滤用户自定义`Advisor`，只考虑Spring内部的`Advisor`的代理自动创建器（比如事务、缓存的AOP）。

PS：判断方法为`BeanDefinition#getRole`是否等于`BeanDefinition#ROLE_INFRASTRUCTURE`。



### 补充

#### TargetSource

`TargetSource`目标源是对被代理对象的封装，Spring的代理的对象实际上都是`TargetSource`，在进行代理前，会使用`TargetSource`对目标进行包装。

```java
public interface TargetSource extends TargetClassAware {

   /**
    * Return the type of targets returned by this {@link TargetSource}.
    * <p>Can return {@code null}, although certain usages of a {@code TargetSource}
    * might just work with a predetermined target class.
    * @return the type of targets returned by this {@link TargetSource}
    */
   @Override
   @Nullable
   Class<?> getTargetClass();

   /**
    * Will all calls to {@link #getTarget()} return the same object?
    * <p>In that case, there will be no need to invoke {@link #releaseTarget(Object)},
    * and the AOP framework can cache the return value of {@link #getTarget()}.
    * @return {@code true} if the target is immutable
    * @see #getTarget
    */
   boolean isStatic();

   /**
    * Return a target instance. Invoked immediately before the
    * AOP framework calls the "target" of an AOP method invocation.
    * @return the target object which contains the joinpoint,
    * or {@code null} if there is no actual target instance
    * @throws Exception if the target object can't be resolved
    */
   @Nullable
   Object getTarget() throws Exception;

   /**
    * Release the given target object obtained from the
    * {@link #getTarget()} method, if any.
    * @param target object obtained from a call to {@link #getTarget()}
    * @throws Exception if the object can't be released
    */
   void releaseTarget(Object target) throws Exception;

}
```

有很多的实现，最主要就是`SingletonTargetSource`

- `SingletonTargetSource`，默认的`TargetSource`
- `LazyInitTargetSource`，针对懒加载`Bean`
- `JndiObjectTargetSource`，针对`JNDI`
- `ThreadLocalTargetSource`，线程隔离的
- ......太多了

#### AdvisorChainFactory

`Advisor`链工厂，提供根据方法筛选所有适用的`Advisor`的能力，只有`DefaultAdvisorChainFactory`一个实现，在`AdvisedSupport#getInterceptorsAndDynamicInterceptionAdvice`方法中使用，在`AopProxy`代理执行时，会根据此方法获取适配的`Advisor`链进行方法的增强，同时`AdvisedSupport`会把相应方法的`Advisors`进行缓存。

```java
public interface AdvisorChainFactory {

   /**
    * Determine a list of {@link org.aopalliance.intercept.MethodInterceptor} objects
    * for the given advisor chain configuration.
    * @param config the AOP configuration in the form of an Advised object
    * @param method the proxied method
    * @param targetClass the target class (may be {@code null} to indicate a proxy without
    * target object, in which case the method's declaring class is the next best option)
    * @return a List of MethodInterceptors (may also include IjavanterceptorAndDynamicMethodMatchers)
    */
   List<Object> getInterceptorsAndDynamicInterceptionAdvice(Advised config, Method method, @Nullable Class<?> targetClass);

}
```



#### AopUtils

提供查询适用的Advisor、执行代理方法等支持



## References

- <https://github.com/spring-projects/spring-framework/tree/main/spring-aop/src/main/java/org/springframework/aop>