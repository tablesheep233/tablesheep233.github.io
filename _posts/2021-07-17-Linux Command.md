---
layout:     post
title:      "Linux Command"
subtitle:   " \"Linux命令备忘\""
date:       2021-07-17 21:10:00
header-style: text
author:     "tablesheep"
catalog: true
tags:
    - Java
    - Command
    - Linux
---

>  “Linux命令备忘”



## 工具安装

- iostat 

```sh
yum install -y sysstat
```



- netcat

```sh
##http://netcat.sourceforge.net/
yum install -y netcat
```



## 系统信息

```sh
getconf LONG_BIT 位数

uname -a  系统版本
cat /proc/version

```



## 磁盘

```sh
df 查看文件系统整体使用情况 -T 文件类型
du 扫描所有文件使用情况
du -h --max-depth=1 查看当前目录下个文件夹大小

lsblk 查看块设备情况（树状）
blkid 查看块设备基本信息（文件类型，UUID）

fdisk /dev/sdb   磁盘分区
fdisk -l 查看分区情况

mkfs 查看支持的格式
mkfs -t ext4 /dev/sdb 格式化磁盘分区

mount /dev/sdb /home/app  挂载
umount /dev/sdb 卸载
/etc/fstab 配置自动挂载，要注意UUID
```



### LVM概念

LVM：逻辑卷管理，Linux使用它管理磁盘，可以实现动态扩容。

PV：物理卷，粗略理解为磁盘分区。

VG：卷组，由一个或多个PV组成，可以在其上创建逻辑卷LV。（汇总多个物理磁盘资源）

LV：逻辑卷，从VG上的磁盘分区，可以动态调整大小。

### LVM管理

```sh
物理卷
pvscan 扫描物理卷并输出信息
pvcreate /dev/sdb 创建物理卷（将磁盘分区转换为物理卷）
pvdisplay 查看物理卷详细情况
pvremove /dev/sdb 删除物理卷
pvs 查看物理卷信息


卷组
vgscan 
vgcreate sdb 创建卷组
vgdisplay
vgremove
vgextend sdb /dev/sdb 扩展卷组
vgs 查看卷组

逻辑卷
lvscan
lvcreate [-L 指定大小 单位一堆记住G完事] [-l 按卷组剩余大小的百分比指定大小] [-n 名字] [逻辑卷]
lvcreate -L 100G -n share_sdb sdb
lvdisplay
lvremove
lvextend 扩容 lvextend -L +10G [lv名 通过lvdisplay查看]
lvresize 调整大小
lvreduce 缩容
lvs 查看逻辑卷

调整逻辑卷前需要先卸载，调整后要使用以下命令调整文件系统的大小
xfs_growfs [lv名]  xfs格式
resize2fs [lv名] ext格式
```



## 文件

```sh
mkdir -p 递归创建目录
find . -type f -size +1G | xargs du -h | sort -nr  查找大于1G的文件使用管道进行du以及排序操作

```

### 句柄

```sh
vim /etc/security/limits.conf 修改句柄
sysctl -a | grep file-max  最大文件句柄数
sysctl -a | grep file-nr 已经打开的文件句柄数
```



### 压缩解压

```sh
tar -zcvf pack.tar.gz pack/
tar -zxvf pack.tar.gz
unzip pack.zip
```



### 发送文件

```sh
scp -r -P 22 /home/app/config root@ip:/home/app/config 指定端口22发送
scp -r root@ip:/home/app/config /home/app/ 从远程拉取
```



## 文本

### vim

```sh
%s/源字符串/目的字符串/g 全局替换
```



### 查找

[grep](https://www.runoob.com/linux/linux-comm-grep.html)

```sh
grep -r -h -A 2 "insert" /g/linux-find/ 查找目录下包含insert的行，输出该行以及之后的两行，忽略文件名
```



 [awk](https://www.runoob.com/linux/linux-comm-awk.html)  文本处理

```sh
df -lh | awk '{print $1, $4}'  打印第一、四列
df -lh | awk 'substr($2, 0, length($2)-1) + 0 > 100' 打印内存大于100G的行
```





## 网络

配置目录 /etc/sysconfig/network-scripts

### 路由

```
traceroute www.baidu.com 查看到某个网站的路由
route 查看路由表，通过add,del改变路由

```



### 端口

```sh
lsof -i:port 查看端口情况
netstat -tunlp | grep 8080 查看8080端口情况
netstat -apu UDP使用情况
```



### netcat

```sh
nc -l -p port 开启tcp监听
nc ip port 发送tcp
nc -u -l -p port  开启udp监听
nc -u ip port 发送udp
```



### 抓包

tcpdump

```sh
tcpdump -i eth0 src localhost tcp port 8005 dst 0.0.0.0
tcpdump -i any port 80 -A
tcpdump udp port 10022 
```



### firewalld

**除了firewalld，还要注意iptable，曾经因为firewalld已禁用，但不知道iptable为何还在，导致端口一直不通，搞了2天**

```shell
---- centos7 开始使用防火墙 ----
systemctl start firewalld
firewall-cmd --permanent --add-port=8056/tcp  开发端口
firewall-cmd --add-forward-port=port=8006:proto=tcp:toport=1935:toaddr=xxx.xx.xx.xxx --permanent
firewall-cmd --permanent  --remove-forward-port=port=8006:proto=tcp:toaddr=xxx.xx.xx.xxx:toport=1935 --删除转发规则 
firewall-cmd --add-masquerade --permanent
firewall-cmd --reload

firewall-cmd --zone=public --list-ports -列出所有类型public 的端口
firewall-cmd --zone=public --list-ports -列出所有类型public 的端口


firewall-cmd --list-ports --列出所有信息
 
firewall-cmd --remove-port=8056/tcp --临时删除端口
firewall-cmd --zone=public --remove-port=8005/tcp --permanent --永久删除卡对外开放的端口
firewall-cmd --permanent --zone=public --remove-forward-port=port=8056:proto=tcp:toaddr=xxx.xx.xx.xxx:toport=3356 --删除转发规则 
```





## 定时任务

编辑 /etc/crontab （建议系统级配置）

```sh
crontab -e  （建议用户脚本）
```



## shell

### os.sh

```shell
DISK=$(df -lh)

CPU=$(mpstat | tail -1 | awk '{print $NF}' | awk -F% '{print 100-$1}')
USED=$(free -m | sed -n '2p' | awk '{print $3"M"}')
TOTAL=$(free -m | grep "Mem: " |awk '{print $2"M"}')
USEDRAT=$(free -m | sed -n '2p' | awk '{print $3/$2*100"%"}')

echo "磁盘使用情况:"
echo "$DISK"
echo " "
echo "内存总共大小: $TOTAL"
echo "内存使用情况：$USED"
echo "内存使用比率: $USEDRAT"
echo "CPU使用比率： $CPU"%""
```

