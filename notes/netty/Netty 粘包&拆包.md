---
layout:     note
title:      "netty learn"
subtitle:   " \"netty learn\""
author:     "tablesheep"
catalog: true
hide-in-nav: true
header-style: text
tags:
- note
- netty
---



## TCP粘包原因

> tcp socket连接在内核中都有一个发送(send)缓冲区和接收(recv)缓冲区。发送时数据包先发送到发送缓冲区中，由Nagle算法决定是不是立即发送。接收时，先到接收缓冲区中，再从内核拷贝到用户空间。

- 发送端：由于Nagle算法可能导致多个包积累在发送缓冲区到一定量或超时才一起发生

- 接收端：应用程序未能及时处理导致包堆积在接收缓冲区导致

  

  根本原因，数据包之间没有办法区分界限，导致应用处理时发生错误。



## Netty粘包解决方案

### FixedLengthFrameDecoder

指定消息的长度，不足的需要填充，比较不灵活。



### LineBasedFrameDecoder

使用 "\n" 或者 "\r\n" 划分一条消息，构造方法可以指定消息最大长度，当达到最大长度还没解析出消息则会报错。



### DelimiterBasedFrameDecoder

指定分隔符作为消息分隔符



### LengthFieldPrepender

长度编码器

```java
public LengthFieldPrepender(
        ByteOrder byteOrder,  //指定消息长度按大端模式还是小端模式
        int lengthFieldLength, //长度信息字段长度
        int lengthAdjustment, //长度信息调整
        boolean lengthIncludesLengthFieldLength //长度占用的字节是否算在消息长度中，true 则长度为 原始消息长度 + 长度占用的字节长度
) {
......
}
```



### LengthFieldBasedFrameDecoder

长度解码器

```java
public LengthFieldBasedFrameDecoder(
        ByteOrder byteOrder, //指定消息长度按大端模式还是小端模式
        int maxFrameLength,  //最大的消息长度
        int lengthFieldOffset, //长度信息字段开始的偏移
        int lengthFieldLength, //长度信息字段长度
        int lengthAdjustment, //长度信息字段后，调整几位后为真正的消息
        int initialBytesToStrip, //解码后去除几个字节，可以用于把长度信息等多余信息去除
        boolean failFast //超过最大长度后是否快速失败
) {
......
}
```



关于`lengthAdjustment`，比较难以理解，举两个🌰

🌰

- lengthFieldOffset ：0 ， 长度偏移量从0开始
- lengthFieldLength ：2 ，长度信息占2字节
- lengthAdjustment ：-2，长度信息补偿调整为-2
- initialBytesToStrip ： 2，解码去除2字节

长度信息记录为0x000E(14)，真实内容长度为0x000C(12) （引号不算），整个消息长度为14，lengthAdjustment 调整 -2，解码去掉2长度的字节，可得真正的内容。

```
  +--------+----------------+      +----------------+
  + 0x000E | "HELLO, WORLD" |   -> | "HELLO, WORLD" |  
  +--------+----------------+      +----------------+   
```



🌰🌰

- lengthFieldOffset ：0 ， 长度偏移量从0开始
- lengthFieldLength ：2 ，长度信息占2字节
- lengthAdjustment ：2，长度信息补偿调整为-2
- initialBytesToStrip ： 4，解码去除2字节

长度信息记录为0x000C(12)，真实内容长度为0x000C(12) （引号不算），整个消息长度为18，lengthAdjustment 调整 2，解码去掉4长度的字节，可得真正的内容。

```
+--------+----------+----------------+      +----------------+
| 0x000C |  0x0000  | "HELLO, WORLD" |  ->  | "HELLO, WORLD" |
+--------+----------+----------------+      +----------------+
```

 