---
layout:     post
title:      "Spring Cloud RefreshScope 工作原理"
subtitle:   " \"Spring Cloud RefreshScope 工作原理\""
author:     "tablesheep"
date:       2022-04-07 19:20:00
header-style: text
catalog: true
tags:
- Java
- Spring
- Spring Cloud
- 源码
---


### RefreshEvent刷新配置过程

偷懒一张图~~~

![RefreshEvent flow](/img/post/spring scope/refresh/RefreshEvent flow.jpg)



翻译翻译

- `RefreshEventListener`在接收到`RefreshEvent`时，会通过`ContextRefresher`刷新配置，使用bootstrap配置的情况下，主要看`LegacyContextRefresher#updateEnvironment`，它的大致逻辑就是创建一个新的`ApplicationContext`去获取最新的配置，然后通过替换原`ApplicationContext`的`PropertySource`的方式达到配置刷新的效果。
- 接着会调用`RefreshScope#refreshAll`方法，该方法的主要作用是清空refresh级别的Bean缓存，当应用重新获取Bean时注入配置的情况下就会获取到最新的配置了。

**LegacyContextRefresher**

```java
ConfigurableApplicationContext addConfigFilesToEnvironment() {
   ConfigurableApplicationContext capture = null;
   try {
      StandardEnvironment environment = copyEnvironment(getContext().getEnvironment());

      Map<String, Object> map = new HashMap<>();
      map.put("spring.jmx.enabled", false);
      map.put("spring.main.sources", "");
      // gh-678 without this apps with this property set to REACTIVE or SERVLET fail
      map.put("spring.main.web-application-type", "NONE");
      map.put(BOOTSTRAP_ENABLED_PROPERTY, Boolean.TRUE.toString());
      environment.getPropertySources().addFirst(new MapPropertySource(REFRESH_ARGS_PROPERTY_SOURCE, map));

      SpringApplicationBuilder builder = new SpringApplicationBuilder(Empty.class).bannerMode(Banner.Mode.OFF)
            .web(WebApplicationType.NONE).environment(environment);
      // Just the listeners that affect the environment (e.g. excluding logging
      // listener because it has side effects)
      builder.application().setListeners(
            Arrays.asList(new BootstrapApplicationListener(), new BootstrapConfigFileApplicationListener()));
       //创建一个新的ApplicationContext
      capture = builder.run();
      if (environment.getPropertySources().contains(REFRESH_ARGS_PROPERTY_SOURCE)) {
         environment.getPropertySources().remove(REFRESH_ARGS_PROPERTY_SOURCE);
      }
      MutablePropertySources target = getContext().getEnvironment().getPropertySources();
      String targetName = null;
      for (PropertySource<?> source : environment.getPropertySources()) {
         String name = source.getName();
         if (target.contains(name)) {
            targetName = name;
         }
         if (!this.standardSources.contains(name)) {
            if (target.contains(name)) {
                //循环替换PropertySource
               target.replace(name, source);
            }
            else {
               if (targetName != null) {
                  target.addAfter(targetName, source);
                  // update targetName to preserve ordering
                  targetName = name;
               }
               else {
                  // targetName was null so we are at the start of the list
                  target.addFirst(source);
                  targetName = name;
               }
            }
         }
      }
   }
   finally {
      ConfigurableApplicationContext closeable = capture;
      while (closeable != null) {
         try {
            closeable.close();
         }
         catch (Exception e) {
            // Ignore;
         }
         if (closeable.getParent() instanceof ConfigurableApplicationContext) {
            closeable = (ConfigurableApplicationContext) closeable.getParent();
         }
         else {
            break;
         }
      }
   }
   return capture;
}
```





### refresh 作用域的Bean获取过程

refresh作用域的Bean会通过`RefreshScope`获取对应的Bean实例，大致的过程如下：

- `GenericScope#get` 方法 通过`BeanLifecycleWrapperCache#put`获取 `BeanLifecycelWrapper`，这个cache 深追其实就是个`ConcurrentMap`，`BeanLifecycleWrapperCache#put` 就是 `Map#putIfAbsent`，第一次会保存`BeanLifecycelWrapper`，后面都是在map中获取。

- `BeanLifecycelWrapper#getBean` 方法获取Bean实例，首次通过`ObjectFactory` 创建，后续直接返回。

  

下面看看类具体的代码

#### RefreshScope

`RefreshScope` 主要实现了refresh的逻辑，获取Bean的实现集中在父类`GenericScope`中。在refresh时是通过销毁此作用域的Bean，然后重新加载以达到refresh的效果，同时refresh作用域的Bean都是懒加载的

```java
/**
 * <p>
 * A Scope implementation that allows for beans to be refreshed dynamically at runtime
 * (see {@link #refresh(String)} and {@link #refreshAll()}). If a bean is refreshed then
 * the next time the bean is accessed (i.e. a method is executed) a new instance is
 * created. All lifecycle methods are applied to the bean instances, so any destruction
 * callbacks that were registered in the bean factory are called when it is refreshed, and
 * then the initialization callbacks are invoked as normal when the new instance is
 * created. A new bean instance is created from the original bean definition, so any
 * externalized content (property placeholders or expressions in string literals) is
 * re-evaluated when it is created.
 * </p>
 *
 * <p>
 * Note that all beans in this scope are <em>only</em> initialized when first accessed, so
 * the scope forces lazy initialization semantics.
 * </p>
 *
 * <p>
 * The scoped proxy approach adopted here has a side benefit that bean instances are
 * automatically {@link Serializable}, and can be sent across the wire as long as the
 * receiver has an identical application context on the other side. To ensure that the two
 * contexts agree that they are identical, they have to have the same serialization ID.
 * One will be generated automatically by default from the bean names, so two contexts
 * with the same bean names are by default able to exchange beans by name. If you need to
 * override the default ID, then provide an explicit {@link #setId(String) id} when the
 * Scope is declared.
 * </p>
 *
 * @author Dave Syer
 * @since 3.1
 *
 */
@ManagedResource
public class RefreshScope extends GenericScope
      implements ApplicationContextAware, ApplicationListener<ContextRefreshedEvent>, Ordered {

   private ApplicationContext context;

   private BeanDefinitionRegistry registry;

   private boolean eager = true;

   private int order = Ordered.LOWEST_PRECEDENCE - 100;

   /**
    * Creates a scope instance and gives it the default name: "refresh".
    */
   public RefreshScope() {
      super.setName("refresh");
   }

......

   @Override
   public void postProcessBeanDefinitionRegistry(BeanDefinitionRegistry registry) throws BeansException {
      this.registry = registry;
    //主要逻辑在GenericScope中
      super.postProcessBeanDefinitionRegistry(registry);
   }

   @Override
   public void onApplicationEvent(ContextRefreshedEvent event) {
      start(event);
   }

   public void start(ContextRefreshedEvent event) {
       //在RefreshScope的Application refresh时才会执行
      if (event.getApplicationContext() == this.context && this.eager && this.registry != null) {
         eagerlyInitialize();
      }
   }

   private void eagerlyInitialize() {
       //在Application refresh时创建所有refresh的Bean
      for (String name : this.context.getBeanDefinitionNames()) {
         BeanDefinition definition = this.registry.getBeanDefinition(name);
         if (this.getName().equals(definition.getScope()) && !definition.isLazyInit()) {
            Object bean = this.context.getBean(name);
            if (bean != null) {
               bean.getClass();
            }
         }
      }
   }

   @ManagedOperation(description = "Dispose of the current instance of bean name "
         + "provided and force a refresh on next method execution.")
   public boolean refresh(String name) {
      if (!ScopedProxyUtils.isScopedTarget(name)) {
         // User wants to refresh the bean with this name but that isn't the one in the
         // cache...
         name = ScopedProxyUtils.getTargetBeanName(name);
      }
      // Ensure lifecycle is finished if bean was disposable
      if (super.destroy(name)) {
         this.context.publishEvent(new RefreshScopeRefreshedEvent(name));
         return true;
      }
      return false;
   }

   @ManagedOperation(description = "Dispose of the current instance of all beans "
         + "in this scope and force a refresh on next method execution.")
   public void refreshAll() {
       //销毁Bean，然后发布RefreshScopeRefreshedEvent事件，所以refresh 的Bean 可以监听此事件以使Bean创建
      super.destroy();
      this.context.publishEvent(new RefreshScopeRefreshedEvent());
   }

   @Override
   public void setApplicationContext(ApplicationContext context) throws BeansException {
      this.context = context;
   }

}
```



#### GenericScope

RefreshScope 的父类，

```java
public class GenericScope
      implements Scope, BeanFactoryPostProcessor, BeanDefinitionRegistryPostProcessor, DisposableBean {

   private static final Log logger = LogFactory.getLog(GenericScope.class);

   private BeanLifecycleWrapperCache cache = new BeanLifecycleWrapperCache(new StandardScopeCache());

   private String name = "generic";

   private ConfigurableListableBeanFactory beanFactory;

   private StandardEvaluationContext evaluationContext;

   private String id;

   private Map<String, Exception> errors = new ConcurrentHashMap<>();

   private ConcurrentMap<String, ReadWriteLock> locks = new ConcurrentHashMap<>();

......

   /**
    * The cache implementation to use for bean instances in this scope.
    * @param cache The cache to use.
    */
   public void setScopeCache(ScopeCache cache) {
      this.cache = new BeanLifecycleWrapperCache(cache);
   }

......

   @Override
   public void destroy() {
      List<Throwable> errors = new ArrayList<Throwable>();
    //清空ScopeCache缓存
      Collection<BeanLifecycleWrapper> wrappers = this.cache.clear();
      for (BeanLifecycleWrapper wrapper : wrappers) {
         try {
            Lock lock = this.locks.get(wrapper.getName()).writeLock();
            lock.lock();
            try {
                //回调销毁方法
               wrapper.destroy();
            }
            finally {
               lock.unlock();
            }
         }
         catch (RuntimeException e) {
            errors.add(e);
         }
      }
      if (!errors.isEmpty()) {
         throw wrapIfNecessary(errors.get(0));
      }
      this.errors.clear();
   }
......

    //AbstractBeanFactory doGetBean调用
   @Override
   public Object get(String name, ObjectFactory<?> objectFactory) {
    //从缓存中获取BeanLifecycleWrapper
      BeanLifecycleWrapper value = this.cache.put(name, new BeanLifecycleWrapper(name, objectFactory));
      this.locks.putIfAbsent(name, new ReentrantReadWriteLock());
      try {
          //然后返回BeanLifecycleWrapper中的Bean
         return value.getBean();
      }
      catch (RuntimeException e) {
         this.errors.put(name, e);
         throw e;
      }
   }

......

   @Override
   public void postProcessBeanFactory(ConfigurableListableBeanFactory beanFactory) throws BeansException {
      this.beanFactory = beanFactory;
      beanFactory.registerScope(this.name, this);
      setSerializationId(beanFactory);
   }

   @Override
   public void postProcessBeanDefinitionRegistry(BeanDefinitionRegistry registry) throws BeansException {
      for (String name : registry.getBeanDefinitionNames()) {
         BeanDefinition definition = registry.getBeanDefinition(name);
         if (definition instanceof RootBeanDefinition) {
            RootBeanDefinition root = (RootBeanDefinition) definition;
            if (root.getDecoratedDefinition() != null && root.hasBeanClass()
                  && root.getBeanClass() == ScopedProxyFactoryBean.class) {
               if (getName().equals(root.getDecoratedDefinition().getBeanDefinition().getScope())) {
                   //争对Scope#proxyMode != NO 的BeanDefinition，在它被ScopedProxyUtils#createScopedProxy处理过后，若属于此Scope
                   //升级BeanClass ScopedProxyFactoryBean -> LockedScopedProxyFactoryBean
                  root.setBeanClass(LockedScopedProxyFactoryBean.class);
                  root.getConstructorArgumentValues().addGenericArgumentValue(this);
                  // surprising that a scoped proxy bean definition is not already
                  // marked as synthetic?
                  root.setSynthetic(true);
               }
            }
         }
      }
   }
......

   protected ReadWriteLock getLock(String beanName) {
      return this.locks.get(beanName);
   }

    //BeanLifecycleWrapper 缓存，主要是通过StandardScopeCache实现，逻辑很简单不赘述了
   private static class BeanLifecycleWrapperCache {

      private final ScopeCache cache;

      BeanLifecycleWrapperCache(ScopeCache cache) {
         this.cache = cache;
      }

      public BeanLifecycleWrapper remove(String name) {
         return (BeanLifecycleWrapper) this.cache.remove(name);
      }

      public Collection<BeanLifecycleWrapper> clear() {
         Collection<Object> values = this.cache.clear();
         Collection<BeanLifecycleWrapper> wrappers = new LinkedHashSet<BeanLifecycleWrapper>();
         for (Object object : values) {
            wrappers.add((BeanLifecycleWrapper) object);
         }
         return wrappers;
      }

      public BeanLifecycleWrapper get(String name) {
         return (BeanLifecycleWrapper) this.cache.get(name);
      }

      public BeanLifecycleWrapper put(String name, BeanLifecycleWrapper value) {
         return (BeanLifecycleWrapper) this.cache.put(name, value);
      }

   }

//包装了Bean name, objectFactory, bean实例，生命周期回调方法
   private static class BeanLifecycleWrapper {

      private final String name;

      private final ObjectFactory<?> objectFactory;

      private volatile Object bean;

      private Runnable callback;

      BeanLifecycleWrapper(String name, ObjectFactory<?> objectFactory) {
         this.name = name;
         this.objectFactory = objectFactory;
      }
......

      public Object getBean() {
         if (this.bean == null) {
            synchronized (this.name) {
               if (this.bean == null) {
                   //从传入的objectFactory获取Bean，默认就是AbstractBeanFactory#doGetBean中的那个匿名内部ObjectFactory
                  this.bean = this.objectFactory.getObject();
               }
            }
         }
         return this.bean;
      }
......
   }

   /**
    * A factory bean with a locked scope.
    *
    * @param <S> - a generic scope extension
    */
   @SuppressWarnings("serial")
   public static class LockedScopedProxyFactoryBean<S extends GenericScope> extends ScopedProxyFactoryBean
         implements MethodInterceptor {

      private final S scope;

      private String targetBeanName;

      public LockedScopedProxyFactoryBean(S scope) {
         this.scope = scope;
      }

      @Override
      public void setBeanFactory(BeanFactory beanFactory) {
         super.setBeanFactory(beanFactory);
         Object proxy = getObject();
         if (proxy instanceof Advised) {
            Advised advised = (Advised) proxy;
            advised.addAdvice(0, this);
         }
      }

      @Override
      public void setTargetBeanName(String targetBeanName) {
         super.setTargetBeanName(targetBeanName);
         this.targetBeanName = targetBeanName;
      }

      @Override
      public Object invoke(MethodInvocation invocation) throws Throwable {
         Method method = invocation.getMethod();
         if (AopUtils.isEqualsMethod(method) || AopUtils.isToStringMethod(method)
               || AopUtils.isHashCodeMethod(method) || isScopedObjectGetTargetObject(method)) {
            return invocation.proceed();
         }
         Object proxy = getObject();
         ReadWriteLock readWriteLock = this.scope.getLock(this.targetBeanName);
         if (readWriteLock == null) {
            if (logger.isDebugEnabled()) {
               logger.debug("For bean with name [" + this.targetBeanName
                     + "] there is no read write lock. Will create a new one to avoid NPE");
            }
            readWriteLock = new ReentrantReadWriteLock();
         }
         Lock lock = readWriteLock.readLock();
         lock.lock();
         try {
            if (proxy instanceof Advised) {
               Advised advised = (Advised) proxy;
               ReflectionUtils.makeAccessible(method);
               return ReflectionUtils.invokeMethod(method, advised.getTargetSource().getTarget(),
                     invocation.getArguments());
            }
            return invocation.proceed();
         }
         // see gh-349. Throw the original exception rather than the
         // UndeclaredThrowableException
         catch (UndeclaredThrowableException e) {
            throw e.getUndeclaredThrowable();
         }
         finally {
            lock.unlock();
         }
      }

      private boolean isScopedObjectGetTargetObject(Method method) {
         return method.getDeclaringClass().equals(ScopedObject.class) && method.getName().equals("getTargetObject")
               && method.getParameterTypes().length == 0;
      }
   }
}
```



##### StandardScopeCache

使用`ConcurrentMap`实现缓存

```java
public class StandardScopeCache implements ScopeCache {

   private final ConcurrentMap<String, Object> cache = new ConcurrentHashMap<String, Object>();

   public Object remove(String name) {
      return this.cache.remove(name);
   }

   public Collection<Object> clear() {
      Collection<Object> values = new ArrayList<Object>(this.cache.values());
      this.cache.clear();
      return values;
   }

   public Object get(String name) {
      return this.cache.get(name);
   }

   public Object put(String name, Object value) {
      Object result = this.cache.putIfAbsent(name, value);
      if (result != null) {
         return result;
      }
      return value;
   }

}
```





### @RefreshScope使用注意事项

由于refresh级别的Bean在刷新时会被销毁而且它是懒加载的，所以在一些功能上会有问题，比如：与`@Scheduled`一起使用时，由于Bean被销毁了，定时任务也会不存在，可以通过几个方法去避免这样的问题。

- 发布对应Application的`ContextRefreshedEvent`，可以继承`RefreshScope` 在`refreshAll `方法操作等等
- refresh级别的Bean监听`RefreshScopeRefreshedEvent`
- 重新设计代码，refresh级别的Bean作为配置类使用，保持类的单一职责
- 使用`Environment` 获取配置
- ......



在使用`@RefreshScope`时，不要随意改动`proxyMode`，当然有接口的情况下可以使用`ScopedProxyMode#INTERFACES`，不要使用`ScopedProxyMode#NO`，除非你能保证Bean永远不会被注入，否则一开始注入的那个Bean将会变成一个脱离容器管理的普通变量存在于被注入的Bean中，无论配置如何改变都会是一开始的值。（类似的对于@RequestScope的Bean也一样）



### 总结

`RefreshEventListener`接收到`RefreshEvent`后，通过`ContextRefresher`刷新配置，然后再由`RefreshScope`清空所有refresh级别的Bean；后续获取Bean时由 `BeanLifecycleWrapperCache`创建&缓存 `BeanLifecycleWrapper`，通过`BeanLifecycleWrapper`获取Bean实例时就会注入最新的配置，从而达到配置的刷新效果。



### References

- <https://docs.spring.io/spring-cloud-commons/docs/current/reference/html/#refresh-scope>
