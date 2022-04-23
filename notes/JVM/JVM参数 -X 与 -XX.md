---
layout:     note
title:      "JVM 参数 -X 与 -XX"
subtitle:   " \"JVM 参数 -X 与 -XX\""
author:     "tablesheep"
catalog: true
hide-in-nav: true
header-style: text
tags:
- note
---

# JVM 参数 -X 与 -XX

**JVM 参数 -X -XX 参数，两者都不是稳定的，-X 要区分版本**

>- Options that begin with `-X` are non-standard (not guaranteed to be supported on all VM implementations), and are subject to change without notice in subsequent releases of the JDK.
>- Options that are specified with `-XX` are not stable and are subject to change without notice.



- -X 可以通过 java -X 查看当前版本支持的所有参数
- -XX 可以通过 java -XX:+PrintFlagsFinal 打印所有系统参数的值
  - -XX:+PrintVMOptions 程序运行时，打印虚拟机接收的命令行显式参数。
  - -XX:+PrintCommandLineFlags 打印传递给虚拟机的显式和隐式参数。

   

-XX 有两种类型，boolean 类型和非boolean类型(number or string)

- boolean 类型，-XX:[±]参数
- 非boolean类型，-XX:参数=？

> - Boolean options are turned on with `-XX:+<option>` and turned off with `-XX:-<option>`.Disa
> - Numeric options are set with `-XX:<option>=<number>`. Numbers can include 'm' or 'M' for megabytes, 'k' or 'K' for kilobytes, and 'g' or 'G' for gigabytes (for example, 32k is the same as 32768).
> - String options are set with `-XX:<option>=<string>`, are usually used to specify a file, a path, or a list of commands



- <https://www.oracle.com/java/technologies/javase/vmoptions-jsp.html>