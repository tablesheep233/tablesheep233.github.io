```
---
layout:     post
title:      "Spring Boot Tomcat源码阅读"
subtitle:   " \"Spring Boot Tomcat源码阅读\""
author:     "tablesheep"
date:       2022-05-11 20:46:00
header-style: text
catalog: true
tags:
- Spring Boot
---
```



## Spring Boot Tomcat源码阅读

整合思路：将Tomcat的配置变为Spring Bean（包含Spring Boot配置和编程配置），然后根据这些Bean配置Tomcat。



### Tomcat创建流程

> 入口：`ServletWebServerApplicationContext#createWebServer`





描述：Spring Boot在 Servlet环境启动时，通过`ServletWebServerApplication#createWebServer`创建Tomcat内嵌容器

- 创建`ServletWebServerFactory`（`TomcatServletWebServerFactory`）

  - `WebServerFactoryCustomizerBeanPostProcessor`在初始化前拦截`ServletWebServerFactory`，通过一系列（有序）`WebServerFactoryCustomizer`配置`ServletWebServerFactory`。

    包括但不限于

    - `ServletWebServerFactoryCustomizer`设置通用容器配置，见`ServerProperties`。
    - `TomcatServletWebServerFactoryCustomizer`、`TomcatWebServerFactoryCustomizer`设置Tomcat Servlet & 容器配置，见`ServerProperties.Tomcat`。

    主要是将配置设置到`TomcatServletWebServerFactory`，或者通过`Tomcat Customizer `设置

- 通过`ServletWebServerFactory#getWebServer`创建并配置`Tomcat`，具体看源码

  ```java
  //TomcatServletWebServerFactory#getWebServer
  @Override
  public WebServer getWebServer(ServletContextInitializer... initializers) {
     if (this.disableMBeanRegistry) {
        Registry.disableRegistry();
     }
      //创建Tomcat&一系列配置
     Tomcat tomcat = new Tomcat();
     File baseDir = (this.baseDirectory != null) ? this.baseDirectory : createTempDir("tomcat");
     tomcat.setBaseDir(baseDir.getAbsolutePath());
     for (LifecycleListener listener : this.serverLifecycleListeners) {
        tomcat.getServer().addLifecycleListener(listener);
     }
     Connector connector = new Connector(this.protocol);
     connector.setThrowOnFailure(true);
     tomcat.getService().addConnector(connector);
      //配置Connector,执行TomcatProtocolHandlerCustomizer & TomcatConnectorCustomizer
     customizeConnector(connector);
     tomcat.setConnector(connector);
     tomcat.getHost().setAutoDeploy(false);
     configureEngine(tomcat.getEngine());
     for (Connector additionalConnector : this.additionalTomcatConnectors) {
        tomcat.getService().addConnector(additionalConnector);
     }
      //配置tomcat context 执行TomcatContextCustomizer
     prepareContext(tomcat.getHost(), initializers);
     return getTomcatWebServer(tomcat);
  }
  ```

  



### Tomcat 配置

- Web容器配置类：`org.springframework.boot.autoconfigure.web.ServerProperties` （配置文件控制）

- Tomcat 配置类：`org.springframework.boot.autoconfigure.web.ServerProperties.Tomcat`（配置文件控制）

- Tomcat Customizer类（编程式控制）：

  - Tomcat Connector配置类： `org.springframework.boot.web.embedded.tomcat.TomcatConnectorCustomizer`
  - Tomcat Context配置类：`org.springframework.boot.web.embedded.tomcat.TomcatContextCustomizer`
  - Tomcat ProtocolHandler 配置类：`org.springframework.boot.web.embedded.tomcat.TomcatProtocolHandlerCustomizer`

  > 编程式控制Spring Boot Tomcat：Spring Boot提供了以上三个Tomcat Customizer配置，我们可以通过BeanPostProcessor等方式拦截ServletWebServerFactory，添加自定义的Tomcat Customizer