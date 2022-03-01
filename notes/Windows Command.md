---
layout:     note
title:      "Windows command"
subtitle:   " \"Windows command\""
author:     "tablesheep"
catalog: true
hide-in-nav: true
header-style: text
tags:
- note
- Windows
---

>  “Windows命令备忘”



## cmd编码

```cmd
chcp 查看
chcp 65001 切换为utf8编码
```



## 进程

```cmd
taskkill /f /t /im PID 杀进程&子进程
```



## 网络

```cmd
netstat -ano | findstr '8080' 端口情况
```



## 文本查找

```cmd
findstr /s /n "insert" .\* 在当前目录下递归查找所有insert文本
find /N /I "insert" result.txt 指定文件查找所有insert文本
```

