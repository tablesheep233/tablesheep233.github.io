---
layout:     post
title:      "CORS & Spring CORS"
subtitle:   " \"Spring CORS相关记录\""
date:       2021-06-18 19:10:00
author:     "tablesheep"
catalog: true
tags:
    - Java
    - CORS
    - Spring Web
---

>  “CORS & Spring CORS相关记录”



## 基础概念

（仅仅只是小总结，具体可以看References）

同源政策规定浏览器请求只能发送给同源的网址。（`同源`指的是协议、域名、端口相同）

`CORS`是跨源资源分享（Cross-Origin Resource Sharing）的缩写，是为了跨域请求能够在同源政策下正常发送的一种机制。`CORS`需要浏览器（主流浏览器都支持）与服务器同时支持。

`CORS`将请求分为`简单请求`与`非简单请求`，对于两种请求采取的策略不同。

#### 简单请求

简单请求定义

1. `GET`、`POST`、`HEAD`
2. 允许人为设置的的请求头只包含[对 CORS 安全的首部字段集合](https://fetch.spec.whatwg.org/#cors-safelisted-request-header)（ `Accept`、`Accept-Language`、`Content-Language`、`Content-Type`（`application/x-www-form-urlencoded`、`multipart/form-data`、`text/plain`））

对于简单请求只需要在请求头添加一个`Origin`表示此次请求的源，服务器会根据`Origin`、请求方法等判断是否允许跨域请求。

#### 非简单请求

对于非简单请求，每次正常请求前会发送一个预检请求（`PreFlight Request`请求），此次请求为`OPTIONS`，会包含几个请求头给服务器判断是否允许。

```
Origin: https://www.baidu.com     //表示请求来源
Access-Control-Request-Method: GET  //请求所使用方法
Access-Control-Request-Headers: Header-1, Header-2 //请求将携带的Headers
```

如果预检请求不通过，服务器会返回一个没有`CORS`相关Headers的响应，此时浏览器就会报跨域错误。

如果请求通过则返回信息会有`CORS`相关Headers

```
Access-Control-Allow-Origin: https://www.baidu.com  //服务器允许的源，*表示所有源都可以跨域
Access-Control-Allow-Methods: POST, GET, OPTIONS //支持的所有跨域请求的方法
Access-Control-Allow-Headers: Header-1, Header-2  //支持的所有跨域请求头
Access-Control-Max-Age: 86400  //不一定有，表明该响应的有效时间（秒）。在有效时间内，浏览器无须为同一请求再次发起预检请求
Access-Control-Expose-Headers: Header-3, Header-4 //不一定有，默认情况下只有几种Header会暴露，可以通过它配置需要暴露的Header
```



#### Credentials请求

另外对于携带`Credentials`（凭证，比如Cookie）的请求也有不同的处理，当请求配置携带`Credentials`时，服务器必须返回如下响应头。

```
Access-Control-Allow-Credentials: true  //另外，当允许附带Credentials时，Access-Control-Allow-Origin不能用*
```



#### `Vary`与`CORS`

`Vary`是一个响应头部，而它的值是请求头，主要跟请求的缓存有关系。

当缓存服务器收到一个请求，只有当前的请求和原始（缓存）的请求头跟缓存的响应头里的Vary都匹配，才能使用缓存的响应。----[带Vary头的响应](https://developer.mozilla.org/zh-CN/docs/Web/HTTP/Caching#%E5%B8%A6Vary%E5%A4%B4%E7%9A%84%E5%93%8D%E5%BA%94)

```
Vary: Origin
Vary: Access-Control-Request-Method
Vary: Access-Control-Request-Headers
如果服务端指定了具体的域名而非“*”，那么响应首部中的 Vary 字段的值必须包含 Origin。这将告诉客户端：服务器对不同的源站返回不同的内容。（不过Spring目前是都会返回）
```



## Spring CORS 配置

#### `CorsConfiguration `

`CORS`配置类，几乎所有的跨域配置都与之有关

```

public class CorsConfiguration {
   ......
   @Nullable
   private List<String> allowedOrigins;  //后端允许的源，对应Access-Control-Allow-Origin

   @Nullable
   private List<OriginPattern> allowedOriginPatterns; //也可使用Pattern动态匹配源

   @Nullable
   private List<String> allowedMethods;  //后端允许的请求方法

   @Nullable
   private List<HttpMethod> resolvedMethods = DEFAULT_METHODS; //后端允许的请求方法

   @Nullable
   private List<String> allowedHeaders;  //后端允许的请求头

   @Nullable
   private List<String> exposedHeaders; //对应Access-Control-Expose-Headers

   @Nullable
   private Boolean allowCredentials; //是否允许携带凭证，对应Access-Control-Allow-Credentials

   @Nullable
   private Long maxAge; //预检请求有效时间，对应Access-Control-Max-Age

   ......
}
```



#### `@CrossOrigin`

通过`@CrossOrigin`可以最细粒度的配置方法或者Controller。

`@CrossOrigin`大致工作原理：

1.Handler注册时，解析`@CrossOrigin`配置并且缓存起来

- [AbstractHandlerMethodMapping](https://github.com/spring-projects/spring-framework/blob/main/spring-webmvc/src/main/java/org/springframework/web/servlet/handler/AbstractHandlerMethodMapping.java)#initHandlerMethods
- ......（省略部分调用链）
- [MappingRegistry](https://github.com/spring-projects/spring-framework/blob/main/spring-webmvc/src/main/java/org/springframework/web/servlet/handler/AbstractHandlerMethodMapping.java)#register

```
......

private final Map<HandlerMethod, CorsConfiguration> corsLookup = new ConcurrentHashMap<>();

......

public void register(T mapping, Object handler, Method method) {
   ......
//根据initCorsConfiguration模板方法获取CORS配置
      CorsConfiguration corsConfig = initCorsConfiguration(handler, method, mapping);
      if (corsConfig != null) {
         corsConfig.validateAllowCredentials();
         //校验通过后将CORS配置缓存在Map中
         this.corsLookup.put(handlerMethod, corsConfig); 
      }

      this.registry.put(mapping,
            new MappingRegistration<>(mapping, handlerMethod, directPaths, name, corsConfig != null));
    ......
}
```

[RequestMappingHandlerMapping](https://github.com/spring-projects/spring-framework/blob/main/spring-webmvc/src/main/java/org/springframework/web/servlet/mvc/method/annotation/RequestMappingHandlerMapping.java)#initCorsConfiguration

```
//获取@CrossOrigin，最终生成CorsConfiguration
@Override
protected CorsConfiguration initCorsConfiguration(Object handler, Method method, RequestMappingInfo mappingInfo) {
   HandlerMethod handlerMethod = createHandlerMethod(handler, method);
   Class<?> beanType = handlerMethod.getBeanType();
   CrossOrigin typeAnnotation = AnnotatedElementUtils.findMergedAnnotation(beanType, CrossOrigin.class);
   CrossOrigin methodAnnotation = AnnotatedElementUtils.findMergedAnnotation(method, CrossOrigin.class);

   if (typeAnnotation == null && methodAnnotation == null) {
      return null;
   }

   CorsConfiguration config = new CorsConfiguration();
   updateCorsConfig(config, typeAnnotation);
   updateCorsConfig(config, methodAnnotation);

   if (CollectionUtils.isEmpty(config.getAllowedMethods())) {
      for (RequestMethod allowedMethod : mappingInfo.getMethodsCondition().getMethods()) {
         config.addAllowedMethod(allowedMethod.name());
      }
   }
   return config.applyPermitDefaultValues();
}
```



2.在获取Hander时会获取`CORS`配置，然后根据配置在`HandlerExecutionChain`添加`CorsInterceptor`进行跨域校验

获取`CORS`配置调用链

- [DispatcherServlet](https://github.com/spring-projects/spring-framework/blob/main/spring-webmvc/src/main/java/org/springframework/web/servlet/DispatcherServlet.java)#doDispatch
- [DispatcherServlet](https://github.com/spring-projects/spring-framework/blob/main/spring-webmvc/src/main/java/org/springframework/web/servlet/DispatcherServlet.java)#getHandler
- [AbstractHandlerMapping](https://github.com/spring-projects/spring-framework/blob/main/spring-webmvc/src/main/java/org/springframework/web/servlet/handler/AbstractHandlerMapping.java)#getHandler
- [AbstractHandlerMethodMapping](https://github.com/spring-projects/spring-framework/blob/main/spring-webmvc/src/main/java/org/springframework/web/servlet/handler/AbstractHandlerMethodMapping.java)#getCorsConfiguration
- [MappingRegistry](https://github.com/spring-projects/spring-framework/blob/main/spring-webmvc/src/main/java/org/springframework/web/servlet/handler/AbstractHandlerMethodMapping.java)#getCorsConfiguration

根据`CorsConfiguration`创建`CorsInterceptor`逻辑 [AbstractHandlerMapping](https://github.com/spring-projects/spring-framework/blob/main/spring-webmvc/src/main/java/org/springframework/web/servlet/handler/AbstractHandlerMapping.java)#getHandler

```
public final HandlerExecutionChain getHandler(HttpServletRequest request) throws Exception {
......
//判断是否有CORS配置或者是否预检请求
   if (hasCorsConfigurationSource(handler) || CorsUtils.isPreFlightRequest(request)) {
   //获取CORS配置
      CorsConfiguration config = getCorsConfiguration(handler, request);
      if (getCorsConfigurationSource() != null) {
         CorsConfiguration globalConfig = getCorsConfigurationSource().getCorsConfiguration(request);
         config = (globalConfig != null ? globalConfig.combine(config) : config);
      }
      if (config != null) {
         config.validateAllowCredentials();
      }
      //添加拦截器
      executionChain = getCorsHandlerExecutionChain(request, executionChain, config);
   }

   return executionChain;
}
```

	protected HandlerExecutionChain getCorsHandlerExecutionChain(HttpServletRequest request,
			HandlerExecutionChain chain, @Nullable CorsConfiguration config) {
	
		if (CorsUtils.isPreFlightRequest(request)) {
			HandlerInterceptor[] interceptors = chain.getInterceptors();
			return new HandlerExecutionChain(new PreFlightHandler(config), interceptors);
		}
		else {
		//最终会根据CorsConfiguration创建CorsInterceptor
			chain.addInterceptor(0, new CorsInterceptor(config));
			return chain;
		}
	}


#### `WebMvcConfigurer#addCorsMappings`

可以通过实现`WebMvcConfigurer`进行`CORS`全局配置

```
@Configuration
public class CustomizeWebConfig implements WebMvcConfigurer {
    @Override
    public void addCorsMappings(CorsRegistry registry) {
        registry.addMapping("/**")
                .allowedOrigins(CorsConfiguration.ALL)
                .allowedMethods(CorsConfiguration.ALL)
                .allowedHeaders(CorsConfiguration.ALL);
    }
}
```

具体原理是在`WebMvcConfigurationSupport`注册`MVC`组件时（几个`AbstractHandlerMapping`），会根据`WebMvcConfigurationSupport`#getCorsConfigurations获取到我们的配置并且合并成`CorsConfigurationSource`，最终也是在[AbstractHandlerMapping](https://github.com/spring-projects/spring-framework/blob/main/spring-webmvc/src/main/java/org/springframework/web/servlet/handler/AbstractHandlerMapping.java)#getHandler通过`CorsConfigurationSource`获取`CorsConfiguration`并且添加`CorsInterceptor`进行跨域校验

具体[AbstractHandlerMapping](https://github.com/spring-projects/spring-framework/blob/main/spring-webmvc/src/main/java/org/springframework/web/servlet/handler/AbstractHandlerMapping.java)#getHandler

```
public final HandlerExecutionChain getHandler(HttpServletRequest request) throws Exception {
......
   if (hasCorsConfigurationSource(handler) || CorsUtils.isPreFlightRequest(request)) {
      CorsConfiguration config = getCorsConfiguration(handler, request);
      if (getCorsConfigurationSource() != null) {
      //获取CorsConfigurationSource，根据CorsConfigurationSource获取全局CorsConfiguration
         CorsConfiguration globalConfig = getCorsConfigurationSource().getCorsConfiguration(request);
         //将全局配置与细粒度配置结合
         config = (globalConfig != null ? globalConfig.combine(config) : config);
      }
      if (config != null) {
         config.validateAllowCredentials();
      }
      executionChain = getCorsHandlerExecutionChain(request, executionChain, config);
   }

   return executionChain;
}
```



#### `CorsFilter`

可以通过`CorsFilter`通过过滤器进行配置（WebFlux是`CorsWebFilter`）

```
@Bean
public CorsFilter corsFilter() {
    UrlBasedCorsConfigurationSource source=  new UrlBasedCorsConfigurationSource();
    CorsConfiguration corsConfiguration = new CorsConfiguration();
    corsConfiguration.addAllowedOrigin(CorsConfiguration.ALL);
    corsConfiguration.addAllowedMethod(CorsConfiguration.ALL);
    corsConfiguration.addAllowedHeader(CorsConfiguration.ALL);
    source.registerCorsConfiguration("/**", corsConfiguration);
    return new CorsFilter(source);
}
```



#### 小结

`@CrossOrigin`和`WebMvcConfigurer`最终是通过`CorsInterceptor`实现，而`CorsFilter`是过滤器，区别就是`Filter` 与 `Interceptor` 的区别，`Filter`在 `DispatcherServlet` 调用前，`Interceptor`在`DispatcherServlet` 之后，`Hander`（Controller）之前。



## 问题记录

### Gateway重复配置

问题：在微服务改造过程中可能会存在网关与应用服务都存在`CORS`配置情况，此时浏览器会因为重复的`CORS header`而出现不能正确处理的情况。（The 'Access-Control-Allow-Origin' header contains multiple values 'xxx, xxx', but only one is allowed）

解决：`Spring Cloud Gateway` 提供了[DedupeResponseHeaderGatewayFilterFactory](https://github.com/spring-cloud/spring-cloud-gateway/blob/main/spring-cloud-gateway-server/src/main/java/org/springframework/cloud/gateway/filter/factory/DedupeResponseHeaderGatewayFilterFactory.java)以解决Response Header重复问题。

```java
/*
Use case: Both your legacy backend and your API gateway add CORS header values. So, your consumer ends up with
          Access-Control-Allow-Credentials: true, true
          Access-Control-Allow-Origin: https://musk.mars, https://musk.mars
(The one from the gateway will be the first of the two.) To fix, add
          DedupeResponseHeader=Access-Control-Allow-Credentials Access-Control-Allow-Origin
Configuration parameters:
- name
    String representing response header names, space separated. Required.
- strategy
	RETAIN_FIRST - Default. Retain the first value only.
	RETAIN_LAST - Retain the last value only.
	RETAIN_UNIQUE - Retain all unique values in the order of their first encounter.
Example 1
      default-filters:
      - DedupeResponseHeader=Access-Control-Allow-Credentials
Response header Access-Control-Allow-Credentials: true, false
Modified response header Access-Control-Allow-Credentials: true
Example 2
      default-filters:
      - DedupeResponseHeader=Access-Control-Allow-Credentials, RETAIN_LAST
Response header Access-Control-Allow-Credentials: true, false
Modified response header Access-Control-Allow-Credentials: false
Example 3
      default-filters:
      - DedupeResponseHeader=Access-Control-Allow-Credentials, RETAIN_UNIQUE
Response header Access-Control-Allow-Credentials: true, true
Modified response header Access-Control-Allow-Credentials: true
 */

/**
 * @author Vitaliy Pavlyuk
 */
public class DedupeResponseHeaderGatewayFilterFactory
		extends AbstractGatewayFilterFactory<DedupeResponseHeaderGatewayFilterFactory.Config> {

	private static final String STRATEGY_KEY = "strategy";

	public DedupeResponseHeaderGatewayFilterFactory() {
		super(Config.class);
	}

	@Override
	public List<String> shortcutFieldOrder() {
		return Arrays.asList(NAME_KEY, STRATEGY_KEY);
	}

	@Override
	public GatewayFilter apply(Config config) {
		return new GatewayFilter() {
			@Override
			public Mono<Void> filter(ServerWebExchange exchange, GatewayFilterChain chain) {
				return chain.filter(exchange)
						.then(Mono.fromRunnable(() -> dedupe(exchange.getResponse().getHeaders(), config)));
			}

			@Override
			public String toString() {
				return filterToStringCreator(DedupeResponseHeaderGatewayFilterFactory.this)
						.append(config.getName(), config.getStrategy()).toString();
			}
		};
	}

	public enum Strategy {

		/**
		 * Default: Retain the first value only.
		 */
		RETAIN_FIRST,

		/**
		 * Retain the last value only.
		 */
		RETAIN_LAST,

		/**
		 * Retain all unique values in the order of their first encounter.
		 */
		RETAIN_UNIQUE

	}

	void dedupe(HttpHeaders headers, Config config) {
		String names = config.getName();
		Strategy strategy = config.getStrategy();
		if (headers == null || names == null || strategy == null) {
			return;
		}
		for (String name : names.split(" ")) {
			dedupe(headers, name.trim(), strategy);
		}
	}

	private void dedupe(HttpHeaders headers, String name, Strategy strategy) {
		List<String> values = headers.get(name);
		if (values == null || values.size() <= 1) {
			return;
		}
		switch (strategy) {
		case RETAIN_FIRST:
			headers.set(name, values.get(0));
			break;
		case RETAIN_LAST:
			headers.set(name, values.get(values.size() - 1));
			break;
		case RETAIN_UNIQUE:
			headers.put(name, new ArrayList<>(new LinkedHashSet<>(values)));
			break;
		default:
			break;
		}
	}

	public static class Config extends AbstractGatewayFilterFactory.NameConfig {

		private Strategy strategy = Strategy.RETAIN_FIRST;

		public Strategy getStrategy() {
			return strategy;
		}

		public Config setStrategy(Strategy strategy) {
			this.strategy = strategy;
			return this;
		}
	}
}
```

如下：

```yaml
spring:
  cloud:
    gateway:
	  default-filters:
	    - DedupeResponseHeader=Access-Control-Allow-Origin Access-Control-Allow-Credentials, RETAIN_FIRST
```



References
----------

- <https://www.w3.org/TR/cors/>
- <https://developer.mozilla.org/zh-CN/docs/Web/HTTP/CORS>
- <https://www.ruanyifeng.com/blog/2016/04/same-origin-policy.html>
- <https://www.ruanyifeng.com/blog/2016/04/CORS.html>
- <https://developer.mozilla.org/zh-CN/docs/Web/HTTP/Headers/Access-Control-Allow-Origin>
- <https://developer.mozilla.org/zh-CN/docs/Web/HTTP/Headers/Access-Control-Expose-Headers>
- <https://developer.mozilla.org/zh-CN/docs/Web/HTTP/Headers/Access-Control-Allow-Credentials>
- <https://developer.mozilla.org/zh-CN/docs/Web/HTTP/Headers/Vary>
- <https://developer.mozilla.org/zh-CN/docs/Web/HTTP/Caching>