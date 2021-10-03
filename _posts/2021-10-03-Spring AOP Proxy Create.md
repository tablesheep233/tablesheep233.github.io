---
layout:     post
title:      "Spring AOP 代理对象创建"
subtitle:   " \"Spring AOP 代理对象创建\""
author:     "tablesheep"
date:       2021-10-03 16:00:00
header-style: text
catalog: true
tags:
    - Java
    - Spring
    - Spring AOP
    - 源码
---



> Spring AOP 代理对象创建分析

# ProxyFactory API创建代理

```java
public class ApiCreateMain {
    public static void main(String[] args) {
        Runnable runnable = new Runnable() {
            @Override
            public void run() {
                System.out.println("do something...");
            }
        };
        //创建ProxyFactory，添加需要代理的对象
//        ProxyFactory proxyFactory = new ProxyFactory(runnable);
        ProxyFactory proxyFactory = new ProxyFactory();
        //直接代理类，设置后可能使用CGLIB代理
//        proxyFactory.setProxyTargetClass(true);
        //添加代理类
        proxyFactory.setTarget(runnable);
        //添加代理类接口
        proxyFactory.setInterfaces(Runnable.class);
        //添加需要加入的操作
        proxyFactory.addAdvisor(new DefaultPointcutAdvisor(new StaticMethodMatcherPointcut() {
            @Override
            public boolean matches(Method method, Class<?> targetClass) {
                return "run".equals(method.getName())
                        && Runnable.class.isAssignableFrom(targetClass);
            }
        }, new MethodInterceptor() {
            @Override
            public Object invoke(MethodInvocation invocation) throws Throwable {
                System.out.println("before");
                Object proceed = invocation.proceed();
                System.out.println("after");
                return proceed;
            }
        }));
        //获取代理对象
        Runnable proxy = (Runnable)proxyFactory.getProxy();
        //执行方法
        proxy.run();
    }
}
```



## 代理对象封装TargetSource

无论是使用构造方法，还是直接调用`setTarget`，最后都会将代理的对象封装成`TargetSource`（默认是`SingletonTargetSource`），Spring默认将代理对象封装成`TargetSource`进行统一的处理。



## 添加增强（Advisor或Advice）

通过`addAdvisor`添加`Advisor`

```java
@Override
public void addAdvisor(Advisor advisor) {
   int pos = this.advisors.size();
   addAdvisor(pos, advisor);
}
```

或者通过`addAdvice`添加`Advice`，其实也是使用`DefaultPointcutAdvisor`（默认所有类、方法都匹配）添加`Advisor`

```java
@Override
public void addAdvice(Advice advice) throws AopConfigException {
   int pos = this.advisors.size();
   addAdvice(pos, advice);
}

/**
 * Cannot add introductions this way unless the advice implements IntroductionInfo.
 */
@Override
public void addAdvice(int pos, Advice advice) throws AopConfigException {
   Assert.notNull(advice, "Advice must not be null");
   if (advice instanceof IntroductionInfo) {
      // We don't need an IntroductionAdvisor for this kind of introduction:
      // It's fully self-describing.
      addAdvisor(pos, new DefaultIntroductionAdvisor(advice, (IntroductionInfo) advice));
   }
   else if (advice instanceof DynamicIntroductionAdvice) {
      // We need an IntroductionAdvisor for this kind of introduction.
      throw new AopConfigException("DynamicIntroductionAdvice may only be added as part of IntroductionAdvisor");
   }
   else {
      addAdvisor(pos, new DefaultPointcutAdvisor(advice));
   }
}
```

以上操作的能力来自于`ProxyFactory` 的父类的父类`AdvisedSupport`（代理配置支持）



## 获取代理对象

`getProxy`方法通过`createAopProxy`方法获取`AopProxy`，再根据`AopProxy#getProxy`方法获取代理对象。

```java
public Object getProxy() {
   return createAopProxy().getProxy();
}
```



### AopProxy创建

`ProxyFactory` 的父类`ProxyCreatorSupport`，继承父类`AdvisedSupport`具备配置代理的能力，同时组合了`AopProxyFactory`（默认使用`DefaultAopProxyFactory`）`AopProxy`工厂类，实现创建`AopProxy`的方法`createAopProxy`（自身提供配置，委托`AopProxyFactory`创建）。

```java
public class ProxyCreatorSupport extends AdvisedSupport {

   private AopProxyFactory aopProxyFactory;
......

   /**
    * Create a new ProxyCreatorSupport instance.
    */
   public ProxyCreatorSupport() {
      this.aopProxyFactory = new DefaultAopProxyFactory();
   }
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

具体创建`AopProxy`逻辑看`DefaultAopProxyFactory`，会根据各种情况选择`JDK`动态代理还是`CGLIB`动态代理。

```java
public class DefaultAopProxyFactory implements AopProxyFactory, Serializable {

   @Override
   public AopProxy createAopProxy(AdvisedSupport config) throws AopConfigException {
           //GraalVM镜像环境用JDK动态代理，根据是否优化，是否配置直接代理目标类，是否无接口或者是否只使用SpringProxy接口选择CGLIB或JDK动态代理
      if (!NativeDetector.inNativeImage() &&
            (config.isOptimize() || config.isProxyTargetClass() || hasNoUserSuppliedProxyInterfaces(config))) {
         Class<?> targetClass = config.getTargetClass();
         if (targetClass == null) {
            throw new AopConfigException("TargetSource cannot determine target class: " +
                  "Either an interface or a target is required for proxy creation.");
         }
          //目标类是接口或已经是代理类使用JDK动态代理
         if (targetClass.isInterface() || Proxy.isProxyClass(targetClass)) {
            return new JdkDynamicAopProxy(config);
         }
          //其余情况使用CGLIB
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



### getProxy()获取代理对象

根据`AopProxyFactory`获取到`AopProxy`后，会通过`getProxy`获取代理对象，分为JDK和CGLIB两种

#### JDK代理

##### JdkDynamicAopProxy

```java
final class JdkDynamicAopProxy implements AopProxy, InvocationHandler, Serializable {
......

   /**
    * Construct a new JdkDynamicAopProxy for the given AOP configuration.
    * @param config the AOP configuration as AdvisedSupport object
    * @throws AopConfigException if the config is invalid. We try to throw an informative
    * exception in this case, rather than let a mysterious failure happen later.
    */
   public JdkDynamicAopProxy(AdvisedSupport config) throws AopConfigException {
    //构造方法使用AdvisedSupport进行相关配置
      Assert.notNull(config, "AdvisedSupport must not be null");
      if (config.getAdvisorCount() == 0 && config.getTargetSource() == AdvisedSupport.EMPTY_TARGET_SOURCE) {
         throw new AopConfigException("No advisors and no TargetSource specified");
      }
      this.advised = config;
    //获取目标类接口
      this.proxiedInterfaces = AopProxyUtils.completeProxiedInterfaces(this.advised, true);
      findDefinedEqualsAndHashCodeMethods(this.proxiedInterfaces);
   }


   @Override
   public Object getProxy() {
      return getProxy(ClassUtils.getDefaultClassLoader());
   }

   @Override
   public Object getProxy(@Nullable ClassLoader classLoader) {
      if (logger.isTraceEnabled()) {
         logger.trace("Creating JDK dynamic proxy: " + this.advised.getTargetSource());
      }
       //使用JDK动态代理API获取代理对象，自身实现InvocationHandler接口
      return Proxy.newProxyInstance(classLoader, this.proxiedInterfaces, this);
   }
    ......
}
```

#### CGLIB代理

Spring将CGLIB API代码直接放到spring-core中（减少版本冲突？降低依赖性？），使用相关API创建CGLIB动态代理对象。

##### CglibAopProxy

```java
class CglibAopProxy implements AopProxy, Serializable {
    	// Constants for CGLIB callback array indices
    //Callback数组的顺序
	private static final int AOP_PROXY = 0;
	private static final int INVOKE_TARGET = 1;
	private static final int NO_OVERRIDE = 2;
	private static final int DISPATCH_TARGET = 3;
	private static final int DISPATCH_ADVISED = 4;
	private static final int INVOKE_EQUALS = 5;
	private static final int INVOKE_HASHCODE = 6;
......

   /**
    * Create a new CglibAopProxy for the given AOP configuration.
    * @param config the AOP configuration as AdvisedSupport object
    * @throws AopConfigException if the config is invalid. We try to throw an informative
    * exception in this case, rather than let a mysterious failure happen later.
    */
   public CglibAopProxy(AdvisedSupport config) throws AopConfigException {
    //也是使用AdvisedSupport进行配置
      Assert.notNull(config, "AdvisedSupport must not be null");
      if (config.getAdvisorCount() == 0 && config.getTargetSource() == AdvisedSupport.EMPTY_TARGET_SOURCE) {
         throw new AopConfigException("No advisors and no TargetSource specified");
      }
      this.advised = config;
      this.advisedDispatcher = new AdvisedDispatcher(this.advised);
   }
......

   @Override
   public Object getProxy() {
      return getProxy(null);
   }

   @Override
   public Object getProxy(@Nullable ClassLoader classLoader) {
......

      try {
         Class<?> rootClass = this.advised.getTargetClass();
		Assert.state(rootClass != null, "Target class must be available for creating a CGLIB proxy");

		Class<?> proxySuperClass = rootClass;
		if (rootClass.getName().contains(ClassUtils.CGLIB_CLASS_SEPARATOR)) {
			proxySuperClass = rootClass.getSuperclass();
			Class<?>[] additionalInterfaces = rootClass.getInterfaces();
			for (Class<?> additionalInterface : additionalInterfaces) {
				this.advised.addInterface(additionalInterface);
			}
		}

		// Validate the class, writing log messages as necessary.
		validateClassIfNecessary(proxySuperClass, classLoader);

             //创建Enhancer并配置
         // Configure CGLIB Enhancer...
         Enhancer enhancer = createEnhancer();
         if (classLoader != null) {
            enhancer.setClassLoader(classLoader);
            if (classLoader instanceof SmartClassLoader &&
                  ((SmartClassLoader) classLoader).isClassReloadable(proxySuperClass)) {
               enhancer.setUseCache(false);
            }
         }
          //以当前代理目标作为父类
         enhancer.setSuperclass(proxySuperClass);
          //获取目标类接口
         enhancer.setInterfaces(AopProxyUtils.completeProxiedInterfaces(this.advised));
         enhancer.setNamingPolicy(SpringNamingPolicy.INSTANCE);
         enhancer.setStrategy(new ClassLoaderAwareGeneratorStrategy(classLoader));

          //获取所有的Callback（标识接口，所有的回调都实现了它，理解为AOP织入的操作就好了）
         Callback[] callbacks = getCallbacks(rootClass);
         Class<?>[] types = new Class<?>[callbacks.length];
         for (int x = 0; x < types.length; x++) {
            types[x] = callbacks[x].getClass();
         }
         // fixedInterceptorMap only populated at this point, after getCallbacks call above
         enhancer.setCallbackFilter(new ProxyCallbackFilter(
               this.advised.getConfigurationOnlyCopy(), this.fixedInterceptorMap, this.fixedInterceptorOffset));
         enhancer.setCallbackTypes(types);

          //使用CGLIB API创建代理类以及实例
         // Generate the proxy class and create a proxy instance.
         return createProxyClassAndInstance(enhancer, callbacks);
      }
      catch (CodeGenerationException | IllegalArgumentException ex) {
         throw new AopConfigException("Could not generate CGLIB subclass of " + this.advised.getTargetClass() +
               ": Common causes of this problem include using a final class or a non-visible class",
               ex);
      }
      catch (Throwable ex) {
         // TargetSource.getTarget() failed
         throw new AopConfigException("Unexpected AOP exception", ex);
      }
......
    
    //创建CGLIB回调方法
    private Callback[] getCallbacks(Class<?> rootClass) throws Exception {
		// Parameters used for optimization choices...
		boolean exposeProxy = this.advised.isExposeProxy();
		boolean isFrozen = this.advised.isFrozen();
		boolean isStatic = this.advised.getTargetSource().isStatic();

    //使用DynamicAdvisedInterceptor进行AOP拦截
		// Choose an "aop" interceptor (used for AOP calls).
		Callback aopInterceptor = new DynamicAdvisedInterceptor(this.advised);

		// Choose a "straight to target" interceptor. (used for calls that are
		// unadvised but can return this). May be required to expose the proxy.
		Callback targetInterceptor;
		if (exposeProxy) {
			targetInterceptor = (isStatic ?
					new StaticUnadvisedExposedInterceptor(this.advised.getTargetSource().getTarget()) :
					new DynamicUnadvisedExposedInterceptor(this.advised.getTargetSource()));
		}
		else {
			targetInterceptor = (isStatic ?
					new StaticUnadvisedInterceptor(this.advised.getTargetSource().getTarget()) :
					new DynamicUnadvisedInterceptor(this.advised.getTargetSource()));
		}

		// Choose a "direct to target" dispatcher (used for
		// unadvised calls to static targets that cannot return this).
		Callback targetDispatcher = (isStatic ?
				new StaticDispatcher(this.advised.getTargetSource().getTarget()) : new SerializableNoOp());

    //创建Callback数组，顺序有讲究
		Callback[] mainCallbacks = new Callback[] {
				aopInterceptor,  // for normal advice
				targetInterceptor,  // invoke target without considering advice, if optimized
				new SerializableNoOp(),  // no override for methods mapped to this
				targetDispatcher, this.advisedDispatcher,
				new EqualsInterceptor(this.advised),
				new HashCodeInterceptor(this.advised)
		};

		Callback[] callbacks;

    //当代理类的配置被冻结，并且代理类每次都返回同一个时，使用FixedChainStaticTargetInterceptor进行优化
		// If the target is a static one and the advice chain is frozen,
		// then we can make some optimizations by sending the AOP calls
		// direct to the target using the fixed chain for that method.
		if (isStatic && isFrozen) {
			Method[] methods = rootClass.getMethods();
			Callback[] fixedCallbacks = new Callback[methods.length];
			this.fixedInterceptorMap = CollectionUtils.newHashMap(methods.length);

			// TODO: small memory optimization here (can skip creation for methods with no advice)
			for (int x = 0; x < methods.length; x++) {
				Method method = methods[x];
				List<Object> chain = this.advised.getInterceptorsAndDynamicInterceptionAdvice(method, rootClass);
				fixedCallbacks[x] = new FixedChainStaticTargetInterceptor(
						chain, this.advised.getTargetSource().getTarget(), this.advised.getTargetClass());
				this.fixedInterceptorMap.put(method, x);
			}

			// Now copy both the callbacks from mainCallbacks
			// and fixedCallbacks into the callbacks array.
			callbacks = new Callback[mainCallbacks.length + fixedCallbacks.length];
			System.arraycopy(mainCallbacks, 0, callbacks, 0, mainCallbacks.length);
			System.arraycopy(fixedCallbacks, 0, callbacks, mainCallbacks.length, fixedCallbacks.length);
			this.fixedInterceptorOffset = mainCallbacks.length;
		}
		else {
			callbacks = mainCallbacks;
		}
		return callbacks;
	}
   ......
}
```

##### ObjenesisCglibAopProxy

```java
class ObjenesisCglibAopProxy extends CglibAopProxy {
......
    private static final SpringObjenesis objenesis = new SpringObjenesis();
......
   @Override
   protected Object createProxyClassAndInstance(Enhancer enhancer, Callback[] callbacks) {
      Class<?> proxyClass = enhancer.createClass();
      Object proxyInstance = null;

      if (objenesis.isWorthTrying()) {
         try {
             //Spring的优化，使用SpringObjenesis创建实例（缓存优化）
            proxyInstance = objenesis.newInstance(proxyClass, enhancer.getUseCache());
         }
         catch (Throwable ex) {
            logger.debug("Unable to instantiate proxy using Objenesis, " +
                  "falling back to regular proxy construction", ex);
         }
      }

      if (proxyInstance == null) {
         // Regular instantiation via default constructor...
         try {
            Constructor<?> ctor = (this.constructorArgs != null ?
                  proxyClass.getDeclaredConstructor(this.constructorArgTypes) :
                  proxyClass.getDeclaredConstructor());
            ReflectionUtils.makeAccessible(ctor);
            proxyInstance = (this.constructorArgs != null ?
                  ctor.newInstance(this.constructorArgs) : ctor.newInstance());
         }
         catch (Throwable ex) {
            throw new AopConfigException("Unable to instantiate proxy using Objenesis, " +
                  "and regular proxy instantiation via default constructor fails as well", ex);
         }
      }

      ((Factory) proxyInstance).setCallbacks(callbacks);
      return proxyInstance;
   }

}
```



## 总结

`ProxyFactory`基本就是个配置类，具体创建代理的能力是通过`AopProxyFactory`进行的，通过`AopProxyFactory`创建`JDK或CGLIB`代理抽象`AopProxy`，最后在调用`AopProxy#getProxy`获取真正的代理对象

# Spring 自动创建代理

在Spring中，有很多功能都是使用AOP实现的（事务、异步、缓存、用户自定义切面等），这些的实现是在创建`Bean`时就已经将原本的`Bean`对象使用代理完成的。

## AbstractAutoProxyCreator

实现自动创建代理`Bean`的核心，继承了`ProxyProcessorSupport`具有配置代理的功能，同时实现了`SmartInstantiationAwareBeanPostProcessor`，能在`Bean`生命周期的中有机会将`Bean`使用代理对象替换，实现了`BeanFactoryAware`注入`IoC`容器，可以实现`Advisor`的依赖查找。

具体参见流程，在`Bean`实例化前和初始化后都可能创建代理，以初始化后为例子，在初始化后会调用`wrapIfNecessary`对有需要进行代理的类使用代理包装。

### wrapIfNecessary

```java
protected Object wrapIfNecessary(Object bean, String beanName, Object cacheKey) {
   if (StringUtils.hasLength(beanName) && this.targetSourcedBeans.contains(beanName)) {
      return bean;
   }
    //是否已经被代理过
   if (Boolean.FALSE.equals(this.advisedBeans.get(cacheKey))) {
      return bean;
   }
    //是否基础设施（Advice, Advisor, AopInfrastructureBean,PointCut）,
    //是否该被跳过（String ORIGINAL_INSTANCE_SUFFIX = ".ORIGINAL";），默认判断后缀，在AspectJ中还根据AspectJPointcutAdvisor#aspectName与BeanName是否一致过滤
   if (isInfrastructureClass(bean.getClass()) || shouldSkip(bean.getClass(), beanName)) {
      this.advisedBeans.put(cacheKey, Boolean.FALSE);
      return bean;
   }

    //重点在这里，使用getAdvicesAndAdvisorsForBean模板方法根据Bean获取适用的Advice和Advisor
   // Create proxy if we have advice.
   Object[] specificInterceptors = getAdvicesAndAdvisorsForBean(bean.getClass(), beanName, null);
   if (specificInterceptors != DO_NOT_PROXY) {
      this.advisedBeans.put(cacheKey, Boolean.TRUE);
       //调用createProxy方法创建代理对象（直接将bean使用TargetSource包装）
      Object proxy = createProxy(
            bean.getClass(), beanName, specificInterceptors, new SingletonTargetSource(bean));
      this.proxyTypes.put(cacheKey, proxy.getClass());
      return proxy;
   }

   this.advisedBeans.put(cacheKey, Boolean.FALSE);
   return bean;
}
```

可以看到重点就是`Advisor`的获取，以及`createProxy`创建代理对象，具体的`getAdvicesAndAdvisorsForBean`逻辑主要集中在子类`AbstractAdvisorAutoProxyCreator`中。



### getAdvicesAndAdvisorsForBean，获取Advisor

[Advisor获取](##advisor获取)



### createProxy，使用ProxyFactory创建代理对象

配置`ProxyFactory`，使用`ProxyFactory`进行代理对象创建

```java
protected Object createProxy(Class<?> beanClass, @Nullable String beanName,
      @Nullable Object[] specificInterceptors, TargetSource targetSource) {

   if (this.beanFactory instanceof ConfigurableListableBeanFactory) {
      AutoProxyUtils.exposeTargetClass((ConfigurableListableBeanFactory) this.beanFactory, beanName, beanClass);
   }

    //创建ProxyFactory,使用自身配置进行配置
   ProxyFactory proxyFactory = new ProxyFactory();
   proxyFactory.copyFrom(this);

    //处理是否直接代理类，以及添加接口
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

    //构建Advisor(会把Advice也构建成Advisor)，添加Advisor以及targetSource
   Advisor[] advisors = buildAdvisors(beanName, specificInterceptors);
   proxyFactory.addAdvisors(advisors);
   proxyFactory.setTargetSource(targetSource);
   customizeProxyFactory(proxyFactory);

    //配置冻结属性，默认false
   proxyFactory.setFrozen(this.freezeProxy);
   if (advisorsPreFiltered()) {
      proxyFactory.setPreFiltered(true);
   }

   // Use original ClassLoader if bean class not locally loaded in overriding class loader
   ClassLoader classLoader = getProxyClassLoader();
   if (classLoader instanceof SmartClassLoader && classLoader != beanClass.getClassLoader()) {
      classLoader = ((SmartClassLoader) classLoader).getOriginalClassLoader();
   }
    //使用ProxyFactory API创建代理对象
   return proxyFactory.getProxy(classLoader);
}
```



## Advisor获取

## AbstractAdvisorAutoProxyCreator

### getAdvicesAndAdvisorsForBean

在`AbstractAdvisorAutoProxyCreator#getAdvicesAndAdvisorsForBean`中，最主要就是调用了`findEligibleAdvisors`方法查找适用的`Advisor`

```java
protected Object[] getAdvicesAndAdvisorsForBean(
      Class<?> beanClass, String beanName, @Nullable TargetSource targetSource) {

   List<Advisor> advisors = findEligibleAdvisors(beanClass, beanName);
   if (advisors.isEmpty()) {
      return DO_NOT_PROXY;
   }
   return advisors.toArray();
}
```



### findEligibleAdvisors

`findEligibleAdvisors`首先调用`findCandidateAdvisors`查找`Advisor`，然后使用`findAdvisorsThatCanApply`方法获取其中可以使用的`Advisor`

```java
protected List<Advisor> findEligibleAdvisors(Class<?> beanClass, String beanName) {
   List<Advisor> candidateAdvisors = findCandidateAdvisors();
   List<Advisor> eligibleAdvisors = findAdvisorsThatCanApply(candidateAdvisors, beanClass, beanName);
   extendAdvisors(eligibleAdvisors);
   if (!eligibleAdvisors.isEmpty()) {
      eligibleAdvisors = sortAdvisors(eligibleAdvisors);
   }
   return eligibleAdvisors;
}
```



#### findCandidateAdvisors

`findCandidateAdvisors`会使用在`setBeanFactory`初始化的`advisorRetrievalHelper`查找容器中的所有`Advisor`（`@AspectJ`注解的类也会添加，在`AnnotationAwareAspectJAutoProxyCreator`中会重写`findCandidateAdvisors`实现添加）

```java
protected void initBeanFactory(ConfigurableListableBeanFactory beanFactory) {
    this.advisorRetrievalHelper = new BeanFactoryAdvisorRetrievalHelperAdapter(beanFactory);
}

protected List<Advisor> findCandidateAdvisors() {
   Assert.state(this.advisorRetrievalHelper != null, "No BeanFactoryAdvisorRetrievalHelper available");
   return this.advisorRetrievalHelper.findAdvisorBeans();
}
```

##### BeanFactoryAdvisorRetrievalHelper#findAdvisorBeans

```java
public class BeanFactoryAdvisorRetrievalHelper {
......

   public List<Advisor> findAdvisorBeans() {
    //从缓存或者根据beanFactory获取advisorNames
      // Determine list of advisor bean names, if not cached already.
      String[] advisorNames = this.cachedAdvisorBeanNames;
      if (advisorNames == null) {
         // Do not initialize FactoryBeans here: We need to leave all regular beans
         // uninitialized to let the auto-proxy creator apply to them!
         advisorNames = BeanFactoryUtils.beanNamesForTypeIncludingAncestors(
               this.beanFactory, Advisor.class, true, false);
         this.cachedAdvisorBeanNames = advisorNames;
      }
      if (advisorNames.length == 0) {
         return new ArrayList<>();
      }

      List<Advisor> advisors = new ArrayList<>();
      for (String name : advisorNames) {
          //判断advisor是否合格，默认合格
         if (isEligibleBean(name)) {
             //判断advisor是否在创建中，是跳过
            if (this.beanFactory.isCurrentlyInCreation(name)) {
               if (logger.isTraceEnabled()) {
                  logger.trace("Skipping currently created advisor '" + name + "'");
               }
            }
            else {
                //否则从容器中获取advisor并添加
               try {
                  advisors.add(this.beanFactory.getBean(name, Advisor.class));
               }
               catch (BeanCreationException ex) {
                  ......
               }
            }
         }
      }
      return advisors;
   }

   /**
    * Determine whether the aspect bean with the given name is eligible.
    * <p>The default implementation always returns {@code true}.
    * @param beanName the name of the aspect bean
    * @return whether the bean is eligible
    */
   protected boolean isEligibleBean(String beanName) {
      return true;
   }

}
```



##### BeanFactoryAdvisorRetrievalHelperAdapter

```java
private class BeanFactoryAdvisorRetrievalHelperAdapter extends BeanFactoryAdvisorRetrievalHelper {
......

   @Override
   protected boolean isEligibleBean(String beanName) {
    //重写isEligibleBean方法，默认使用AbstractAdvisorAutoProxyCreator当前类isEligibleAdvisorBean方法判断，主要体现在InfrastructureAdvisorAutoProxyCreator中
      return AbstractAdvisorAutoProxyCreator.this.isEligibleAdvisorBean(beanName);
   }
}
```

@see [InfrastructureAdvisorAutoProxyCreator](#infrastructureadvisorautoproxycreator)



#### findAdvisorsThatCanApply

`findAdvisorsThatCanApply`会调用`AopUtils#findAdvisorsThatCanApply`查找所有能用的`Advisor`

```java
protected List<Advisor> findAdvisorsThatCanApply(
      List<Advisor> candidateAdvisors, Class<?> beanClass, String beanName) {
   ProxyCreationContext.setCurrentProxiedBeanName(beanName);
   try {
      return AopUtils.findAdvisorsThatCanApply(candidateAdvisors, beanClass);
   }
   finally {
      ProxyCreationContext.setCurrentProxiedBeanName(null);
   }
}
```

##### AopUtils#canApply

最终会使用`canApply`方法判断，主要还是获取`Advisor`中的`ClassFilter`或者`Pointcut`进行判断

```java
public static boolean canApply(Advisor advisor, Class<?> targetClass, boolean hasIntroductions) {
   if (advisor instanceof IntroductionAdvisor) {
       //IntroductionAdvisor使用ClassFilter判断
      return ((IntroductionAdvisor) advisor).getClassFilter().matches(targetClass);
   }
   else if (advisor instanceof PointcutAdvisor) {
      PointcutAdvisor pca = (PointcutAdvisor) advisor;
       //PointcutAdvisor获取Pointcut进行判断
      return canApply(pca.getPointcut(), targetClass, hasIntroductions);
   }
   else {
      // It doesn't have a pointcut so we assume it applies.
      return true;
   }
}
```



## InfrastructureAdvisorAutoProxyCreator，基础设施自动代理

Spring基础设施自动代理，忽略用户自定义的`Advisor`（`@EnableCaching`、`@EnableTransactionManagement`等）

```java
public class InfrastructureAdvisorAutoProxyCreator extends AbstractAdvisorAutoProxyCreator {
......

   @Override
   protected boolean isEligibleAdvisorBean(String beanName) {
    //根据BeanDefinition的role进行判断，比如在@EnableCaching引入的BeanFactoryCacheOperationSourceAdvisor上有@Role(BeanDefinition.ROLE_INFRASTRUCTURE)
      return (this.beanFactory != null && this.beanFactory.containsBeanDefinition(beanName) &&
            this.beanFactory.getBeanDefinition(beanName).getRole() == BeanDefinition.ROLE_INFRASTRUCTURE);
   }

}
```



## AnnotationAwareAspectJAutoProxyCreator，@AspcetJ自动代理

```java
public class AnnotationAwareAspectJAutoProxyCreator extends AspectJAwareAdvisorAutoProxyCreator {

   @Nullable
   private List<Pattern> includePatterns;

   @Nullable
   private AspectJAdvisorFactory aspectJAdvisorFactory;

   @Nullable
   private BeanFactoryAspectJAdvisorsBuilder aspectJAdvisorsBuilder;

......

   @Override
   protected void initBeanFactory(ConfigurableListableBeanFactory beanFactory) {
      super.initBeanFactory(beanFactory);
      if (this.aspectJAdvisorFactory == null) {
         this.aspectJAdvisorFactory = new ReflectiveAspectJAdvisorFactory(beanFactory);
      }
      this.aspectJAdvisorsBuilder =
            new BeanFactoryAspectJAdvisorsBuilderAdapter(beanFactory, this.aspectJAdvisorFactory);
   }

    //除了调用父类的方法，增加了使用BeanFactoryAspectJAdvisorsBuilder获取@AspcetJ类型Advisor的能力,具体逻辑与BeanFactoryAdvisorRetrievalHelper系列类相似
   @Override
   protected List<Advisor> findCandidateAdvisors() {
      // Add all the Spring advisors found according to superclass rules.
      List<Advisor> advisors = super.findCandidateAdvisors();
      // Build Advisors for all AspectJ aspects in the bean factory.
      if (this.aspectJAdvisorsBuilder != null) {
         advisors.addAll(this.aspectJAdvisorsBuilder.buildAspectJAdvisors());
      }
      return advisors;
   }
......

}
```



## 总结

Spring自动创建代理对象基于`Bean`生命周期扩展点实现，使用容器中注册的`Advisor`，通过`ProxyFactory`实现自动代理的创建。



## References

- <https://github.com/spring-projects/spring-framework/tree/main/spring-aop/src/main/java/org/springframework/aop>