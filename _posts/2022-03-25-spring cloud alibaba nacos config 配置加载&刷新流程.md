---
layout:     post
title:      "spring cloud alibaba nacos config 配置加载&刷新流程"
subtitle:   " \"spring cloud alibaba nacos config 配置加载&刷新流程\""
author:     "tablesheep"
date:       2022-03-25 20:00:00
header-style: text
catalog: true
tags:
- Java
- Spring
- Spring Cloud Alibaba
- nacos
- 源码
---

## spring cloud alibaba nacos config 配置加载

其实就是利用了 spring cloud bootstrap配置加载的机制实现的。

- 通过SPI机制（spring.factories）配置`NacosConfigBootstrapConfiguration`，注入`NacosConfigManager`、`NacosPropertySourceLocator`
- `NacosPropertySourceLocator` 使用 nacos api（`NacosConfigService`、`ClientWorker`）加载配置，使用`NacosPropertySourceBuilder`把配置封装成`NacosPropertySource`

![spring-cloud-alibaba-nacos-config-load](/img/post/spring%20cloud%20alibaba%20nacos/spring%20cloud%20alibaba%20nacos%20config%20load.jpg)



## spring cloud alibaba nacos config 配置刷新

具体配置刷新是用spring cloud refreshScope 实现的，spring-cloud-starter-alibaba-nacos-config 主要负责监听nacos server 配置发布，然后拉取最新配置，发布`RefreshEvent`。

- 具体的监听在 `ClientWorker` 中

  - nacos 1.x 通过 `LongPollingRunnable` 内部类使用 http 长轮询的方式实现
  - nacos 2.x 通过 `ConfigRpcTransportClient` 内部类使用 grcp 长连接实现

  nacos 服务端会返回发生变化的配置给 `ClientWorker`，然后由`ClientWorker` 主动去拉取最新的配置

-  拉取到配置后会使用 `NacosContextRefresher` 中匿名内部的 `AbstractSharedListener` 去发布 `RefreshEvent`，接着会进入到 spring cloud refresh scope 的处理过程去刷新Bean 获取最新的配置。


![spring-cloud-alibaba-nacos-config-refresh](/img/post/spring%20cloud%20alibaba%20nacos/spring%20cloud%20alibaba%20nacos%20config%20refresh.jpg)




### References

- <https://github.com/alibaba/spring-cloud-alibaba>
- <https://github.com/alibaba/Nacos>