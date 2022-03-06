---
layout:     post
title:      "static 影响 @Bean 与 BeanFactoryPostProcessor 分析"
subtitle:   " \"static 影响 @Bean 与 BeanFactoryPostProcessor 分析\""
author:     "tablesheep"
date:       2022-03-05 20:00:00
header-style: text
catalog: true
tags:
- Java
- Spring
- 源码
---

> Spring Version：5.3.x



**@Bean配置两种模式**

- Full Mode：在`@Configuration`中默认的mode，`@Bean`方法都会被CGLIB增强，以达到Bean之间的正确引用

- Lite Mode：在`@Configuration(proxyBeanMethods=false)`或`@Component`等注解下，`@Bean`方法不会被增强，`@Bean`方法之间调用是正常的方法调用



## static 关键字与@Bean与部分BeanFactoryPostProcessor

### 情况说明

当`@Configuration`配置类中有`@Bean`注入的部分（为什么说是部分呢，请看分析）`BeanFactoryPostProcessor`时，会导致Full模式失效，造成Bean之间引用的错误

### 情况分析

```java
import org.slf4j.Logger;
import org.slf4j.LoggerFactory;
import org.springframework.beans.BeansException;
import org.springframework.beans.factory.config.BeanFactoryPostProcessor;
import org.springframework.beans.factory.config.ConfigurableListableBeanFactory;
import org.springframework.beans.factory.support.BeanDefinitionRegistry;
import org.springframework.beans.factory.support.BeanDefinitionRegistryPostProcessor;
import org.springframework.context.annotation.AnnotationConfigApplicationContext;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;
import javax.annotation.PostConstruct;

@Configuration(proxyBeanMethods = false)
public class StaticConfigurationBean {
    private static final Logger LOG = LoggerFactory.getLogger(StaticConfigurationBean.class);
    
    public static void main(String[] args) {
        AnnotationConfigApplicationContext context = new AnnotationConfigApplicationContext();
        context.register(StaticConfigurationBean.class);
        context.refresh();
        context.close();
    }

    @Configuration
    public static class Config {
        public Config() {
            LOG.info("Config construct");
        }

        @PostConstruct
        public void init() {
            LOG.info("Config init");
        }
        
        //    @Bean
//    public BeanDefinitionRegistryPostProcessor customPostProcessor() {  //情况一

//        @Bean
//        public static BeanDefinitionRegistryPostProcessor customPostProcessor() {  //情况二

//        @Bean
//        public BeanFactoryPostProcessor customPostProcessor() {  //情况三
            
        @Bean
        public static BeanFactoryPostProcessor customPostProcessor() {  //情况四
            LOG.info("customPostProcessor construct");
            return new BeanDefinitionRegistryPostProcessor() {
                @Override
                public void postProcessBeanDefinitionRegistry(BeanDefinitionRegistry registry) throws BeansException {
                    registry.removeBeanDefinition("c");
                }

                @Override
                public void postProcessBeanFactory(ConfigurableListableBeanFactory beanFactory) throws BeansException {
                }
            };
        }

        @Bean
        public B b() {
            return new B(a());
        }

        @Bean
        public A a() {
            return new A();
        }

        @Bean
        public C c() {
            return new C();
        }

        public static class A {
            public A() {
                LOG.info("A construct a is " + this.hashCode());
            }

            @PostConstruct
            public void init() {
                LOG.info("A init");
            }
        }

        public static class B {
            public B(A a) {
                LOG.info("B construct B.a is " + a.hashCode());
            }

            @PostConstruct
            public void init() {
                LOG.info("B init");
            }
        }

        public static class C {
            public C() {
                LOG.info("C construct");
            }
        }
    }
}
```



> 先提一嘴
>
> AbstractApplicationContext#invokeBeanFactoryPostProcessors会执行所有的BeanFactoryPostProcessor，其中BeanDefinitionRegistryPostProcessor#postProcessBeanDefinitionRegistry会先执行，负责扫描所有的BeanDefinition，见PostProcessorRegistrationDelegate。
>
> 配置类的BeanDefinition注册见ConfigurationClassPostProcessor、ConfigurationClassBeanDefinitionReader，@Bean方法的BeanDefinition见ConfigurationClassBeanDefinitionReader#loadBeanDefinitionsForBeanMethod，其中AbstractBeanDefinition#factoryBeanName 是外部类名称，AbstractBeanDefinition#factoryMethodName是方法名。
>
> ConstructorResolver#instantiateUsingFactoryMethod 负责FactoryMethod方式的Bean的实例化（@Bean）



#### 情况一

使用`@Bean`注入`BeanDefinitionRegistryPostProcessor`会导致`@Bean`方法间的引用失效，A类实例化了两次，并且注入B类的A实例并不是容器中的那个，而且Spring 给出了对应的警告说我们的Config配置类太早实例化了，同时我们Config类的生命周期执行不正常（`@PostConstruct`方法没执行）。 不过customPostProcessor的功能正常执行，C类的`BeanDefinition`被移除了。

```console
10:31:39.311 [main] INFO org.table.config.staticwithconfig.StaticConfigurationBean - Config construct
10:31:39.311 [main] INFO org.table.config.staticwithconfig.StaticConfigurationBean - customPostProcessor construct
10:31:39.316 [main] INFO org.springframework.context.annotation.ConfigurationClassPostProcessor - Cannot enhance @Configuration bean definition 'org.table.config.staticwithconfig.StaticConfigurationBean$Config' since its singleton instance has been created too early. The typical cause is a non-static @Bean method with a BeanDefinitionRegistryPostProcessor return type: Consider declaring such methods as 'static'.
10:31:39.386 [main] DEBUG org.springframework.beans.factory.support.DefaultListableBeanFactory - Creating shared instance of singleton bean 'b'
10:31:39.393 [main] INFO org.table.config.staticwithconfig.StaticConfigurationBean - A construct a is 1230955136
10:31:39.394 [main] INFO org.table.config.staticwithconfig.StaticConfigurationBean - B construct B.a is 1230955136
10:31:39.394 [main] INFO org.table.config.staticwithconfig.StaticConfigurationBean - B init
10:31:39.394 [main] DEBUG org.springframework.beans.factory.support.DefaultListableBeanFactory - Creating shared instance of singleton bean 'a'
10:31:39.394 [main] INFO org.table.config.staticwithconfig.StaticConfigurationBean - A construct a is 1107985860
10:31:39.395 [main] INFO org.table.config.staticwithconfig.StaticConfigurationBean - A init
```

**原因分析：**为什么Full Mode失效了呢，根据Spring警告可知，是由于Config类过早实例化导致，那么它为什么会过早实例化？因为customPostProcessor的`BeanDefinition`中`factoryBeanName`不为null

```java
class ConstructorResolver {
......
   public BeanWrapper instantiateUsingFactoryMethod(
         String beanName, RootBeanDefinition mbd, @Nullable Object[] explicitArgs) {

      BeanWrapperImpl bw = new BeanWrapperImpl();
      this.beanFactory.initBeanWrapper(bw);

      Object factoryBean;
      Class<?> factoryClass;
      boolean isStatic;

      String factoryBeanName = mbd.getFactoryBeanName();
      if (factoryBeanName != null) {
          //factoryBeanName 不为null 会先触发factoryBeanName Bean 的创建
         if (factoryBeanName.equals(beanName)) {
            throw new BeanDefinitionStoreException(mbd.getResourceDescription(), beanName,
                  "factory-bean reference points back to the same bean definition");
         }
         factoryBean = this.beanFactory.getBean(factoryBeanName);
         if (mbd.isSingleton() && this.beanFactory.containsSingleton(beanName)) {
            throw new ImplicitlyAppearedSingletonException();
         }
         this.beanFactory.registerDependentBean(factoryBeanName, beanName);
         factoryClass = factoryBean.getClass();
         isStatic = false;
      }
      ......
   }
}
```

那么为什么`factoryBeanName`会有值呢？因为方法不是static的，在创建`BeanDefinition`时，若果方法是static的不会设置`factoryBeanName`

```java
class ConfigurationClassBeanDefinitionReader {
......

   /**
    * Read the given {@link BeanMethod}, registering bean definitions
    * with the BeanDefinitionRegistry based on its contents.
    */
   @SuppressWarnings("deprecation")  // for RequiredAnnotationBeanPostProcessor.SKIP_REQUIRED_CHECK_ATTRIBUTE
   private void loadBeanDefinitionsForBeanMethod(BeanMethod beanMethod) {
      ......
          
      if (metadata.isStatic()) {
         // static @Bean method
         if (configClass.getMetadata() instanceof StandardAnnotationMetadata) {
            beanDef.setBeanClass(((StandardAnnotationMetadata) configClass.getMetadata()).getIntrospectedClass());
         }
         else {
            beanDef.setBeanClassName(configClass.getMetadata().getClassName());
         }
         beanDef.setUniqueFactoryMethodName(methodName);
      }
      else {
         // instance @Bean method
         beanDef.setFactoryBeanName(configClass.getBeanName());
         beanDef.setUniqueFactoryMethodName(methodName);
      }
       ......
   }
    ......
}
```

>  至于是如何判断是否过早实例化，其实就是在准备增强类前，判断一下容器是否已经存在对应的Bean，见ConfigurationClassPostProcessor#enhanceConfigurationClasses

至于为什么Config类的生命周期执行不正常，很明显它初始化的时候`BeanPostProcessor`还没实例化，当然就不正常了。



#### 情况二

加入`static`后会发现一切回复正常，customPostProcessor的实例化不会触发Config实例化影响Bean注入顺序

```console
10:33:52.679 [main] INFO org.table.config.staticwithconfig.StaticConfigurationBean - customPostProcessor construct
10:33:52.749 [main] INFO org.table.config.staticwithconfig.StaticConfigurationBean - Config construct
10:52:17.713 [main] INFO org.table.config.staticwithconfig.StaticConfigurationBean - Config init
10:33:52.749 [main] DEBUG org.springframework.beans.factory.support.DefaultListableBeanFactory - Creating shared instance of singleton bean 'b'
10:33:52.758 [main] DEBUG org.springframework.beans.factory.support.DefaultListableBeanFactory - Creating shared instance of singleton bean 'a'
10:33:52.765 [main] INFO org.table.config.staticwithconfig.StaticConfigurationBean - A construct a is 1801942731
10:33:52.766 [main] INFO org.table.config.staticwithconfig.StaticConfigurationBean - A init
10:33:52.766 [main] INFO org.table.config.staticwithconfig.StaticConfigurationBean - B construct B.a is 1801942731
10:33:52.766 [main] INFO org.table.config.staticwithconfig.StaticConfigurationBean - B init
```

​       

#### 情况三

将`static`去掉，并将注入的类型改为`BeanFactoryPostProcessor`，Spring给出了警告，Config类的生命周期执行不正常（`@PostConstruct`方法没执行），不过A与B之间的引用是正常的，但是customPostProcessor功能没执行，C类被注入进来了。

```console
10:39:46.057 [main] INFO org.table.config.staticwithconfig.StaticConfigurationBean - Config construct
10:39:46.062 [main] INFO org.springframework.context.annotation.ConfigurationClassEnhancer - @Bean method Config.customPostProcessor is non-static and returns an object assignable to Spring's BeanFactoryPostProcessor interface. This will result in a failure to process annotations such as @Autowired, @Resource and @PostConstruct within the method's declaring @Configuration class. Add the 'static' modifier to this method to avoid these container lifecycle issues; see @Bean javadoc for complete details.
10:39:46.069 [main] INFO org.table.config.staticwithconfig.StaticConfigurationBean - customPostProcessor construct
10:39:46.082 [main] DEBUG org.springframework.beans.factory.support.DefaultListableBeanFactory - Creating shared instance of singleton bean 'b'
10:39:46.083 [main] DEBUG org.springframework.beans.factory.support.DefaultListableBeanFactory - Creating shared instance of singleton bean 'a'
10:39:46.090 [main] INFO org.table.config.staticwithconfig.StaticConfigurationBean - A construct a is 127702987
10:39:46.090 [main] INFO org.table.config.staticwithconfig.StaticConfigurationBean - A init
10:39:46.090 [main] INFO org.table.config.staticwithconfig.StaticConfigurationBean - B construct B.a is 127702987
10:39:46.091 [main] INFO org.table.config.staticwithconfig.StaticConfigurationBean - B init
10:39:46.091 [main] DEBUG org.springframework.beans.factory.support.DefaultListableBeanFactory - Creating shared instance of singleton bean 'c'
10:39:46.091 [main] INFO org.table.config.staticwithconfig.StaticConfigurationBean - C construct
```

 **原因分析：**换成`BeanFactoryPostProcessor`后，`postProcessBeanDefinitionRegistry`方法不会执行，这是因为Spring是根据你声明的类型进行Bean筛选，并不是Bean的实际类型，具体可从`DefaultListableBeanFactory#getBeanNamesForType`跟踪，然后见`AbstracctBeanFactory#isTypeMatch`。

至于为什么Full mode 还是生效了，得看具体CGLIB增强的时机，看`ConfigurationClassPostProcessor#enhanceConfigurationClasses`方法，会发现它是在`ConfigurationClassPostProcessor#postProcessBeanFactory`方法中调用，而`ConfigurationClassPostProcessor`的优先级是比较高的，会在我们的customPostProcessor之前，所以能正常的增强到Config类。





#### 情况四

注入的类型为`BeanFactoryPostProcessor`并且加上`static`关键字，除了customPostProcessor失效外别的都正常

```console
11:00:47.823 [main] INFO org.table.config.staticwithconfig.StaticConfigurationBean - customPostProcessor construct
11:00:47.838 [main] INFO org.table.config.staticwithconfig.StaticConfigurationBean - Config construct
11:00:47.839 [main] INFO org.table.config.staticwithconfig.StaticConfigurationBean - Config init
11:00:47.839 [main] DEBUG org.springframework.beans.factory.support.DefaultListableBeanFactory - Creating shared instance of singleton bean 'b'
11:00:47.849 [main] DEBUG org.springframework.beans.factory.support.DefaultListableBeanFactory - Creating shared instance of singleton bean 'a'
11:00:47.858 [main] INFO org.table.config.staticwithconfig.StaticConfigurationBean - A construct a is 105579928
11:00:47.858 [main] INFO org.table.config.staticwithconfig.StaticConfigurationBean - A init
11:00:47.859 [main] INFO org.table.config.staticwithconfig.StaticConfigurationBean - B construct B.a is 105579928
11:00:47.859 [main] INFO org.table.config.staticwithconfig.StaticConfigurationBean - B init
11:00:47.859 [main] DEBUG org.springframework.beans.factory.support.DefaultListableBeanFactory - Creating shared instance of singleton bean 'c'
11:00:47.859 [main] INFO org.table.config.staticwithconfig.StaticConfigurationBean - C construct
```

 **原因分析：** 见情况三



### 总结一下

- `static` 关键字在`Spring` 实例化 `@Bean`注入的 `BeanFactoryPostProcessor` 时，避免外部配置类的过早实例化而导致的配置类生命周期异常，以及@Bean Full Mode失效问题（其实只有`BeanDefinitionRegistryPostProcessor`会），所以通过`@Bean`注入`BeanFactoryPostProessor`时要用`static`

- 在自定义`BeanDefinitionRegistryPostProcessor`时，注入时类型一定要是`BeanDefinitionRegistryPostProcessor`，不能是父类的`BeanFactoryPostProcessor`

  

## References

-<https://docs.spring.io/spring-framework/docs/current/reference/html/core.html#beans-java-basic-concepts>