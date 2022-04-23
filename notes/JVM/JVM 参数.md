---
layout:     note
title:      "JVM 参数"
subtitle:   " \"JVM 参数\""
author:     "tablesheep"
catalog: true
hide-in-nav: true
header-style: text
tags:
- note
---





## JVM参数笔记



## -X

| **-Xms** | **设置初始堆大小，仅仅只是虚拟内存，只有当程序运行需要更多的内存时才会绑定物理内存** |
| -------- | ------------------------------------------------------------ |
| **-Xmx** | **设置最大堆大小，为了防止内存申请释放的消耗，建议-Xms -Xmx 设置为一样的值** |
| **-Xmn** | **设置年轻代的初始和最大堆大小**                             |
| **-Xss** | **设置线程栈大小**                                           |



## -XX

| -XX:+HeapDumpOnOutOfMemoryError | 发生OOM时转储dump文件                                        |
| ------------------------------- | ------------------------------------------------------------ |
| **-XX:HeapDumpPath=/yourpath**  | **转储dump文件路径**                                         |
| **-XX:MetaspaceSize**           | **metaspace 初始大小**                                       |
| **-XX:MaxMetaspaceSize**        | **metaspace 最大大小**                                       |
| **-XX:NewRatio**                | **老年代:新生代比值，默认为2，即老年代占2，新生代占1**       |
| **-XX:SurvivorRatio**           | **新生代比值，含义为 Eden:S0:S1，默认为8，即Eden:S0:S1=8:1:1** |

