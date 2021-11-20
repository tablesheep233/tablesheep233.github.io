---
layout:     post
title:      "Spring Cloud Gateway workflow"
subtitle:   " \"Spring Cloud Gateway 工作流程\""
author:     "tablesheep"
date:       2021-11-20 22:28:00
header-style: text
catalog: true
tags:
- Java
- Spring
- Spring Cloud Gateway
- 源码
---


## Spring Gateway 重要概念

**Route**：路由，包含了路由uri，Predicate，Filter等属性，在执行时会匹配Predicate判断是否能通过，并结合Filter将请求路由到后续的服务中去

**Predicate**：通过ServerWebExchange对象匹配请求是否符合路由

**Filter**：这个Filter是Gateway中的Filter，而不是webFlux中的，但是作用是差不多的



## Spring Gateway 工作流程

在网关启用服务发现配置下，以请求`GET http://127.0.0.1:8080/user-service/user/1`为例子说明。

```yaml
spring:
  cloud:
    gateway:
      discovery:
        locator:
        #该配置主要是导入了DiscoveryClientRouteDefinitionLocator，其中主要是PathRoutePredicateFactory创建的Predicate和RewritePathGatewayFilterFactory创建的RewritePathGatewayFilter（具体的规则看GatewayDiscoveryClientAutoConfiguration）
          enabled: true 
          lower-case-service-id: true
```



`Spring Gateway` 基于`Spring webFlux`，请求进来时会进入`DispatcherHandler`最后会进入`Gateway`定义的`HandlerMapping`，`RoutePredicateHandlerMapping`中

> PS： `Gateway`的一些信息的传递几乎都是放在`ServerWebExchange`的`attributes`中

### RoutePredicateHandlerMapping

`RoutePredicateHandlerMapping`继承了`AbstractHandlerMapping`，在`getHandlerInternal`方法中，会调用`lookupRoute`方法获取路由，然后根据路由中的`Predicate`去判断请求是否符合规则

```java
protected Mono<Route> lookupRoute(ServerWebExchange exchange) {
    //使用RouteLocator获取路由，在其中会将路由的Predicate、Filter设置进去
    //主要看RouteDefinitionRouteLocator类 convertToRoute方法
		return this.routeLocator.getRoutes()
				// individually filter routes so that filterWhen error delaying is not a
				// problem
				.concatMap(route -> Mono.just(route).filterWhen(r -> {
					// add the current route we are testing
					exchange.getAttributes().put(GATEWAY_PREDICATE_ROUTE_ATTR, r.getId());
                    //获取路由中的谓词判断请求是否符合该路由规则
					return r.getPredicate().apply(exchange);
				})
				.doOnError(e -> logger.error("Error applying predicate for route: " + route.getId(), e))
						.onErrorResume(e -> Mono.empty()))
				.next()
				// TODO: error handling
				.map(route -> {
					if (logger.isDebugEnabled()) {
						logger.debug("Route matched: " + route.getId());
					}
					validateRoute(route, exchange);
					return route;
				});
	}
```

`getHandlerInternal`获取到`Rotue`后会将`Rotue`设置到`ServerWebExchange`中，然后返回`FilteringWebHandler`供后续执行`Gateway Filter`。返回的`Rotue`是根据`DiscoveryClientRouteDefinitionLocator`创建的`RouteDefinition`来的，`Route`会包含后续服务的`uri`，在这里是`lb://user-service`，服务发现的一些元数据`metadata`等等

```java
@Override
protected Mono<?> getHandlerInternal(ServerWebExchange exchange) {
//......
   return lookupRoute(exchange)
         .flatMap((Function<Route, Mono<?>>) r -> {
            exchange.getAttributes().remove(GATEWAY_PREDICATE_ROUTE_ATTR);
            if (logger.isDebugEnabled()) {
               logger.debug("Mapping [" + getExchangeDesc(exchange) + "] to " + r);
            }
             //将路由设置到ServerWebExchange中
            exchange.getAttributes().put(GATEWAY_ROUTE_ATTR, r);
            return Mono.just(webHandler);
         }).switchIfEmpty(Mono.empty().then(Mono.fromRunnable(() -> {
            exchange.getAttributes().remove(GATEWAY_PREDICATE_ROUTE_ATTR);
            if (logger.isTraceEnabled()) {
               logger.trace("No RouteDefinition found for [" + getExchangeDesc(exchange) + "]");
            }
         })));
}
```



### FilteringWebHandler

`FilteringWebHandler`是一个`WebHandler`，作为`RoutePredicateHandlerMapping`匹配的结果，会在`DispatcherHandler`中调用`handle`方法

```java
@Override
public Mono<Void> handle(ServerWebExchange exchange) {
    //获取Route中的filter
   Route route = exchange.getRequiredAttribute(GATEWAY_ROUTE_ATTR);
   List<GatewayFilter> gatewayFilters = route.getFilters();

   //结合全局filter
   List<GatewayFilter> combined = new ArrayList<>(this.globalFilters);
   combined.addAll(gatewayFilters);
   // TODO: needed or cached?
   AnnotationAwareOrderComparator.sort(combined);

   if (logger.isDebugEnabled()) {
      logger.debug("Sorted gatewayFilterFactories: " + combined);
   }

    //使用DefaultGatewayFilterChain执行Gateway Filter
   return new DefaultGatewayFilterChain(combined).filter(exchange);
}
```

`GatewayFilter`是真正执行`Gateway`请求的地方

```java
//根据规则重写请求，根据GatewayDiscoveryClientAutoConfiguration配置，这里就是将请求的服务名截出来，http://127.0.0.1:8080/user-service/user/1 -> http://127.0.0.1:8080/user/1
RewritePathGatewayFilterFactory 

//根据Route转换请求URI，并将URI放在GATEWAY_REQUEST_URL_ATTR中，/user/1 -> lb://user-service/user/1
RouteToRequestUrlFilter 
    
//负载均衡 将 lb 开头的请求负载到真实的服务去，会将负载后的URI写入GATEWAY_REQUEST_URL_ATTR，原URI放入GATEWAY_ORIGINAL_REQUEST_URL_ATTR，lb://user-service/user/1 -> http://ip:port/user/1
ReactiveLoadBalancerClientFilter
    
//请求后续服务，会将header、body等数据发送到后续服务中
NettyRoutingFilter
    
//在别的Gateway后运行，将代理请求的响应写入到网关的response中
NettyWriteResponseFilter
```



## References

- <https://cloud.spring.io/spring-cloud-gateway/reference/html/>