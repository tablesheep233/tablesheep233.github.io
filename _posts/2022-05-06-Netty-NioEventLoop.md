---
layout:     post
title:      "读 Netty NioEventLoop"
subtitle:   " \"读 Netty NioEventLoop\""
author:     "tablesheep"
date:       2022-05-06 19:20:00
header-style: text
catalog: true
tags:
- Java
- Netty
- 源码
---







在看完了`EventLoopGroup`后，接着看具体工作的类`EventLoop`。

### EventLoop

![EventLoop](/img/post/netty/EventLoop.png)

#### EventExecutor

`EventLoop`接口最主要的就是继承了`EventExecutor`，而`EventExecutor`则继承了`EventExecutorGroup`，同时扩展了一些方法。

```java
public interface EventExecutor extends EventExecutorGroup {

    /**
     * Returns a reference to itself.
     * 返回自身
     */
    @Override
    EventExecutor next();

    /**
     * Return the {@link EventExecutorGroup} which is the parent of this {@link EventExecutor},
     * 返回父EventExecutorGroup从这个方法可看出 EventExecutorGroup 跟 EventExecutor 是父子关系
     * 就如 在 NioEventLoopGroup 中会创建多个 NioEventLoop一样
     */
    EventExecutorGroup parent();

    /**
     * Calls {@link #inEventLoop(Thread)} with {@link Thread#currentThread()} as argument
     */
    boolean inEventLoop();

    /**
     * Return {@code true} if the given {@link Thread} is executed in the event loop,
     * {@code false} otherwise.
     * 判断线程是否在 event loop (事件循环)中
     */
    boolean inEventLoop(Thread thread);

    /**
     * Return a new {@link Promise}.
     */
    <V> Promise<V> newPromise();

    /**
     * Create a new {@link ProgressivePromise}.
     */
    <V> ProgressivePromise<V> newProgressivePromise();

    /**
     * Create a new {@link Future} which is marked as succeeded already. So {@link Future#isSuccess()}
     * will return {@code true}. All {@link FutureListener} added to it will be notified directly. Also
     * every call of blocking methods will just return without blocking.
     */
    <V> Future<V> newSucceededFuture(V result);

    /**
     * Create a new {@link Future} which is marked as failed already. So {@link Future#isSuccess()}
     * will return {@code false}. All {@link FutureListener} added to it will be notified directly. Also
     * every call of blocking methods will just return without blocking.
     */
    <V> Future<V> newFailedFuture(Throwable cause);
}
```



### NioEventLoop

![NioEventLoop](/img/post/netty/NioEventLoop.png)



接着看`NioEventLoop`实现，还是自上而下

#### AbstractEventExecutor

`AbstractEventExecutor`主要实现了线程池的大部分方法以及`EventExecutor`的大部分方法（除了`inEventLoop(Thread thread)`）



#### AbstractScheduledEventExecutor

`AbstractScheduledEventExecutor`扩展了`AbstractEventExecutor`，使其支持定时调度执行。



#### SingleThreadEventExecutor

使用单线程实现的`EventExecutor`，一个线程拥有一个独立的任务队列。

```java
//重要的成员变量 
private final Queue<Runnable> taskQueue;  //线程队列

private volatile Thread thread; //执行event loop 的线程

private final Executor executor;  //这个变量很有意思，就是靠它创建的线程
```



#### 线程的启动

##### ThreadPerTaskExecutor

`executor`变量默认情况下是`ThreadPerTaskExecutor`（见`MultithreadEventExecutorGroup`构造方法）

```java
public final class ThreadPerTaskExecutor implements Executor {
    private final ThreadFactory threadFactory;

    public ThreadPerTaskExecutor(ThreadFactory threadFactory) {
        this.threadFactory = ObjectUtil.checkNotNull(threadFactory, "threadFactory");
    }

    @Override
    public void execute(Runnable command) {
        //使用线程工厂创建线程并启动
        threadFactory.newThread(command).start();
    }
}
```

##### SingleThreadEventExecutor#doStartThread

启动线程的方法，一开始看会觉得很绕，看懂了会觉得6。

```java
private void doStartThread() {
    assert thread == null;
    //当前线程为null，即未启动的情况下，使用executor去执行，而executor会创建一个新的线程去执行
    executor.execute(new Runnable() {
        @Override
        public void run() {
            //而任务执行的第一步就是将当前线程（executor创建出来的那个）引用设置给 thread变量（EventLoop持有的线程）
            thread = Thread.currentThread();
           ......
               
            try {
                //启动模板方法run()执行任务
               SingleThreadEventExecutor.this.run();
               success = true;
           } 
        }
    });
}
```







#### SingleThreadEventLoop

`SingleThreadEventLoop`主要是实现了`EventLoopGroup`中定义的`register`方法

```java
@Override
public EventLoopGroup parent() {
    return (EventLoopGroup) super.parent();
}

@Override
public EventLoop next() {
    return (EventLoop) super.next();
}

@Override
public ChannelFuture register(Channel channel) {
    return register(new DefaultChannelPromise(channel, this));
}

@Override
public ChannelFuture register(final ChannelPromise promise) {
    ObjectUtil.checkNotNull(promise, "promise");
    promise.channel().unsafe().register(this, promise);
    return promise;
}
```



### NioEventLoop

`NioEventLoop`实现了`SingleThreadEventExecutor#run()`方法，是整个事件循环具体执行内容的实现

```java
@Override
protected void run() {
    int selectCnt = 0;
    for (;;) { //是一个无限的循环
 
        //暂时略
        .......
    }
}
```



