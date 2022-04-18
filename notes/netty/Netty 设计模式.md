---
layout:     note
title:      "netty 设计模式"
subtitle:   " \"netty 设计模式\""
author:     "tablesheep"
catalog: true
hide-in-nav: true
header-style: text
tags:
- note
- netty
---



### 工厂模式 - ChannelFactory

在使用 `Netty` 时都会设置 `Bootstrap` 的`Channel` 类型

```java
//Server
new ServerBootstrap().channel(NioServerSocketChannel.class);

//Client
new Bootstrap().channel(NioSocketChannel.class);
```

设置`Channel`的类型其实是创建了一个对应泛型的`ReflectiveChannelFactory`

```java
//AbstractBootstrap.java
public B channel(Class<? extends C> channelClass) {
    return channelFactory(new ReflectiveChannelFactory<C>(
            ObjectUtil.checkNotNull(channelClass, "channelClass")
    ));
}
```

而`ReflectiveChannelFactory`就是一个用反射创建对应`Channel`类型的工厂

```java
/**
 * A {@link ChannelFactory} that instantiates a new {@link Channel} by invoking its default constructor reflectively.
 */
public class ReflectiveChannelFactory<T extends Channel> implements ChannelFactory<T> {

    private final Constructor<? extends T> constructor;

    public ReflectiveChannelFactory(Class<? extends T> clazz) {
        ObjectUtil.checkNotNull(clazz, "clazz");
        try {
            this.constructor = clazz.getConstructor();
        } catch (NoSuchMethodException e) {
            throw new IllegalArgumentException("Class " + StringUtil.simpleClassName(clazz) +
                    " does not have a public non-arg constructor", e);
        }
    }

    @Override
    public T newChannel() {
        try {
            return constructor.newInstance();
        } catch (Throwable t) {
            throw new ChannelException("Unable to create Channel from class " + constructor.getDeclaringClass(), t);
        }
    }
......
}
```



### 模板模式

#### AbstractBootstrap#init(Channel channel)

在客户端调用`connect()`方法或者服务端调用`bind()`方法时，会创建`Channel`并进行初始化操作。

```java
//AbstractBootstrap#initAndRegister()
final ChannelFuture initAndRegister() {
    Channel channel = null;
    try {
        channel = channelFactory.newChannel();
        #init方法是个抽象方法，具体的实现在Bootstrap或ServerBootstrap中
        init(channel);
    } ......
}
```



#### MultithreadEventExecutorGroup#newChild(Executor executor, Object... args)

Netty 有不同的`EventLoopGroup`（`NioEventLoopGroup`、`EpollEventLoopGroup`、`KQueueEventLoopGroup`），这些`EventLoopGroup`都继承了`MultithreadEventExecutorGroup`，而在`MultithreadEventExecutorGroup`的构造方法中，通过` newChild(Executor executor, Object... args)`创建实际的`EventExecutor`，而这个方法是个抽象方法，具体实现在上述的子类中会创建对应的`EventLoop`（`NioEventLoop`、`EpollEventLoop`、`KQueueEventLoop`）。

```java
//MultithreadEventExecutorGroup
protected MultithreadEventExecutorGroup(int nThreads, Executor executor,
                                        EventExecutorChooserFactory chooserFactory, Object... args) {
    checkPositive(nThreads, "nThreads");

    if (executor == null) {
        executor = new ThreadPerTaskExecutor(newDefaultThreadFactory());
    }

    children = new EventExecutor[nThreads];

    for (int i = 0; i < nThreads; i ++) {
        boolean success = false;
        try {
            children[i] = newChild(executor, args);
            success = true;
        }......
    }

......
}

protected abstract EventExecutor newChild(Executor executor, Object... args) throws Exception;
```





### 未完待续...