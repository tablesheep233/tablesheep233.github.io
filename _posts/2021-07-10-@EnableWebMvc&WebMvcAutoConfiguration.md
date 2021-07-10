---
layout:     post
title:      "@EnableWebMvc与WebMvcAutoConfiguration"
subtitle:   " \"Spring Boot WebMVC 配置的一些注意点\""
author:     "tablesheep"
date:       2021-07-10 20:10:00
header-style: text
catalog: true
tags:
    - Java
    - Spring
    - Spring Boot
    - Spring Web MVC
    - 源码
---



> 记一次WebMVC配置问题以及源码阅读



> version：
>
> spring-boot 2.5.x



## 背景

后端`Long`类型在前端会出现精度丢失问题，可以通过`Jackson`配置序列化为`String`（其实已经过时了）

```yaml
spring:
  jackson:
    generator:
      write_numbers_as_strings: true
```

但在某次代码更新后发现失效了。



## 解决

通过debug跟踪，发现通过`ObjectMapper`生成的`UTF8JsonGenerator`并不符合预期

```
public class UTF8JsonGenerator extends JsonGeneratorImpl {
    .......
    @Override
    public void writeNumber(long l) throws IOException
    {
        _verifyValueWrite(WRITE_NUMBER);
        if (_cfgNumbersAsStrings) { //_cfgNumbersAsStrings为false，导致序列化出现异常
            _writeQuotedLong(l);
            return;
        }
        if ((_outputTail + 21) >= _outputEnd) {
            // up to 20 digits, minus sign
            _flushBuffer();
        }
        _outputTail = NumberOutput.outputLong(l, _outputBuffer, _outputTail);
    }
   ......
}
```

跟踪后发现`MappingJackson2HttpMessageConverter`并不是通过`Jackson`自动装配注入而来，而是通过`WebMvcConfigurationSupport#addDefaultHttpMessageConverters`而来，并没有读取使用配置。最后查看代码提交和控制变量法，发现是由于为了自定义枚举的转换，但却添加了`@EnableWebMvc`导致的。

去除`@EnableWebMvc`其实就已经可以了，但是发现上述配置已经过时了，于是采用了`WebMvcConfigurer`重新配置。

```java
//@EnableWebMvc
@Configuration
public class CustomizeWebConfig implements WebMvcConfigurer {

    @Override
    public void extendMessageConverters(List<HttpMessageConverter<?>> converters) {
        // 修改默认MappingJackson2HttpMessageConverter 配置jackson 序列化 long to string
        MappingJackson2HttpMessageConverter jackson2HttpMessageConverter = null;
        for (HttpMessageConverter<?> converter : converters) {
            if (converter instanceof MappingJackson2HttpMessageConverter) {
                jackson2HttpMessageConverter = (MappingJackson2HttpMessageConverter) converter;
                break;
            }
        }
        if (Objects.isNull(jackson2HttpMessageConverter)) {
            jackson2HttpMessageConverter = new MappingJackson2HttpMessageConverter();
        }

        ObjectMapper objectMapper = jackson2HttpMessageConverter.getObjectMapper();
        SimpleModule module = new SimpleModule();
        module.addSerializer(Long.class, ToStringSerializer.instance);
        objectMapper.registerModule(module);

        converters.add(jackson2HttpMessageConverter);
    }
    
    ...
}
```



## 源码分析

代码较多，截取部分重要代码，同时以`HttpMessageConverter`配置为例。

### @EnableWebMvc

可以看到`EnableWebMvc`通过`@Import`注入了`DelegatingWebMvcConfiguration`。

```java
@Retention(RetentionPolicy.RUNTIME)
@Target(ElementType.TYPE)
@Documented
@Import(DelegatingWebMvcConfiguration.class)
public @interface EnableWebMvc {
}
```

所以接下来看`DelegatingWebMvcConfiguration`的父类`WebMvcConfigurationSupport` :D

### WebMvcConfigurationSupport

可以看到`WebMvcConfigurationSupport`中定义了很多通过`@Bean`注入的Spring Web MVC的Bean，同时暴露了一些方法去扩展Bean的行为（典型的模板方法）。可以通过`@EnableWebMvc`来注入，或者继承该类使用`@Configuration`来注入，同时可以通过覆盖该类的`@Bean`方法（记得添加`@Bean`），或者重写扩展方法来自定义MVC的配置。

```java
/**
 * This is the main class providing the configuration behind the MVC Java config.
 * It is typically imported by adding {@link EnableWebMvc @EnableWebMvc} to an
 * application {@link Configuration @Configuration} class. An alternative more
 * advanced option is to extend directly from this class and override methods as
 * necessary, remembering to add {@link Configuration @Configuration} to the
 * subclass and {@link Bean @Bean} to overridden {@link Bean @Bean} methods.
 * For more details see the javadoc of {@link EnableWebMvc @EnableWebMvc}.
 * ......
 */
public class WebMvcConfigurationSupport implements ApplicationContextAware, ServletContextAware {
    ......

	@Bean
	public RouterFunctionMapping routerFunctionMapping(
			@Qualifier("mvcConversionService") FormattingConversionService conversionService,
			@Qualifier("mvcResourceUrlProvider") ResourceUrlProvider resourceUrlProvider) {

		RouterFunctionMapping mapping = new RouterFunctionMapping();
		mapping.setOrder(3);
		mapping.setInterceptors(getInterceptors(conversionService, resourceUrlProvider));
		mapping.setCorsConfigurations(getCorsConfigurations());
        // 调用了getMessageConverters()方法获取MessageConverters
		mapping.setMessageConverters(getMessageConverters()); 

		PathPatternParser patternParser = getPathMatchConfigurer().getPatternParser();
		if (patternParser != null) {
			mapping.setPatternParser(patternParser);
		}

		return mapping;
	}
    
    /**
     * 首先调用configureMessageConverters配置HttpMessageConverter
     * 若没有配置则获取默认的HttpMessageConverter
     * 最后通过extendMessageConverters扩展HttpMessageConverter
     */
	protected final List<HttpMessageConverter<?>> getMessageConverters() {
		if (this.messageConverters == null) {
			this.messageConverters = new ArrayList<>();
			configureMessageConverters(this.messageConverters);
			if (this.messageConverters.isEmpty()) {
				addDefaultHttpMessageConverters(this.messageConverters);
			}
			extendMessageConverters(this.messageConverters);
		}
		return this.messageConverters;
	}
	
	protected void configureMessageConverters(List<HttpMessageConverter<?>> converters) {
	}

	protected void extendMessageConverters(List<HttpMessageConverter<?>> converters) {
	}
   
    ......
}
```



### DelegatingWebMvcConfiguration

`@EnableWebMvc`实际导入的类，通过继承`WebMvcConfigurationSupport`导入一堆Bean，同时自定义了部分MVC的配置。这里出现了两个新的类`WebMvcConfigurer`和`WebMvcConfigurerComposite`，而对于MVC的配置都是通过`WebMvcConfigurer`和`WebMvcConfigurerComposite`完成的，可以看到注入了所有的`WebMvcConfigurer`，同时委托了`WebMvcConfigurerComposite`代为执行。

```java
/**
 * A subclass of {@code WebMvcConfigurationSupport} that detects and delegates
 * to all beans of type {@link WebMvcConfigurer} allowing them to customize the
 * configuration provided by {@code WebMvcConfigurationSupport}. This is the
 * class actually imported by {@link EnableWebMvc @EnableWebMvc}.
 *
 * @author Rossen Stoyanchev
 * @since 3.1
 */
@Configuration(proxyBeanMethods = false)
public class DelegatingWebMvcConfiguration extends WebMvcConfigurationSupport {

   private final WebMvcConfigurerComposite configurers = new WebMvcConfigurerComposite();


   @Autowired(required = false)
   public void setConfigurers(List<WebMvcConfigurer> configurers) {
      if (!CollectionUtils.isEmpty(configurers)) {
         this.configurers.addWebMvcConfigurers(configurers);
      }
   }
   
   ......

   @Override
   protected void configureMessageConverters(List<HttpMessageConverter<?>> converters) {
      this.configurers.configureMessageConverters(converters);
   }

   @Override
   protected void extendMessageConverters(List<HttpMessageConverter<?>> converters) {
      this.configurers.extendMessageConverters(converters);
   }
   ......
}
```



### WebMvcConfigurer

定义了很多用于配置MVC的方法，结合`@EnableWebMvc`可以注入所有的`WebMvcConfigurer`，同时MVC组件会调用所有的配置方法进行自定义的配置。

```
/**
 * Defines callback methods to customize the Java-based configuration for
 * Spring MVC enabled via {@code @EnableWebMvc}.
 *
 * <p>{@code @EnableWebMvc}-annotated configuration classes may implement
 * this interface to be called back and given a chance to customize the
 * default configuration.
 *
 * @author Rossen Stoyanchev
 * @author Keith Donald
 * @author David Syer
 * @since 3.1
 */
public interface WebMvcConfigurer {
   ......

   /**
    * Configure the {@link HttpMessageConverter HttpMessageConverter}s for
    * reading from the request body and for writing to the response body.
    * <p>By default, all built-in converters are configured as long as the
    * corresponding 3rd party libraries such Jackson JSON, JAXB2, and others
    * are present on the classpath.
    * <p><strong>Note</strong> use of this method turns off default converter
    * registration. Alternatively, use
    * {@link #extendMessageConverters(java.util.List)} to modify that default
    * list of converters.
    * @param converters initially an empty list of converters
    */
   default void configureMessageConverters(List<HttpMessageConverter<?>> converters) {
   }

   /**
    * Extend or modify the list of converters after it has been, either
    * {@link #configureMessageConverters(List) configured} or initialized with
    * a default list.
    * <p>Note that the order of converter registration is important. Especially
    * in cases where clients accept {@link org.springframework.http.MediaType#ALL}
    * the converters configured earlier will be preferred.
    * @param converters the list of configured converters to be extended
    * @since 4.1.3
    */
   default void extendMessageConverters(List<HttpMessageConverter<?>> converters) {
   }

   ......
}
```



### WebMvcConfigurerComposite

代理（组合）了所有的`WebMvcConfigurer`，通过循环执行所有的自定义配置。

```java
/**
 * A {@link WebMvcConfigurer} that delegates to one or more others.
 *
 * @author Rossen Stoyanchev
 * @since 3.1
 */
class WebMvcConfigurerComposite implements WebMvcConfigurer {

   private final List<WebMvcConfigurer> delegates = new ArrayList<>();


   public void addWebMvcConfigurers(List<WebMvcConfigurer> configurers) {
      if (!CollectionUtils.isEmpty(configurers)) {
         this.delegates.addAll(configurers);
      }
   }

   ......

   @Override
   public void configureMessageConverters(List<HttpMessageConverter<?>> converters) {
      for (WebMvcConfigurer delegate : this.delegates) {
         delegate.configureMessageConverters(converters);
      }
   }

   @Override
   public void extendMessageConverters(List<HttpMessageConverter<?>> converters) {
      for (WebMvcConfigurer delegate : this.delegates) {
         delegate.extendMessageConverters(converters);
      }
   }
   ......
}
```



### WebMvcAutoConfiguration

再来看看Spring Boot 的自动装配是如何实现MVC配置的。

从`@ConditionalOnMissingBean(WebMvcConfigurationSupport.class)`可以知道`@EnableWebMvc`为什么会导致自动装配失效。

再看`WebMvcAutoConfiguration#EnableWebMvcConfiguration`，继承了`DelegatingWebMvcConfiguration`，引入了很多的配置，使我们能通过文件配置Spring MVC。（注入了`EnableWebMvcConfiguration`就相当于`@EnableWebMvc`注入了`DelegatingWebMvcConfiguration`）

最后再看`WebMvcAutoConfiguration#WebMvcAutoConfigurationAdapter`，实现了`WebMvcConfigurer`，通过它Spring Boot完成了对MVC组件的配置。

```java
@Configuration(proxyBeanMethods = false)
@ConditionalOnWebApplication(type = Type.SERVLET)
@ConditionalOnClass({ Servlet.class, DispatcherServlet.class, WebMvcConfigurer.class })
//可以看到自动装配的条件之一就是Bean中没有WebMvcConfigurationSupport，所以当我们使用了@EnableWebMvc之后自动装配便会失效
@ConditionalOnMissingBean(WebMvcConfigurationSupport.class) 
@AutoConfigureOrder(Ordered.HIGHEST_PRECEDENCE + 10)
@AutoConfigureAfter({ DispatcherServletAutoConfiguration.class, TaskExecutionAutoConfiguration.class,
      ValidationAutoConfiguration.class })
public class WebMvcAutoConfiguration {
   
   ......

   @SuppressWarnings("deprecation")
   @Configuration(proxyBeanMethods = false)
   @Import(EnableWebMvcConfiguration.class)
   @EnableConfigurationProperties({ WebMvcProperties.class, WebProperties.class })
   @Order(0)
   public static class WebMvcAutoConfigurationAdapter implements WebMvcConfigurer, ServletContextAware {
      ......

      private final ObjectProvider<HttpMessageConverters> messageConvertersProvider;

      public WebMvcAutoConfigurationAdapter(WebProperties webProperties, WebMvcProperties mvcProperties,
            ListableBeanFactory beanFactory, ObjectProvider<HttpMessageConverters> messageConvertersProvider,
            ObjectProvider<ResourceHandlerRegistrationCustomizer> resourceHandlerRegistrationCustomizerProvider,
            ObjectProvider<DispatcherServletPath> dispatcherServletPath,
            ObjectProvider<ServletRegistrationBean<?>> servletRegistrations) {
         ......
         this.messageConvertersProvider = messageConvertersProvider;
         ......
      }

     ......

      @Override
      public void configureMessageConverters(List<HttpMessageConverter<?>> converters) {
         /**
         * 通过ObjectProvider获取了HttpMessageConverters，
         * 而HttpMessageConverters包含了所有自动装配的HttpMessageConverter（例如MappingJackson2HttpMessageConverter），
         * 具体可以看HttpMessageConvertersAutoConfiguration
         */
         this.messageConvertersProvider
               .ifAvailable((customConverters) -> converters.addAll(customConverters.getConverters()));
      }
     ......
   }

   /**
    * Configuration equivalent to {@code @EnableWebMvc}.
    */
   @Configuration(proxyBeanMethods = false)
   @EnableConfigurationProperties(WebProperties.class)
   public static class EnableWebMvcConfiguration extends DelegatingWebMvcConfiguration implements ResourceLoaderAware {
       ......
           
       public EnableWebMvcConfiguration(WebMvcProperties mvcProperties, WebProperties webProperties,
				ObjectProvider<WebMvcRegistrations> mvcRegistrationsProvider,
				ObjectProvider<ResourceHandlerRegistrationCustomizer> resourceHandlerRegistrationCustomizerProvider,
				ListableBeanFactory beanFactory) {
			this.resourceProperties = webProperties.getResources();
			this.mvcProperties = mvcProperties;
			this.webProperties = webProperties;
			this.mvcRegistrations = mvcRegistrationsProvider.getIfUnique();
			this.beanFactory = beanFactory;
		}
       ......
   }

}
```



References
----------

- <https://docs.spring.io/spring-framework/docs/current/reference/html/web.html#spring-web>

- <https://github.com/spring-projects/spring-framework/tree/main/spring-webmvc/src/main/java/org/springframework/web/servlet/config/annotation>

- <https://github.com/spring-projects/spring-boot/blob/main/spring-boot-project/spring-boot-autoconfigure/src/main/java/org/springframework/boot/autoconfigure/web/servlet/WebMvcAutoConfiguration.java>
