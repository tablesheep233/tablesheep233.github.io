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

**StringEncoder、StringDecoder**

字符串编解码器



**IdleStateHandler**

空闲状态处理，构造函数可以指定读、写、读写操作多久没发生，然后向后面的`ChannelHandler`发送`IdleStateEvent`，后面的`ChannelHandler`需要实现`ChannelInboundHandler#userEventTriggered`方法处理



## HTTP

**HttpRequestDecoder**

http request 解码器，用于服务端入站流量



**HttpResponseEncoder**

http response 编码器，用于服务端出站流量



**HttpServerCodec**

http服务端编解码器，通过`CombinedChannelDuplexHandler`组合`HttpRequestDecoder`、`HttpResponseEncoder`



**HttpRequestEncoder**

http requeset编码器，用于客户端出站流量



**HttpResponseDecoder**

http response解码器，用于客户端入站流量



**HttpClientCodec**

http 客户端编解码器，通过`CombinedChannelDuplexHandler`组合了`HttpRequestEncoder`、`HttpResponseDecoder`



**HttpObjectAggregator**

Http 消息聚合器，可以将分块的`HttpMessage`和`HttpContent`聚合到单个的`FullHttpRequest`或`FullHttpResponse`中，用于处理Response时将它插入`HttpResponseDecoder`之后，处理Request时将它插在`HttpRequestDecoder`和`HttpResponseEncoder`之后



**HttpContentCompressor**

Http 内容压缩，用于服务端，支持`gzip`、`deflate`、`br`压缩，同时支持`Accept-Encoding`请求头协商



**HttpContentDecompressor**

Http 内容解压，用于客户端



## Protobuf

> protobuf 传输时，无脑把这四个加上就完事了

**ProtobufVarint32FrameDecoder**

解析varint 32 长度信息

```
* BEFORE DECODE (302 bytes)       AFTER DECODE (300 bytes)
* +--------+---------------+      +---------------+
* | Length | Protobuf Data |----->| Protobuf Data |
* | 0xAC02 |  (300 bytes)  |      |  (300 bytes)  |
* +--------+---------------+      +---------------+
```

**ProtobufVarint32LengthFieldPrepender**

拼接varint 32 长度信息

```
* BEFORE ENCODE (300 bytes)       AFTER ENCODE (302 bytes)
* +---------------+               +--------+---------------+
* | Protobuf Data |-------------->| Length | Protobuf Data |
* |  (300 bytes)  |               | 0xAC02 |  (300 bytes)  |
* +---------------+               +--------+---------------+
```

**ProtobufDecoder**

**ProtobufEncoder**



## 异步处理

```java
//ctx: ChannelHandlerContext
ctx.channel().eventLoop().execute();
ctx.channel().eventLoop().scheduleWithFixedDelay();
ctx.channel().eventLoop().scheduleAtFixedRate();
```



## ChannelHandler

**@ChannelHandler.Sharable** 

用于标识ChannelHandler是否为共享实例，需要注意线程安全问题，使用🌰

```java
@ChannelHandler.Sharable
public class ShareableChannelHandler extends SimpleChannelInboundHandler<String> {
......    
}


public class DefaultChannelInitializer extends ChannelInitializer<SocketChannel> {
    
    private ShareableChannelHandler shareabledHandler = new ShareableChannelHandler();
    
    @Override
    protected void initChannel(SocketChannel sc) throws Exception {
        //注意注入时要以成员变量或者别的不变的方式，不能以新建对象的方式，否则共享不会失效
        sc.pipeline().addLast(shareabledHandler);
    }
}
```

*DefaultChannelPipeline*

对于`@ChannelHandler.Sharable`注解，Netty 仅仅只是在添加`ChannelHandler`时检查这个handler是否共享以及是否第一次添加，不满足其一则在报错

```java
private static void checkMultiplicity(ChannelHandler handler) {
    if (handler instanceof ChannelHandlerAdapter) {
        ChannelHandlerAdapter h = (ChannelHandlerAdapter) handler;
        //判断是否共享以及是否第一次添加
        if (!h.isSharable() && h.added) {
            throw new ChannelPipelineException(
                    h.getClass().getName() +
                    " is not a @Sharable handler, so can't be added or removed multiple times.");
        }
        //标记为已添加过
        h.added = true;
    }
}
```