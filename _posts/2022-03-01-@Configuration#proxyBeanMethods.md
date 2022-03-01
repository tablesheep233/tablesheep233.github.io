---
layout:     post
title:      "@Configuration#proxyBeanMethods分析"
subtitle:   " \"@Configuration#proxyBeanMethods分析\""
author:     "tablesheep"
date:       2022-03-01 20:00:00
header-style: text
catalog: true
tags:
- Java
- Spring
- 源码
---


> 看源码的时候发现Spring 一些配置类上面经常有`@Configuration(proxyBeanMethods=false)`，出于好奇瞅了瞅，记录一下。



## @Configuration注释

```java
@Target(ElementType.TYPE)
@Retention(RetentionPolicy.RUNTIME)
@Documented
@Component
public @interface Configuration {
......

   /**
    * Specify whether {@code @Bean} methods should get proxied in order to enforce
    * bean lifecycle behavior, e.g. to return shared singleton bean instances even
    * in case of direct {@code @Bean} method calls in user code. This feature
    * requires method interception, implemented through a runtime-generated CGLIB
    * subclass which comes with limitations such as the configuration class and
    * its methods not being allowed to declare {@code final}.
    * <p>The default is {@code true}, allowing for 'inter-bean references' via direct
    * method calls within the configuration class as well as for external calls to
    * this configuration's {@code @Bean} methods, e.g. from another configuration class.
    * If this is not needed since each of this particular configuration's {@code @Bean}
    * methods is self-contained and designed as a plain factory method for container use,
    * switch this flag to {@code false} in order to avoid CGLIB subclass processing.
    * <p>Turning off bean method interception effectively processes {@code @Bean}
    * methods individually like when declared on non-{@code @Configuration} classes,
    * a.k.a. "@Bean Lite Mode" (see {@link Bean @Bean's javadoc}). It is therefore
    * behaviorally equivalent to removing the {@code @Configuration} stereotype.
    * @since 5.2
    */
   boolean proxyBeanMethods() default true;

}
```

根据注释，`proxyBeanMethods`作用如下

- true，当`@Bean`注入的类调用另一个`@Bean`注释的方法获取需要的类时，不会直接调用方法创建对象，而是会通过`CGLIB`子类从Spring容器中获取，从而保证`Bean`的生命周期以及`Bean`之间正确的引用
- false，表示不需要代理，会直接调用方法创建需要的实例，`Bean`之间不会有关联
- 需要注意的是当为true时，需要保证`CGLIB`能正确代理方法，比如方法不能为`final`、`private`、`static`

同时还提到了`@Bean Lite Mode`，看一下`@Bean`的注释

## @Bean注释

```java
package org.springframework.context.annotation;

import java.lang.annotation.Documented;
import java.lang.annotation.ElementType;
import java.lang.annotation.Retention;
import java.lang.annotation.RetentionPolicy;
import java.lang.annotation.Target;

import org.springframework.beans.factory.annotation.Autowire;
import org.springframework.beans.factory.support.AbstractBeanDefinition;
import org.springframework.core.annotation.AliasFor;

/**
 * ......
 * <h3>{@code @Bean} Methods in {@code @Configuration} Classes</h3>
 *
 * <p>Typically, {@code @Bean} methods are declared within {@code @Configuration}
 * classes. In this case, bean methods may reference other {@code @Bean} methods in the
 * same class by calling them <i>directly</i>. This ensures that references between beans
 * are strongly typed and navigable. Such so-called <em>'inter-bean references'</em> are
 * guaranteed to respect scoping and AOP semantics, just like {@code getBean()} lookups
 * would. These are the semantics known from the original 'Spring JavaConfig' project
 * which require CGLIB subclassing of each such configuration class at runtime. As a
 * consequence, {@code @Configuration} classes and their factory methods must not be
 * marked as final or private in this mode. For example:
 *
 * <pre class="code">
 * &#064;Configuration
 * public class AppConfig {
 *
 *     &#064;Bean
 *     public FooService fooService() {
 *         return new FooService(fooRepository());
 *     }
 *
 *     &#064;Bean
 *     public FooRepository fooRepository() {
 *         return new JdbcFooRepository(dataSource());
 *     }
 *
 *     // ...
 * }</pre>
 *
 * <h3>{@code @Bean} <em>Lite</em> Mode</h3>
 *
 * <p>{@code @Bean} methods may also be declared within classes that are <em>not</em>
 * annotated with {@code @Configuration}. For example, bean methods may be declared
 * in a {@code @Component} class or even in a <em>plain old class</em>. In such cases,
 * a {@code @Bean} method will get processed in a so-called <em>'lite'</em> mode.
 *
 * <p>Bean methods in <em>lite</em> mode will be treated as plain <em>factory
 * methods</em> by the container (similar to {@code factory-method} declarations
 * in XML), with scoping and lifecycle callbacks properly applied. The containing
 * class remains unmodified in this case, and there are no unusual constraints for
 * the containing class or the factory methods.
 *
 * <p>In contrast to the semantics for bean methods in {@code @Configuration} classes,
 * <em>'inter-bean references'</em> are not supported in <em>lite</em> mode. Instead,
 * when one {@code @Bean}-method invokes another {@code @Bean}-method in <em>lite</em>
 * mode, the invocation is a standard Java method invocation; Spring does not intercept
 * the invocation via a CGLIB proxy. This is analogous to inter-{@code @Transactional}
 * method calls where in proxy mode, Spring does not intercept the invocation &mdash;
 * Spring does so only in AspectJ mode.
 *
 * <p>For example:
 *
 * <pre class="code">
 * &#064;Component
 * public class Calculator {
 *     public int sum(int a, int b) {
 *         return a+b;
 *     }
 *
 *     &#064;Bean
 *     public MyBean myBean() {
 *         return new MyBean();
 *     }
 * }</pre>
 *
 * <h3>Bootstrapping</h3>
 *
 * <p>See the @{@link Configuration} javadoc for further details including how to bootstrap
 * the container using {@link AnnotationConfigApplicationContext} and friends.
 *
 * <h3>{@code BeanFactoryPostProcessor}-returning {@code @Bean} methods</h3>
 *
 * <p>Special consideration must be taken for {@code @Bean} methods that return Spring
 * {@link org.springframework.beans.factory.config.BeanFactoryPostProcessor BeanFactoryPostProcessor}
 * ({@code BFPP}) types. Because {@code BFPP} objects must be instantiated very early in the
 * container lifecycle, they can interfere with processing of annotations such as {@code @Autowired},
 * {@code @Value}, and {@code @PostConstruct} within {@code @Configuration} classes. To avoid these
 * lifecycle issues, mark {@code BFPP}-returning {@code @Bean} methods as {@code static}. For example:
 *
 * <pre class="code">
 *     &#064;Bean
 *     public static PropertySourcesPlaceholderConfigurer pspc() {
 *         // instantiate, configure and return pspc ...
 *     }
 * </pre>
 *
 * By marking this method as {@code static}, it can be invoked without causing instantiation of its
 * declaring {@code @Configuration} class, thus avoiding the above-mentioned lifecycle conflicts.
 * Note however that {@code static} {@code @Bean} methods will not be enhanced for scoping and AOP
 * semantics as mentioned above. This works out in {@code BFPP} cases, as they are not typically
 * referenced by other {@code @Bean} methods. As a reminder, an INFO-level log message will be
 * issued for any non-static {@code @Bean} methods having a return type assignable to
 * {@code BeanFactoryPostProcessor}.
 *......
 */
@Target({ElementType.METHOD, ElementType.ANNOTATION_TYPE})
@Retention(RetentionPolicy.RUNTIME)
@Documented
public @interface Bean {
......
}
```

根据@Bean的注释描述

- Lite Mode ，精简模式，当@Bean用在`@Component`类或普通类中时，会以FactoryMethod的方式处理（就是正常的Bean生命周期，见`AbstractBeanDefinition#factoryMethodName`），并不会使用`CGLIB`代理，也就是说引用@Bean的方法获取实例时会以方法直接调用的形式直接返回，`@Configuration#proxyBeanMethods=false`就相当于Lite Mode
- 与之对应的是 Full Mode，`@Configuration#proxyBeanMethods=true`，这两种模式的定义见`ConfigurationClassUtils`，其中的`checkConfigurationClassCandidate`方法有确定mode的逻辑

> 总结一下，就是@Configuration 中 @Bean的使用与其他类中不一样，@Configuration中默认情况会将@Bean方法通过CGLIB代理，实现@Bean方法之间的引用都出自容器，保证Bean之间的正确引用关系。而proxyBeanMethods=false则会使@Bean的处理行为回到Lite Mode，即@Bean注解的方法注入的对象会根据Bean生命周期创建，但是@Bean方法相互引用则是直接调用方法创建



> `@Bean`注释中还建议当使用`@Bean`注入`BeanFactoryPostProcessor`类型的Bean时，建议使用`static`关键字标注`BeanFactoryPostProcessor`的方法，这里面关于`static`很有说法，挖个坑



## 测试

**测试代码**

```java
import org.springframework.context.annotation.AnnotationConfigApplicationContext;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;

import javax.annotation.PostConstruct;

public class ConfigurationProxyBeanMethodMain {
    
    // cglib 代理类 存放路径
    static {
        System.setProperty(DebuggingClassWriter.DEBUG_LOCATION_PROPERTY, "D:\\ConfigurationProxyBeanMethod");
    }
    
    public static void main(String[] args) {
        AnnotationConfigApplicationContext context = new AnnotationConfigApplicationContext();
        context.register(ConfigurationProxyBeanMethods.class);
        context.refresh();
        context.close();
    }

//    @Configuration(proxyBeanMethods = false)
    @Configuration
    public static class ConfigurationProxyBeanMethods {


        @Bean
        public BBean bBean() {
            return new BBean(aBean());
        }

        @Bean
        public ABean aBean() {
            return new ABean();
        }

    }

    public static class ABean {

        public ABean() {
            System.out.println("A Bean construct");
        }

        @PostConstruct
        public void init() {
            System.out.println("A Bean init");
        }
    }

    public static class BBean {

        public BBean(ABean aBean) {
            System.out.println("B Bean construct");
        }

        @PostConstruct
        public void init() {
            System.out.println("B Bean init");
        }
    }
}
```

**执行结果**

```console
###@Configuration 执行结果， ABean 只会new一次，在BBean构造时需要的ABean是从容器中获取的
A Bean construct
A Bean init
B Bean construct
B Bean init

###@Configuration(proxyBeanMethods = false) 执行结果， ABean会new两次，创建BBean时的ABean是直接调用aBean()方法创建的
A Bean construct
B Bean construct
B Bean init
A Bean construct
A Bean init
```



## GCLIB代理实现

代理后的方法如下

```java
final BBean CGLIB$bBean$0() {
    return super.bBean();
}

public final BBean bBean() {
    MethodInterceptor var10000 = this.CGLIB$CALLBACK_0;
    if (var10000 == null) {
        CGLIB$BIND_CALLBACKS(this);
        var10000 = this.CGLIB$CALLBACK_0;
    }

    return var10000 != null ? (BBean)var10000.intercept(this, CGLIB$bBean$0$Method, CGLIB$emptyArgs, CGLIB$bBean$0$Proxy) : super.bBean();
}

final ABean CGLIB$aBean$1() {
    return super.aBean();
}

public final ABean aBean() {
    MethodInterceptor var10000 = this.CGLIB$CALLBACK_0;
    if (var10000 == null) {
        CGLIB$BIND_CALLBACKS(this);
        var10000 = this.CGLIB$CALLBACK_0;
    }

    return var10000 != null ? (ABean)var10000.intercept(this, CGLIB$aBean$1$Method, CGLIB$emptyArgs, CGLIB$aBean$1$Proxy) : super.aBean();
}
```

通过断点可以找出对应的`MethodInterceptor`为`ConfigurationClassEnhancer.BeanMethodInterceptor`，看具体的`intercept`方法

```java
@Override
@Nullable
public Object intercept(Object enhancedConfigInstance, Method beanMethod, Object[] beanMethodArgs,
         MethodProxy cglibMethodProxy) throws Throwable {

   ConfigurableBeanFactory beanFactory = getBeanFactory(enhancedConfigInstance);
   String beanName = BeanAnnotationHelper.determineBeanNameFor(beanMethod);

......

    //判断是否当前调用的方法是否是容器调用的
   if (isCurrentlyInvokedFactoryMethod(beanMethod)) {
      // The factory is calling the bean method in order to instantiate and register the bean
      // (i.e. via a getBean() call) -> invoke the super implementation of the method to actually
      // create the bean instance.
      if (logger.isInfoEnabled() &&
            BeanFactoryPostProcessor.class.isAssignableFrom(beanMethod.getReturnType())) {
         logger.info(String.format("@Bean method %s.%s is non-static and returns an object " +
                     "assignable to Spring's BeanFactoryPostProcessor interface. This will " +
                     "result in a failure to process annotations such as @Autowired, " +
                     "@Resource and @PostConstruct within the method's declaring " +
                     "@Configuration class. Add the 'static' modifier to this method to avoid " +
                     "these container lifecycle issues; see @Bean javadoc for complete details.",
               beanMethod.getDeclaringClass().getSimpleName(), beanMethod.getName()));
      }
       //是的话直接执行父类方法，也就是原本的方法
      return cglibMethodProxy.invokeSuper(enhancedConfigInstance, beanMethodArgs);
   }
 //不是的话就会调用resolveBeanReference去容器获取依赖
   return resolveBeanReference(beanMethod, beanMethodArgs, beanFactory, beanName);
}

......
    
private boolean isCurrentlyInvokedFactoryMethod(Method method) {
    //其实就是通过ThreadLocal存放了容器调用时的方法，后续方法栈上面的方法都跟它比较，见org.springframework.beans.factory.support.SimpleInstantiationStrategy
   Method currentlyInvoked = SimpleInstantiationStrategy.getCurrentlyInvokedFactoryMethod();
   return (currentlyInvoked != null && method.getName().equals(currentlyInvoked.getName()) &&
         Arrays.equals(method.getParameterTypes(), currentlyInvoked.getParameterTypes()));
}
```

套用到我们的测试就是（以下方法需结合代理类看）

> 1. createBean创建BBean
> 2. 容器将currentlyInvokedFactoryMethod 设置为 bBean()
> 3. bBean() 增强方法
> 4. cglibMethodProxy.invokeSuper
> 5. CGLIB$bBean$0() (原 bBean方法)
> 6. aBean() 增强方法
> 7. resolveBeanReference() 触发依赖查询导致先注入ABean，容器通过createBean创建ABean（这里有个点要注意，SimpleInstantiationStrategy会先将currentlyInvokedFactoryMethod设置为aBean()方法，好一个递归）
> 8. cglibMethodProxy.invokeSuper
> 9. CGLIB$aBean$1() (原 aBean方法) 创建ABean 

