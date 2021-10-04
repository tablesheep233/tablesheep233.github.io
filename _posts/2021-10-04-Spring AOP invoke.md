---
layout:     post
title:      "Spring AOP 代理方法执行"
subtitle:   " \"Spring AOP 代理方法执行\""
author:     "tablesheep"
date:       2021-10-04 10:00:00
header-style: text
catalog: true
tags:
    - Java
    - Spring
    - Spring AOP
    - 源码
---

> Spring AOP 方法执行过程



### JdkDynamicAopProxy

在Spring AOP中使用`JdkDynamicAopProxy`作为JDK代理的实现，它继承了`InvocationHandler`，所以在实际执行方法时会调用`invoke`方法。

```java
final class JdkDynamicAopProxy implements AopProxy, InvocationHandler, Serializable {

   /** Config used to configure this proxy. */
   private final AdvisedSupport advised;
......

   /**
    * Implementation of {@code InvocationHandler.invoke}.
    * <p>Callers will see exactly the exception thrown by the target,
    * unless a hook method throws an exception.
    */
   @Override
   @Nullable
   public Object invoke(Object proxy, Method method, Object[] args) throws Throwable {
      Object oldProxy = null;
      boolean setProxyContext = false;

      TargetSource targetSource = this.advised.targetSource;
      Object target = null;

      try {
          //equals & hashCode方法执行
         if (!this.equalsDefined && AopUtils.isEqualsMethod(method)) {
            // The target does not implement the equals(Object) method itself.
            return equals(args[0]);
         }
         else if (!this.hashCodeDefined && AopUtils.isHashCodeMethod(method)) {
            // The target does not implement the hashCode() method itself.
            return hashCode();
         }
         else if (method.getDeclaringClass() == DecoratingProxy.class) {
            // There is only getDecoratedClass() declared -> dispatch to proxy config.
            return AopProxyUtils.ultimateTargetClass(this.advised);
         }
          //目标方法来自Advised，直接使用当前advised执行
         else if (!this.advised.opaque && method.getDeclaringClass().isInterface() &&
               method.getDeclaringClass().isAssignableFrom(Advised.class)) {
            // Service invocations on ProxyConfig with the proxy config...
            return AopUtils.invokeJoinpointUsingReflection(this.advised, method, args);
         }

         Object retVal;

          //若配置可暴露当前代理，则将当前代理添加到ThreadLocal中
         if (this.advised.exposeProxy) {
            // Make invocation available if necessary.
            oldProxy = AopContext.setCurrentProxy(proxy);
            setProxyContext = true;
         }

         // Get as late as possible to minimize the time we "own" the target,
         // in case it comes from a pool.
         target = targetSource.getTarget();
         Class<?> targetClass = (target != null ? target.getClass() : null);

          //获取符合此次方法执行的AOP拦截链
         // Get the interception chain for this method.
         List<Object> chain = this.advised.getInterceptorsAndDynamicInterceptionAdvice(method, targetClass);

         // Check whether we have any advice. If we don't, we can fallback on direct
         // reflective invocation of the target, and avoid creating a MethodInvocation.
         if (chain.isEmpty()) {
            // We can skip creating a MethodInvocation: just invoke the target directly
            // Note that the final invoker must be an InvokerInterceptor so we know it does
            // nothing but a reflective operation on the target, and no hot swapping or fancy proxying.
             //链为空则通过AopUtils发射执行原本业务
            Object[] argsToUse = AopProxyUtils.adaptArgumentsIfNecessary(method, args);
            retVal = AopUtils.invokeJoinpointUsingReflection(target, method, argsToUse);
         }
         else {
             //创建ReflectiveMethodInvocation对象执行相应的AOP动作以及原本业务
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
         if (target != null && !targetSource.isStatic()) {
            // Must have come from TargetSource.
            targetSource.releaseTarget(target);
         }
         if (setProxyContext) {
            // Restore old proxy.
            AopContext.setCurrentProxy(oldProxy);
         }
      }
   }


   /**
    * Equality means interfaces, advisors and TargetSource are equal.
    * <p>The compared object may be a JdkDynamicAopProxy instance itself
    * or a dynamic proxy wrapping a JdkDynamicAopProxy instance.
    */
   @Override
   public boolean equals(@Nullable Object other) {
      if (other == this) {
         return true;
      }
      if (other == null) {
         return false;
      }

      JdkDynamicAopProxy otherProxy;
      if (other instanceof JdkDynamicAopProxy) {
         otherProxy = (JdkDynamicAopProxy) other;
      }
      else if (Proxy.isProxyClass(other.getClass())) {
         InvocationHandler ih = Proxy.getInvocationHandler(other);
         if (!(ih instanceof JdkDynamicAopProxy)) {
            return false;
         }
         otherProxy = (JdkDynamicAopProxy) ih;
      }
      else {
         // Not a valid comparison...
         return false;
      }

      // If we get here, otherProxy is the other AopProxy.
      return AopProxyUtils.equalsInProxy(this.advised, otherProxy.advised);
   }

   /**
    * Proxy uses the hash code of the TargetSource.
    */
   @Override
   public int hashCode() {
      return JdkDynamicAopProxy.class.hashCode() * 13 + this.advised.getTargetSource().hashCode();
   }

}
```



### CglibAopProxy

CGLIB动态代理主要通过Callback接口进行AOP操作，而在`CglibAopProxy`在添加`CallBack`时，对于AOP增强操作，使用了`DynamicAdvisedInterceptor`进行相关的操作

```java
class CglibAopProxy implements AopProxy, Serializable {
......

   private Callback[] getCallbacks(Class<?> rootClass) throws Exception {
......

      // Choose an "aop" interceptor (used for AOP calls).
      Callback aopInterceptor = new DynamicAdvisedInterceptor(this.advised);
......
   }

......

   /**
    * General purpose AOP callback. Used when the target is dynamic or when the
    * proxy is not frozen.
    */
   private static class DynamicAdvisedInterceptor implements MethodInterceptor, Serializable {

      private final AdvisedSupport advised;

      public DynamicAdvisedInterceptor(AdvisedSupport advised) {
         this.advised = advised;
      }

      @Override
      @Nullable
      public Object intercept(Object proxy, Method method, Object[] args, MethodProxy methodProxy) throws Throwable {
         Object oldProxy = null;
         boolean setProxyContext = false;
         Object target = null;
         TargetSource targetSource = this.advised.getTargetSource();
         try {
            if (this.advised.exposeProxy) {
               // Make invocation available if necessary.
               oldProxy = AopContext.setCurrentProxy(proxy);
               setProxyContext = true;
            }
            // Get as late as possible to minimize the time we "own" the target, in case it comes from a pool...
            target = targetSource.getTarget();
            Class<?> targetClass = (target != null ? target.getClass() : null);
             //同样是获取AOP拦截链
            List<Object> chain = this.advised.getInterceptorsAndDynamicInterceptionAdvice(method, targetClass);
            Object retVal;
            // Check whether we only have one InvokerInterceptor: that is,
            // no real advice, but just reflective invocation of the target.
            if (chain.isEmpty() && CglibMethodInvocation.isMethodProxyCompatible(method)) {
               // We can skip creating a MethodInvocation: just invoke the target directly.
               // Note that the final invoker must be an InvokerInterceptor, so we know
               // it does nothing but a reflective operation on the target, and no hot
               // swapping or fancy proxying.
               Object[] argsToUse = AopProxyUtils.adaptArgumentsIfNecessary(method, args);
               try {
                   //若链是空的则直接执行原本业务
                  retVal = methodProxy.invoke(target, argsToUse);
               }
               catch (CodeGenerationException ex) {
                  CglibMethodInvocation.logFastClassGenerationFailure(method);
                  retVal = AopUtils.invokeJoinpointUsingReflection(target, method, argsToUse);
               }
            }
            else {
                //否则创建CglibMethodInvocation进行处理，而CglibMethodInvocation继承至ReflectiveMethodInvocation
               // We need to create a method invocation...
               retVal = new CglibMethodInvocation(proxy, target, method, args, targetClass, chain, methodProxy).proceed();
            }
            retVal = processReturnType(proxy, target, method, retVal);
            return retVal;
         }
         finally {
            if (target != null && !targetSource.isStatic()) {
               targetSource.releaseTarget(target);
            }
            if (setProxyContext) {
               // Restore old proxy.
               AopContext.setCurrentProxy(oldProxy);
            }
         }
      }

      @Override
      public boolean equals(@Nullable Object other) {
         return (this == other ||
               (other instanceof DynamicAdvisedInterceptor &&
                     this.advised.equals(((DynamicAdvisedInterceptor) other).advised)));
      }

      /**
       * CGLIB uses this to drive proxy creation.
       */
      @Override
      public int hashCode() {
         return this.advised.hashCode();
      }
   }


......

}
```



### 总结

无论是`CGLIB`还是`JDK`代理，在执行方法时大体上会先获取`AOP`相关方法拦截链，然后通过`ReflectiveMethodInvocation`进行相应的方法执行，下面看看这两步的具体执行。



### AdvisorChain链获取

#### AdvisedSupport

`AdvisedSupport`主要是通过`AdvisorChainFactory`去获取AOP方法拦截链，同时在`AdvisedSupport`中用Map作为缓存，提升了每次调用的性能。

```java
public class AdvisedSupport extends ProxyConfig implements Advised {
......

   /** The AdvisorChainFactory to use. */
   AdvisorChainFactory advisorChainFactory = new DefaultAdvisorChainFactory();

   /** Cache with Method as key and advisor chain List as value. */
   private transient Map<MethodCacheKey, List<Object>> methodCache;

......


   /**
    * Determine a list of {@link org.aopalliance.intercept.MethodInterceptor} objects
    * for the given method, based on this configuration.
    * @param method the proxied method
    * @param targetClass the target class
    * @return a List of MethodInterceptors (may also include InterceptorAndDynamicMethodMatchers)
    */
   public List<Object> getInterceptorsAndDynamicInterceptionAdvice(Method method, @Nullable Class<?> targetClass) {
      MethodCacheKey cacheKey = new MethodCacheKey(method);
      List<Object> cached = this.methodCache.get(cacheKey);
      if (cached == null) {
         cached = this.advisorChainFactory.getInterceptorsAndDynamicInterceptionAdvice(
               this, method, targetClass);
         this.methodCache.put(cacheKey, cached);
      }
      return cached;
   }
......
}
```



#### DefaultAdvisorChainFactory

`DefaultAdvisorChainFactory`首先根据`Advisor`中的`Pointcut`或者`ClassFilter`进行判断，如果此`Advisor`适用，则会使用`AdvisorAdapterRegistry`将`Advisor`中的`Advice`适配成`MethodInterceptor`

```java
public class DefaultAdvisorChainFactory implements AdvisorChainFactory, Serializable {

   @Override
   public List<Object> getInterceptorsAndDynamicInterceptionAdvice(
         Advised config, Method method, @Nullable Class<?> targetClass) {

      // This is somewhat tricky... We have to process introductions first,
      // but we need to preserve order in the ultimate list.
      AdvisorAdapterRegistry registry = GlobalAdvisorAdapterRegistry.getInstance();
      Advisor[] advisors = config.getAdvisors();
      List<Object> interceptorList = new ArrayList<>(advisors.length);
      Class<?> actualClass = (targetClass != null ? targetClass : method.getDeclaringClass());
      Boolean hasIntroductions = null;

      for (Advisor advisor : advisors) {
         if (advisor instanceof PointcutAdvisor) {
             //PointcutAdvisor处理
            // Add it conditionally.
            PointcutAdvisor pointcutAdvisor = (PointcutAdvisor) advisor;
             //使用Pointcut进行判断，首先根据的ClassFilter进行判断，在根据MethodMatcher判断
            if (config.isPreFiltered() || pointcutAdvisor.getPointcut().getClassFilter().matches(actualClass)) {
               MethodMatcher mm = pointcutAdvisor.getPointcut().getMethodMatcher();
               boolean match;
               if (mm instanceof IntroductionAwareMethodMatcher) {
                  if (hasIntroductions == null) {
                     hasIntroductions = hasMatchingIntroductions(advisors, actualClass);
                  }
                  match = ((IntroductionAwareMethodMatcher) mm).matches(method, actualClass, hasIntroductions);
               }
               else {
                  match = mm.matches(method, actualClass);
               }
               if (match) {
                   //Pointcut判断通过，则会使用AdvisorAdapterRegistry获取所有的MethodInterceptor
                  MethodInterceptor[] interceptors = registry.getInterceptors(advisor);
                  if (mm.isRuntime()) {
                      //如果该MethodMatcher可以根据运行时参数进行判断，则把MethodInterceptor和MethodMatcher封装成InterceptorAndDynamicMethodMatcher
                     // Creating a new object instance in the getInterceptors() method
                     // isn't a problem as we normally cache created chains.
                     for (MethodInterceptor interceptor : interceptors) {
                        interceptorList.add(new InterceptorAndDynamicMethodMatcher(interceptor, mm));
                     }
                  }
                  else {
                     interceptorList.addAll(Arrays.asList(interceptors));
                  }
               }
            }
         }
         else if (advisor instanceof IntroductionAdvisor) {
             //针对IntroductionAdvisor处理
            IntroductionAdvisor ia = (IntroductionAdvisor) advisor;
            if (config.isPreFiltered() || ia.getClassFilter().matches(actualClass)) {
               Interceptor[] interceptors = registry.getInterceptors(advisor);
               interceptorList.addAll(Arrays.asList(interceptors));
            }
         }
         else {
            Interceptor[] interceptors = registry.getInterceptors(advisor);
            interceptorList.addAll(Arrays.asList(interceptors));
         }
      }

      return interceptorList;
   }

   /**
    * Determine whether the Advisors contain matching introductions.
    */
   private static boolean hasMatchingIntroductions(Advisor[] advisors, Class<?> actualClass) {
      for (Advisor advisor : advisors) {
         if (advisor instanceof IntroductionAdvisor) {
            IntroductionAdvisor ia = (IntroductionAdvisor) advisor;
            if (ia.getClassFilter().matches(actualClass)) {
               return true;
            }
         }
      }
      return false;
   }

}
```



##### DefaultAdvisorAdapterRegistry

`DefaultAdvisorAdapterRegistry`注册了`Before、After、Throws`的`AdvisorAdapter`，能将对应的`Advice`适配成`MethodInterceptor`，通过它将保证所有的AOP增强操作都统一成`MethodInterceptor`。

```java
public class DefaultAdvisorAdapterRegistry implements AdvisorAdapterRegistry, Serializable {

   private final List<AdvisorAdapter> adapters = new ArrayList<>(3);


   /**
    * Create a new DefaultAdvisorAdapterRegistry, registering well-known adapters.
    */
   public DefaultAdvisorAdapterRegistry() {
      registerAdvisorAdapter(new MethodBeforeAdviceAdapter());
      registerAdvisorAdapter(new AfterReturningAdviceAdapter());
      registerAdvisorAdapter(new ThrowsAdviceAdapter());
   }
......

   @Override
   public MethodInterceptor[] getInterceptors(Advisor advisor) throws UnknownAdviceTypeException {
      List<MethodInterceptor> interceptors = new ArrayList<>(3);
      Advice advice = advisor.getAdvice();
      if (advice instanceof MethodInterceptor) {
         interceptors.add((MethodInterceptor) advice);
      }
      for (AdvisorAdapter adapter : this.adapters) {
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



##### InterceptorAndDynamicMethodMatcher

`InterceptorAndDynamicMethodMatcher`就是将`MethodInterceptor`跟`MethodMatcher`组合起来，方便运行时判断。

```java
class InterceptorAndDynamicMethodMatcher {

   final MethodInterceptor interceptor;

   final MethodMatcher methodMatcher;

   public InterceptorAndDynamicMethodMatcher(MethodInterceptor interceptor, MethodMatcher methodMatcher) {
      this.interceptor = interceptor;
      this.methodMatcher = methodMatcher;
   }

}
```



### 方法的执行

#### ReflectiveMethodInvocation

`ReflectiveMethodInvocation`在执行方法时，通过递归遍历interceptor链表进行相应的增强操作，最后当遍历完后在调用原本的方法执行原本都业务。

```java
public class ReflectiveMethodInvocation implements ProxyMethodInvocation, Cloneable {

......

   /**
    * List of MethodInterceptor and InterceptorAndDynamicMethodMatcher
    * that need dynamic checks.
    */
   protected final List<?> interceptorsAndDynamicMethodMatchers;

   /**
    * Index from 0 of the current interceptor we're invoking.
    * -1 until we invoke: then the current interceptor.
    */
   private int currentInterceptorIndex = -1;

......


   @Override
   @Nullable
   public Object proceed() throws Throwable {
      // We start with an index of -1 and increment early.
      if (this.currentInterceptorIndex == this.interceptorsAndDynamicMethodMatchers.size() - 1) {
          //当Interceptor链执行完后调用原本业务方法
         return invokeJoinpoint();
      }

      Object interceptorOrInterceptionAdvice =
            this.interceptorsAndDynamicMethodMatchers.get(++this.currentInterceptorIndex);
      if (interceptorOrInterceptionAdvice instanceof InterceptorAndDynamicMethodMatcher) {
          //InterceptorAndDynamicMethodMatcher执行，需要先根据参数判断是否适用
         // Evaluate dynamic method matcher here: static part will already have
         // been evaluated and found to match.
         InterceptorAndDynamicMethodMatcher dm =
               (InterceptorAndDynamicMethodMatcher) interceptorOrInterceptionAdvice;
         Class<?> targetClass = (this.targetClass != null ? this.targetClass : this.method.getDeclaringClass());
         if (dm.methodMatcher.matches(this.method, targetClass, this.arguments)) {
            return dm.interceptor.invoke(this);
         }
         else {
            // Dynamic matching failed.
            // Skip this interceptor and invoke the next in the chain.
            return proceed();
         }
      }
      else {
         // It's an interceptor, so we just invoke it: The pointcut will have
         // been evaluated statically before this object was constructed.
          //普通MethodInterceptor执行，将自身（MethodInvocation）传递进去，形成递归调用
         return ((MethodInterceptor) interceptorOrInterceptionAdvice).invoke(this);
      }
   }

   /**
    * Invoke the joinpoint using reflection.
    * Subclasses can override this to use custom invocation.
    * @return the return value of the joinpoint
    * @throws Throwable if invoking the joinpoint resulted in an exception
    */
   @Nullable
   protected Object invokeJoinpoint() throws Throwable {
       //使用反射执行原有业务
      return AopUtils.invokeJoinpointUsingReflection(this.target, this.method, this.arguments);
   }

......

}
```



#### CglibMethodInvocation

`CglibMethodInvocation`主要重写了`invokeJoinpoint`方法进行，其余大部分逻辑都在`ReflectiveMethodInvocation`中

```java
/**
 * Implementation of AOP Alliance MethodInvocation used by this AOP proxy.
 */
private static class CglibMethodInvocation extends ReflectiveMethodInvocation {

   @Nullable
   private final MethodProxy methodProxy;

   public CglibMethodInvocation(Object proxy, @Nullable Object target, Method method,
         Object[] arguments, @Nullable Class<?> targetClass,
         List<Object> interceptorsAndDynamicMethodMatchers, MethodProxy methodProxy) {

      super(proxy, target, method, arguments, targetClass, interceptorsAndDynamicMethodMatchers);

      // Only use method proxy for public methods not derived from java.lang.Object
      this.methodProxy = (isMethodProxyCompatible(method) ? methodProxy : null);
   }

   @Override
   @Nullable
   public Object proceed() throws Throwable {
      try {
         return super.proceed();
      }
      catch (RuntimeException ex) {
         throw ex;
      }
      catch (Exception ex) {
         if (ReflectionUtils.declaresException(getMethod(), ex.getClass()) ||
               KotlinDetector.isKotlinType(getMethod().getDeclaringClass())) {
            // Propagate original exception if declared on the target method
            // (with callers expecting it). Always propagate it for Kotlin code
            // since checked exceptions do not have to be explicitly declared there.
            throw ex;
         }
         else {
            // Checked exception thrown in the interceptor but not declared on the
            // target method signature -> apply an UndeclaredThrowableException,
            // aligned with standard JDK dynamic proxy behavior.
            throw new UndeclaredThrowableException(ex);
         }
      }
   }

   /**
    * Gives a marginal performance improvement versus using reflection to
    * invoke the target when invoking public methods.
    */
   @Override
   protected Object invokeJoinpoint() throws Throwable {
      if (this.methodProxy != null) {
         try {
            return this.methodProxy.invoke(this.target, this.arguments);
         }
         catch (CodeGenerationException ex) {
            logFastClassGenerationFailure(this.method);
         }
      }
      return super.invokeJoinpoint();
   }

   static boolean isMethodProxyCompatible(Method method) {
      return (Modifier.isPublic(method.getModifiers()) &&
            method.getDeclaringClass() != Object.class && !AopUtils.isEqualsMethod(method) &&
            !AopUtils.isHashCodeMethod(method) && !AopUtils.isToStringMethod(method));
   }

   static void logFastClassGenerationFailure(Method method) {
      if (logger.isDebugEnabled()) {
         logger.debug("Failed to generate CGLIB fast class for method: " + method);
      }
   }
}
```



### 总结

`Advisor`中的`Advice`最终会被适配成`MethodInterceptor`或者`InterceptorAndDynamicMethodMatcher`（针对需要运行时判断的），而后在执行时通过`MethodInvocation`进行递归执行，完成AOP增强操作以及原本的业务方法。



## References

- <https://github.com/spring-projects/spring-framework/tree/main/spring-aop/src/main/java/org/springframework/aop>