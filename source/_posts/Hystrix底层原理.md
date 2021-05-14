---

title: Hystrix底层原理
date: 2021-05-03 20:31:56
tags: ['中间件', '底层原理', 'Hystrix']
---

### Hystrix 是什么？

在分布式系统中，每个服务都可能会调用很多其他服务，被调用的那些服务就是**依赖服务**，有的时候某些依赖服务出现故障也是很正常的。

### Hystrix可以提供那些功能

Hystrix可以提供：

- 服务熔断
- 服务降级
- 服务限流

### Hystrix 更加细节的设计原则

- 阻止任何一个依赖服务耗尽所有的资源，比如 tomcat 中的所有线程资源。
- 避免请求排队和积压，采用限流和 `fail fast` 来控制故障。
- 提供 fallback 降级机制来应对故障。
- 使用资源隔离技术，比如 `bulkhead`（舱壁隔离技术）、`swimlane`（泳道技术）、`circuit breaker`（断路技术）来限制任何一个依赖服务的故障的影响。

### HystrixCommand的调用方法

- **execute**：同步堵塞，调用了queue().get()方法，execute()执行完后，会创建一个新线程运行run()；
- **queue**：异步非堵塞，它调用了toObservable().toBlocking().toFuture()方法，queue()执行完后，会创建一个新线程运行run()。Future.get()是堵塞的，它等待run()运行完才返回结果；
- **observe()** ：异步热响应调用，它调用了toObservable().subscribe(subject)方法，observe()执行完后，会创建一个新线程运行run()。toBlocking().single()是堵塞的，需要等run()运行完才返回结果；
- **toObservable()**：异步的冷响应调用，该方法不会主动创建线程运行run()，只有当调用了toBlocking().single()或subscribe()时，才会去创建线程运行run()。

### 

Hystrix 实现资源隔离，主要有两种技术：

- 线程池

  - **第三方客户端（执行Hystrix的run()方法）会在单独的线程执行**，会与调用的该任务的线程进行隔离，以此来防止调用者调用依赖所消耗的时间过长而阻塞调用者的线程。

    使用线程隔离的好处：

    - 应用程序可以不受失控的第三方客户端的威胁，如果第三方客户端出现问题，可以通过降级来隔离依赖。
    - 当失败的客户端服务恢复时，线程池将会被清除，应用程序也会恢复，而不至于使整个Tomcat容器出现故障。
    - 如果一个客户端库的配置错误，线程池可以很快的感知这一错误（通过增加错误比例，延迟，超时，拒绝等），并可以在不影响应用程序的功能情况下来处理这些问题（可以通过动态配置来进行实时的改变）。
    - 如果一个客户端服务的性能变差，可以通过改变线程池的指标（错误、延迟、超时、拒绝）来进行属性的调整，并且这些调整可以不影响其他的客户端请求。

- 信号量

  - 信号隔离是通过限制依赖服务的并发请求数，来控制隔离开关。信号隔离方式下，**业务请求线程和执行依赖服务的线程是同一个线程**（例如Tomcat容器线程）

> 默认情况下，Hystrix 使用线程池模式

### 线程池与信号量区别

线程池：使用的是Hystrix内部维护的线程池，每一个依赖服务都会拥有自己的小线程池

信号量隔离：是使用发起请求的线程去调用这个依赖服务，它只是一道关卡，信号量有多少，就允许多少个tomcat线程通过，然后执行

![image-20210306220714378](Hystrix%E5%BA%95%E5%B1%82%E5%8E%9F%E7%90%86/image-20210306220714378.png)

**适用场景**：

- **线程池技术**，适合绝大多数场景，比如说我们对依赖服务的网络请求的调用和访问、需要对调用的 timeout 进行控制（捕捉 timeout 超时异常）。
- **信号量技术**，适合说你的访问不是对外部依赖的访问，而是对内部的一些比较复杂的业务逻辑的访问，并且系统内部的代码，其实不涉及任何的网络请求，那么只要做信号量的普通限流就可以了，因为不需要去捕获 timeout 类似的问题。

## Hystrix 隔离策略细粒度控制

- **command key** ：代表了一类 command，一般来说，代表了下游依赖服务的某个接口。
- **command group** ，代表了某一个下游依赖服务，这是很合理的，一个依赖服务可能会暴露出来多个接口，每个接口就是一个 command key。command group 在逻辑上对一堆 command key 的调用次数、成功次数、timeout 次数、失败次数等进行统计，可以看到某一个服务整体的一些访问情况。**一般来说，推荐根据一个服务区划分出一个线程池，command key 默认都是属于同一个线程池的。**
- **command thread pool**：代表了每个command key的线程池，默认大小为10
- **coreSize**：设置线程池的大小，默认是 10。一般来说，用这个默认的 10 个线程大小就够了
- **queueSizeRejectionThreshold**：控制queue满了之后拒绝的阈值，和最大队列数量(maxQueueSize)不是一个意思，默认为 5
- 查看默认值的位置：在HystrixThreadPoolProperties类中查看

> 说白点，就是说如果你的 command key 要用自己的线程池，可以定义自己的 thread pool key，就 ok 了。

### Hystrix 执行时内部原理

![new-hystrix-process](Hystrix%E5%BA%95%E5%B1%82%E5%8E%9F%E7%90%86/new-hystrix-process.jpg)

- ##### 1.创建 command

- ##### 2.调用 command 执行方法

  要执行 command，可以在 4 个方法中选择其中的一个：execute()、queue()、observe()、toObservable()。

- ##### 3.检查是否开启缓存（不太常用）

  如果这个 command 开启了请求缓存 Request Cache，而且这个调用的结果在缓存中存在，那么直接从缓存中返回结果。否则，继续往后的步骤。

- ##### 4.检查是否开启了断路器

  检查这个 command 对应的依赖服务是否开启了断路器。如果断路器被打开了，那么 Hystrix 就不会执行这个 command，而是直接去执行 fallback 降级机制，返回降级结果。

- ##### 5.检查线程池/队列/信号量是否已满

  如果这个 command 线程池和队列已满，或者 semaphore 信号量已满，那么也不会执行 command，而是直接去调用 fallback 降级机制

- ##### 6.执行 command

  调用 HystrixObservableCommand 对象的 construct() 方法，或者 HystrixCommand 的 run() 方法来实际执行这个 command。

  如果是采用线程池方式，并且 HystrixCommand.run() 或者 HystrixObservableCommand.construct() 的执行时间超过了 timeout 时长的话，那么 command 所在的线程会抛出一个 TimeoutException，这时会执行 fallback 降级机制，不会去管 run() 或 construct() 返回的值了。另一种情况，如果 command 执行出错抛出了其它异常，那么也会走 fallback 降级。这两种情况下，Hystrix 都会发送异常事件给断路器统计

- ##### 7.断路健康检查

  Hystrix 会把每一个依赖服务的调用成功、失败、Reject、Timeout 等事件发送给 circuit breaker 断路器。断路器就会对这些事件的次数进行统计，根据异常事件发生的比例来决定是否要进行断路（熔断）。如果打开了断路器，那么在接下来一段时间内，会直接断路，返回降级结果。

  如果在之后，断路器尝试执行 command，调用没有出错，返回了正常结果，那么 Hystrix 就会把断路器关闭。

- ##### 8.调用 fallback 降级机制

  在以下几种情况中，Hystrix 会调用 fallback 降级机制。

  - 断路器处于打开状态；
  - 线程池/队列/semaphore 满了；
  - command 执行超时；
  - run() 或者 construct() 抛出异常。

  不同的 command 执行方式，其 fallback 为空或者异常时的返回结果不同。

  - 对于 execute()，直接抛出异常。
  - 对于 queue()，返回一个 Future，调用 get() 时抛出异常

### 深入 Hystrix 断路器执行原理

Hystrix 断路器有三种状态

- 关闭（Closed）：调用依赖服务的请求正常通过
- 打开（Open）：阻断对依赖服务的请求调用，直接走FallBack逻辑
- 半开（Half-Open）：

状态转换关系如下：

![image-20210306231414813](Hystrix%E5%BA%95%E5%B1%82%E5%8E%9F%E7%90%86/image-20210306231414813.png)

运行机制：在一次统计时间的 滑动窗口（默认10秒） 中，最少有20次请求通过断路器，且异常比例超过50%，就会进入打开状态，在打开断路器后，接下来的3000毫秒都不会去请求依赖服务，而是直接走的fallBack机制

3000毫秒后，会进入半开状态，新的请求依赖服务会被执行，如果执行成功，就进入关闭状态，如果失败，则再次进入打开状态

### 两种最经典的降级机制

- 纯内存数据
  在降级逻辑中，你可以在内存中维护一个 ehcache，作为一个纯内存的基于 LRU 自动清理的缓存，让数据放在缓存内。如果说外部依赖有异常，fallback 这里直接尝试从 ehcache 中获取数据。
- 默认值
  fallback 降级逻辑中，也可以直接返回一个默认值。