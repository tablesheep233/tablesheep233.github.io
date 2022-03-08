---
layout:     post
title:      "Spring Cloud loadbalancer 源码速读"
subtitle:   " \"Spring Cloud loadbalancer 源码速读\""
author:     "tablesheep"
date:       2022-01-25 21:43:00
header-style: text
catalog: true
tags:
- Spring Cloud loadbalancer
- Spring Cloud
---

## Spring Cloud loadbalancer

Spring Cloud 2020版本以后默认移除了对Netflix的依赖，使用Spring Cloud Commons提供抽象和公共类，用于不同的Spring Cloud实现。而对于负载均衡来说使用`Spring Cloud loadbalancer` 替代了`Ribbon`。



### 主要的类

![loadbalance](/img/post/LB/loadbalance.png)

`ReactiveLoadBalancer`、`ReactorLoadBalancer`：负载均衡接口

```java
public interface ReactiveLoadBalancer<T> {

   /**
    * Default implementation of a request.
    */
   Request<DefaultRequestContext> REQUEST = new DefaultRequest<>();

   /**
    * Choose the next server based on the load balancing algorithm.
    * @param request - incoming request
    * @return publisher for the response
    */
   @SuppressWarnings("rawtypes")
   Publisher<Response<T>> choose(Request request);

   default Publisher<Response<T>> choose() { // conflicting name
      return choose(REQUEST);
   }
}
```

`ReactorServiceInstanceLoadBalancer`：基于`ServiceInstance`的负载均衡接口

```java
public interface ReactorServiceInstanceLoadBalancer extends ReactorLoadBalancer<ServiceInstance> {
}
```

`ReactiveLoadBalancer.Factory`：`ReactiveLoadBalancer`工厂

`LoadBalancerClientFactory`：基于`ServiceInstance`的`ReactiveLoadBalancer`工厂，同时继承了`NamedContextFactory`

```java
public class LoadBalancerClientFactory extends NamedContextFactory<LoadBalancerClientSpecification>
      implements ReactiveLoadBalancer.Factory<ServiceInstance> {
    ......
}
```

`ServiceInstance`、`ServiceInstanceListSupplier`：服务实例&服务实例列表提供商


默认所有的`Loadbalancer`都实现了`ReactorServiceInstanceLoadBalancer`，所以它们实现的`choose`方法返回值最终都是`Response<ServiceInstance>`。实际上看过这几个`LoadBalancer`的实现会发现它们都是依赖于`ServiceInstanceListSupplier`提供的`List<ServiceInstancel>`，在使用各自的算法筛选`ServiceInstance`。



### Spring Cloud loadbalancer 配置

`LoadBalancerAutoConfiguration`：获取所有的负载均衡配置（`LoadBalancerClientSpecification`、`LoadBalancerClientsProperties`），注入`LoadBalancerClientFactory`

`LoadBalancerClientConfiguration`：负载均衡客户端默认配置，默认注入了`RoundRobinLoadBalancer`以及根据环境注入`ServiceInstanceListSupplier`

`@LoadBalancerClients`、`@LoadBalancerClient`、`LoadBalancerClientSpecification`、`LoadBalancerClientConfigurationRegistrar`：配置组件们

`BlockingLoadBalancerClientAutoConfiguration`：`RestTemplate`负载均衡配置

**配置主要流程**

注解`@LoadBalancerClients`、`@LoadBalancerClient`通过`@Import`导入`LoadBalancerClientConfigurationRegistrar`，而`registrar`根据注解的信息构建`LoadBalancerClientSpecification`类型的`BeanDefinition`，最后在自动装配时会获取所有的配置注入到`LoadBalancerClientFactory`中。

`LoadBalancerClientFactory`继承了`org.springframework.cloud.context.named.NamedContextFactory` ，配合`@LoadBalancerClients`或者`@LoadBalancerClient`，可以实现对于不同负载均衡实例的独立配置。（`Spring Cloud Open Feign`也是根据`NamedContentFactory`实现不同`Feign`的独立配置，大致上就是根据name构建子`AnnotationConfigApplicationContext`存储在`Map`中，获取`Bean`时从对应的`Context`获取从而实现配置的隔离）



### ServiceInstanceListSupplier

Spring Cloud 提供了许多的`ServiceInstanceListSupplier`

`DiscoveryClientServiceInstanceListSupplier`：服务发现

`CachingServiceInstanceListSupplier`：缓存（结合Discovery使用时要注意二次缓存问题，比如在跟nacos使用时，因为nacos有缓存，所以并不需要它）

`RetryAwareServiceInstanceListSupplier`：重试相关，重试时不会选取失败的服务

`HealthCheckServiceInstanceListSupplier`：健康检查相关，只返回可用的服务

`HintBasedServiceInstanceListSupplier`：hint翻译是提示，其实就是根据标签进行负载均衡，可以实现简单的灰度路由

`ZonePreferenceServiceInstanceListSupplier`：根据区域进行负载均衡

......

可以通过`ServiceInstanceListSupplierBuilder`组合多种的`ServiceInstanceListSupplier`



### loadbalancer自定义配置

根据文档，可以实现自己的`ServiceInstanceListSupplier`或`ReactorLoadBalancer`，配合`@LoadBalancerClients`、`@LoadBalancerClient`实现自定义的配置，要注意的是实现的自定义类不要配合`@Configuration`使用，不然会变成全局配置。

![custom_loadbalance](/img/post/LB/custom_loadbalance.jpg)

![configuration_loadbalance](/img/post/LB/configuration_loadbalance.jpg)

#### 全局灰度路由

- **使用HintBasedServiceInstanceListSupplier简单实现 **

  > env：Spring Cloud 2020.0.x + Nacos，RPC使用feign
  >
  > 服务有三个，调用链路是gateway -> provider-service -> provider1-service

  ```java
  /*******************************gateway、provider-service配置******************************/
  
  /**
   * 灰度配置类 
   **/
  public class GrayConfiguration {
  
      @Bean
      public ServiceInstanceListSupplier discoveryClientServiceInstanceListSupplier(
              ConfigurableApplicationContext context) {
          return ServiceInstanceListSupplier.builder()
                  .withDiscoveryClient()
                  .withHints()
                  .build(context);
      }
   }
  
  /*------------------------------------------------------------------------------*/
  
  /**
   * 在配置类上面配置loadBalancer
   **/
   @LoadBalancerClients(defaultConfiguration = GrayConfiguration.class)
   或 
   @LoadBalancerClient(name = "provider-service", configuration = GrayConfiguration.class)
  
  /*********************************网关配置文件********************************/
  spring:
    application:
      name: gateway
    cloud:
      loadbalancer:
        hint: //hint标志
          provider-service: gray
        hint-header-name: H-GRAY //http hint请求头
        
  /******************************provider-service配置***************************/
  /*---------------------------------配置文件-----------------------------------------*/
   spring:
    application:
      name: provider-service
    cloud:
      loadbalancer:
        hint: //hint标志
          provider1-service: gray
        hint-header-name: H-GRAY //http hint请求头
      nacos:
        discovery:
          metadata:
            hint: gray //服务元数据hint标记
  
  /*-----------------------------------servlet过滤器-------------------------------------------*/
  /*
  ** 将hint请求头放入threadlocal中，方便rpc服务获取
  */
  public static TransmittableThreadLocal<String> GRAY_VERSION = TransmittableThreadLocal.withInitial(() -> "");
  
  @Bean
  public OncePerRequestFilter grayFilter() {
      return new OncePerRequestFilter() {
          @Override
          protected void doFilterInternal(HttpServletRequest request, HttpServletResponse response, FilterChain filterChain) throws ServletException, IOException {
              String header = request.getHeader("H-GRAY");
              if (StringUtils.hasText(header)) {
                  GRAY_VERSION.set(header);
              }
              filterChain.doFilter(request, response);
          }
      };
  }
  
  /*-------------------------------------feign interceptor-----------------------------------------*/
  /*
  ** 将hint请求头放入RequestTemplate
  */
  @Bean
  public RequestInterceptor grayRequestInterceptor() {
      return new RequestInterceptor() {
          @Override
          public void apply(RequestTemplate template) {
              if (StringUtils.hasText(GRAY_VERSION.get())) {
                  template.header("H-GRAY", GRAY_VERSION.get());
              }
          }
      };
  }
                
  /******************************provider1-service配置文件***************************/       
   spring:
    application:
      name: provider1-service
    cloud:
      nacos:
        discovery:
          metadata:
            hint: gray //服务元数据hint标记
  ```



- **自定义`ServiceInstanceListSupplier`或`ReactorLoadBalancer`实现**

  略🤔🤔🤔


## References

- <https://docs.spring.io/spring-cloud-commons/docs/current/reference/html/>
- <https://github.com/spring-cloud/spring-cloud-commons>