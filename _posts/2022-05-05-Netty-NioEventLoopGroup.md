---
layout:     post
title:      "读 Netty NioEventLoopGroup"
subtitle:   " \"读 Netty NioEventLoopGroup\""
author:     "tablesheep"
date:       2022-05-05 19:20:00
header-style: text
catalog: true
tags:
- Java
- Netty
- 源码
---

在使用Netty时，都写过下面的代码，一开始菜跟懒，根本都不了解`EventLoopGroup`是个什么鬼东西。

```java
EventLoopGroup bossGroup = new NioEventLoopGroup(1);
EventLoopGroup workerGroup = new NioEventLoopGroup();
```



### EventLoopGroup

根据以下类图可以得出几个结论

![EventLoopGroup](/img/post/netty/EventLoopGroup.png)



- `EventLoopGroup`是一个支持定时的线程池
- 从`EventExecutorGroup`看出`EventLoopGroup`支持迭代功能，迭代的对象为`EventExecutor`（`EventExecutor`是一个特殊的`EventExecutorGroup`，可以大致理解`EventExecutor`是真正负责工作的，而`EventExecutorGroup`只是迭代（选择）出`EventExecutor`来工作，其实就是线程池和线程的关系，看完下面的具体实现懂了）



再看代码，`EventLoopGroup`将`next()`方法的对象进一步具体到了`EventLoop`，同时定义了几个`Channel`的`register`方法

```java
public interface EventLoopGroup extends EventExecutorGroup {
    /**
     * Return the next {@link EventLoop} to use
     */
    @Override
    EventLoop next();

    /**
     * Register a {@link Channel} with this {@link EventLoop}. The returned {@link ChannelFuture}
     * will get notified once the registration was complete.
     */
    ChannelFuture register(Channel channel);

    /**
     * Register a {@link Channel} with this {@link EventLoop} using a {@link ChannelFuture}. The passed
     * {@link ChannelFuture} will get notified once the registration was complete and also will get returned.
     */
    ChannelFuture register(ChannelPromise promise);

    /**
     * Register a {@link Channel} with this {@link EventLoop}. The passed {@link ChannelFuture}
     * will get notified once the registration was complete and also will get returned.
     *
     * @deprecated Use {@link #register(ChannelPromise)} instead.
     */
    @Deprecated
    ChannelFuture register(Channel channel, ChannelPromise promise);
}
```



接着用最常使用的`NioEventLoopGroup`看看



### NioEventLoopGroup

![NioEventLoopGroup](/img/post/netty/NioEventLoopGroup.png)



结合类图，我们从父类到子类讲。

#### AbstractEventExecutorGroup

先看`AbstractEventExecutorGroup`，它实现了线程池工作的所有方法，实现的也很简单，就是调用`next()`方法获取`EventExecutor`（放在`NioEventLoopGroup`里就是`NioEventLoop`），让它去执行。

```java
public abstract class AbstractEventExecutorGroup implements EventExecutorGroup {
    @Override
    public Future<?> submit(Runnable task) {
        return next().submit(task);
    }

    @Override
    public <T> Future<T> submit(Runnable task, T result) {
        return next().submit(task, result);
    }

    @Override
    public <T> Future<T> submit(Callable<T> task) {
        return next().submit(task);
    }

    @Override
    public ScheduledFuture<?> schedule(Runnable command, long delay, TimeUnit unit) {
        return next().schedule(command, delay, unit);
    }
    ......
}
```



#### MultithreadEventExecutorGroup

这个类最主要的方法有两个

- 构造方法，包含了整个线程池创建的核心，同时提供模板方法`newChild(Executor executor, Object... args)`给子类创建不同的`EventExecutor`，具体创建逻辑放后面`NioEventLoopGroup`创建流程介绍。

- 其次就是实现了`next()`方法，使用了`EventExecutorChooserFactory.EventExecutorChooser`去选择`EventExecutor`

  ```java
  @Override
  public EventExecutor next() {
      return chooser.next();
  }
  ```



#### MultithreadEventLoopGroup

这个类实现`EventLoopGroup`定义的方法

- 实现`EventLoopGroup#next()`同时也是重写父类的`next()`方法，因为`EventLoop`是`EventExecutor`子类
- 实现了`EventLoopGroup`中声明的`Channel`的`register`方法，实现的方法也一样，调用`next()`方法交由`EventLoop`进行`register`

```java
public abstract class MultithreadEventLoopGroup extends MultithreadEventExecutorGroup implements EventLoopGroup {
......

    @Override
    public EventLoop next() {
        return (EventLoop) super.next();
    }

    @Override
    protected abstract EventLoop newChild(Executor executor, Object... args) throws Exception;

    @Override
    public ChannelFuture register(Channel channel) {
        return next().register(channel);
    }

    @Override
    public ChannelFuture register(ChannelPromise promise) {
        return next().register(promise);
    }

    @Deprecated
    @Override
    public ChannelFuture register(Channel channel, ChannelPromise promise) {
        return next().register(channel, promise);
    }

}
```





#### NioEventLoopGroup

抛开构造方法，`NioEventLoopGroup`实现的就是一个`newChild`方法，创建具体的`EventLoop` （`NioEventLoop`）

```java
public class NioEventLoopGroup extends MultithreadEventLoopGroup {

   ......

    @Override
    protected EventLoop newChild(Executor executor, Object... args) throws Exception {
        SelectorProvider selectorProvider = (SelectorProvider) args[0];
        SelectStrategyFactory selectStrategyFactory = (SelectStrategyFactory) args[1];
        RejectedExecutionHandler rejectedExecutionHandler = (RejectedExecutionHandler) args[2];
        EventLoopTaskQueueFactory taskQueueFactory = null;
        EventLoopTaskQueueFactory tailTaskQueueFactory = null;

        int argsLength = args.length;
        if (argsLength > 3) {
            taskQueueFactory = (EventLoopTaskQueueFactory) args[3];
        }
        if (argsLength > 4) {
            tailTaskQueueFactory = (EventLoopTaskQueueFactory) args[4];
        }
        return new NioEventLoop(this, executor, selectorProvider,
                selectStrategyFactory.newSelectStrategy(),
                rejectedExecutionHandler, taskQueueFactory, tailTaskQueueFactory);
    }
}
```



大概了解了类的关系后，接下来可以看看`NioEventLoopGroup`的创建过程了



### NioEventLoopGroup创建

跟踪`NioEventLoopGroup`的构造方法，最终会来到`MultithreadEventExecutorGroup`构造方法中，主要干3几件事情

1. 检查或使用`DefaultThreadFactory`创建线程执行器（`DefaultThreadFactory`使用`FastThreadLocalRunnable`包装`Runnable`，使用`FastThreadLocalThread`执行任务）
2. 创建指定线程数量的`EventExecutor`，这里用了模板方法，具体的创建交由子类完成，至于线程数量没设置的话默认是cpu线程数*2（见`MultithreadEventLoopGroup#DEFAULT_EVENT_LOOP_THREADS`）
3. 创建`EventExecutor`选择器

```java
protected MultithreadEventExecutorGroup(int nThreads, Executor executor,
                                        EventExecutorChooserFactory chooserFactory, Object... args) {
    checkPositive(nThreads, "nThreads");

    //1.
    if (executor == null) {
        executor = new ThreadPerTaskExecutor(newDefaultThreadFactory());
    }

    children = new EventExecutor[nThreads];

    for (int i = 0; i < nThreads; i ++) {
        boolean success = false;
        try {
            //2.这里放到EventLoop的时候看
            children[i] = newChild(executor, args);
            success = true;
        } catch (Exception e) {
            // TODO: Think about if this is a good exception type
            throw new IllegalStateException("failed to create a child event loop", e);
        } finally {
            if (!success) {
                for (int j = 0; j < i; j ++) {
                    children[j].shutdownGracefully();
                }

                for (int j = 0; j < i; j ++) {
                    EventExecutor e = children[j];
                    try {
                        while (!e.isTerminated()) {
                            e.awaitTermination(Integer.MAX_VALUE, TimeUnit.SECONDS);
                        }
                    } catch (InterruptedException interrupted) {
                        // Let the caller handle the interruption.
                        Thread.currentThread().interrupt();
                        break;
                    }
                }
            }
        }
    }

    //3.使用工厂方法创建EventExecutorChooser，DefaultEventExecutorChooserFactory根据EventExecutor数量创建不同的EventExecutorChooser
    chooser = chooserFactory.newChooser(children);

    final FutureListener<Object> terminationListener = new FutureListener<Object>() {
        @Override
        public void operationComplete(Future<Object> future) throws Exception {
            if (terminatedChildren.incrementAndGet() == children.length) {
                terminationFuture.setSuccess(null);
            }
        }
    };

    for (EventExecutor e: children) {
        e.terminationFuture().addListener(terminationListener);
    }

    //复制一份EventExecutor只读集合
    Set<EventExecutor> childrenSet = new LinkedHashSet<EventExecutor>(children.length);
    Collections.addAll(childrenSet, children);
    readonlyChildren = Collections.unmodifiableSet(childrenSet);
}
```



### 总结

- `EventLoopGroup`其实就是个线程池，具体的工作会交由`EventLoop`去执行，它只负责选择`EventLoop`
- 与Java线程池对比，它的线程数量是稳定的，同时`EventLoopGroup`少了一些东西，任务队列、拒绝策略（其实并不是没有，只是不在`EventLoopGroup`中，而是在`EventLoop`中，可以回头看看`NioEventLoopGroup#newChild`）