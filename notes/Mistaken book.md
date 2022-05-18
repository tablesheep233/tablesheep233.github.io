---
layout:     note
title:      "Mistaken book"
subtitle:   " \"Mistaken book\""
author:     "tablesheep"
catalog: true
hide-in-nav: true
header-style: text
tags:
- note
- Mistaken book
---

# Mistaken book



> 记录一些问题以及解决方法



## Java Exception



| Name                                                         | description                                                  | repetition | solution                                                     | remark                                                       |
| ------------------------------------------------------------ | ------------------------------------------------------------ | ---------- | ------------------------------------------------------------ | ------------------------------------------------------------ |
| **NPE**                                                      | 1.`Java 9+ `集合 `of` 方法创建的集合类使用时要注意，`ImmutableCollections`中到处都是`Objects#requireNonNull`（说你呢`indexOf`方法）<br> |            |                                                              | 1.不过作为不可变集合，不能为null的检测其实也很合理，为了该死的性能 |
| **org.springframework.orm.<br>ObjectOptimisticLockingFailureException** | `Spring data jpa` 使用`@Version` 修改记录时出现版本不一致导致 |            | 看业务，如果是定时任务一类的可能是重复的任务执行，可以选择忽略，如果是用户端操作，可以让用户重试或者采取加锁的方式 |                                                              |
| **com.mysql.cj.exceptions.UnableToConnectException: Public Key Retrieval is not allowed** | 客户端由于没有公钥无法进行通讯                               |            | 去掉jdbc连接中useSSL=false解决（我遇到的是由于配置了后无法使用SSL/TLS协议导致） |                                                              |
|                                                              |                                                              |            |                                                              |                                                              |
|                                                              |                                                              |            |                                                              |                                                              |





## problem with Java

| description                                                  | description&solution                                         | remark                                 | tag          |
| ------------------------------------------------------------ | ------------------------------------------------------------ | -------------------------------------- | ------------ |
|                                                              |                                                              |                                        |              |
| 上传文件存放到服务器，由nginx进行路由获取无权限问题          | 1. `java.io.File#setReadable(true, false) `设置文件所有人可读<br>2. 保证用户或用户组一致<br>3. ...... |                                        |              |
| [ERROR] Failed to execute goal org.springframework.boot:spring-boot-maven-plugin:3.0.0:repackage (default) on project nacos-console: Execution default of goal org.springframework.boot:spring-boot-maven-plugin:3.0.0:repackage failed:<br/> Unable to load the mojo 'repackage' in the plugin 'org.springframework.boot:spring-boot-maven-plugin:3.0.0' due to an API incompatibility: org.codehaus.plexus.component.repository.exception.ComponentLookupException: org/springframe<br/>work/boot/maven/RepackageMojo has been compiled by a more recent version of the Java Runtime (class file version 61.0), this version of the Java Runtime only recognizes class file versions up to 55.0 | 由于spring-boot-maven-plugin版本过高导致，指定低版本解决     | nacos Build distribution packages 报错 | nacos、maven |
| nacos  cluster 启动三个节点一直 <b>Nacos is starting...</b>  | 查issue有两种情况<br>1.指定ip不能为127.0.0.1<br>2.网络原因导致raft协商失败，关闭VPN，重置了网络都不行<br>最后查看`protocol-raft.log`，发现有两个节点协商的ip是之前缓存的（其实在排查网络原因的时候就有看过日志但是看的是没问题的节点，所以才导致排查了很久。。），最后重置有问题的节点的数据解决了 | nacos raft协商问题                     | nacos        |
| 文档等导出，服务器字体问题<br/>Caused by: java.lang.NullPointerException: null at java.desktop/sun.awt.FontConfiguration.getVersion(FontConfiguration.java:1264) | 上传字体文件 <br/>cd /usr/share/fonts/ (一般用simsun.ttf就好)<br/>mkfontscale<br/>mkfontdir<br/>fc-cache<br/>fc-list 查看字体<br/>重启服务 | yum install -y fontconfig mkfontscale  |              |
| hibernate自增主键报错<br/>relation "hibernate_sequence" does not exist | PostgreSQL: 需要创建序列CREATE SEQUENCE hibernate_sequence START 1 INCREMENT 1; |                                        |              |
| 由于服务器时间与客户发送时间不一致导致的业务逻辑问题         | ntp校准&同步服务器时间                                       |                                        |              |
|                                                              |                                                              |                                        |              |
