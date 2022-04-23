---
layout:     note
title:      "ssh 连接服务器问题"
subtitle:   " \"ssh 连接服务器问题\""
author:     "tablesheep"
catalog: true
hide-in-nav: true
header-style: text
tags:
- note
- Linux
- ssh
---

## 本地虚拟机

### 防火墙问题

- 将对应ip加入白名单，或者关闭防火墙



### 服务器不支持root远程登陆

修改ssh 配置，然后重启

```
#/etc/ssh/sshd_config 修改为如下
.....
# Authentication:

LoginGraceTime 2m
PermitRootLogin yes
StrictModes yes
......
```





## 阿里云服务器

- <https://help.aliyun.com/document_detail/41487.htm>
