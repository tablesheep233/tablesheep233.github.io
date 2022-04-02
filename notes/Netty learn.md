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


## Basic



### StringEncoder、StringDecoder

字符串编解码器



### IdleStateHandler

空闲状态处理，构造函数可以指定读、写、读写操作多久没发生，然后向后面的`ChannelHandler`发送`IdleStateEvent`，后面的`ChannelHandler`需要实现`ChannelInboundHandler#userEventTriggered`方法处理



## HTTP

### HttpRequestDecoder

http request 解码器，用于服务端入站流量



### HttpResponseEncoder

http response 编码器，用于服务端出站流量



### HttpServerCodec

http服务端编解码器，通过`CombinedChannelDuplexHandler`组合`HttpRequestDecoder`、`HttpResponseEncoder`



### HttpRequestEncoder

http requeset编码器，用于客户端出站流量



### HttpResponseDecoder

http response解码器，用于客户端入站流量



### HttpClientCodec

http 客户端编解码器，通过`CombinedChannelDuplexHandler`组合了`HttpRequestEncoder`、`HttpResponseDecoder`



### HttpObjectAggregator

Http 消息聚合器，可以将分块的`HttpMessage`和`HttpContent`聚合到单个的`FullHttpRequest`或`FullHttpResponse`中，用于处理Response时将它插入`HttpResponseDecoder`之后，处理Request时将它插在`HttpRequestDecoder`和`HttpResponseEncoder`之后



### HttpContentCompressor

Http 内容压缩，用于服务端，支持`gzip`、`deflate`、`br`压缩，同时支持`Accept-Encoding`请求头协商



### HttpContentDecompressor

Http 内容解压，用于客户端



## Protobuf

> protobuf 传输时，无脑把这四个加上就完事了

### ProtobufVarint32FrameDecoder

解析varint 32 长度信息

```
* BEFORE DECODE (302 bytes)       AFTER DECODE (300 bytes)
* +--------+---------------+      +---------------+
* | Length | Protobuf Data |----->| Protobuf Data |
* | 0xAC02 |  (300 bytes)  |      |  (300 bytes)  |
* +--------+---------------+      +---------------+
```

### ProtobufVarint32LengthFieldPrepender

拼接varint 32 长度信息

```
* BEFORE ENCODE (300 bytes)       AFTER ENCODE (302 bytes)
* +---------------+               +--------+---------------+
* | Protobuf Data |-------------->| Length | Protobuf Data |
* |  (300 bytes)  |               | 0xAC02 |  (300 bytes)  |
* +---------------+               +--------+---------------+
```

### ProtobufDecoder

### ProtobufEncoder