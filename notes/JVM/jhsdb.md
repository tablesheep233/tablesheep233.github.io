---
layout:     note
title:      "jhsdb"
subtitle:   " \"jhsdb\""
author:     "tablesheep"
catalog: true
hide-in-nav: true
header-style: text
tags:
- note
---

jhsdb 是Java 9引入的命令行工具，它有以下几种模式，可以看到一些是我们很熟悉的，看得出来，官方在往工具整合的方向搞。

```
PS D:\> jhsdb.exe
    clhsdb              command line debugger
    hsdb                ui debugger
    debugd --help       to get more information
    jstack --help       to get more information
    jmap   --help       to get more information
    jinfo  --help       to get more information
    jsnap  --help       to get more information
```



其实俺也是在用 `jmap -heap PID` 发现报错了，然后提示说用这个才发现的（谁会去关心新版本的命令行工具啊。。。）

```
PS D:\> jmap -heap 10848
Error: -heap option used
Cannot connect to core dump or remote debug server. Use jhsdb jmap instead
```

变成

```
PS D:\> jhsdb.exe jmap --heap --pid 10848
```



整体用下来差不多，最多用`--help`查看手册咯~



- <https://docs.oracle.com/javase/9/tools/jhsdb.htm>