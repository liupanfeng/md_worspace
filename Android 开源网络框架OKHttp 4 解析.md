### Android 开源网络框架OKHttp 4 解析

[TOC]

从OkHttp4开始实现改成Kotlin，OkHttp是Android使用最广泛的网络框架。从Android4.4开始HttpURLConnection的底层实现采用的是OKHttp

还有一个框架Retrofit，这个框架也比较火，但是需要明确：这个框架是对OkHttp的封装，框架本身并不具备网络请求的能力，框架增加了线程切换、解析器适配器等让OkHttp更加容易使用。

#### 1.Http2.0协议主要增加的优化点：

* 头部压缩
* serverPush
* 多路复用：增加了二进制数据帧，这个帧里边会包含一个顺序标识。

http1.1 增加了keep-alive 的功能，这个可以通过客户端控制，也可以通过服务端控制。

Connection  keep-alive  表示开启保持连接的功能，close 表示关闭。

Http1.1的版本，发起的请求必须是串行的。

Http2一下的版本，OkHttp增加了一个对象池，复用连接。

#### 2.OkHttp支持的内容

* OkHttp支持Http2.0，并允许对同一个主机的所有请求共享同一个套接字
* 如果不是Http2.0，通过使用对象连接池，减少请求耗时
* 请求默认使用Gzip压缩
* 增加了响应缓存的功能，避免重复网络请求。默认是关闭的，需要在构建客户端的时候设置cache开启。只支持Get请求的缓存，不支持post的。
* 开启缓存需要执行文件的路径和缓存的大小。

#### 3.OkHttp的使用流程

* 构建OkHttpClient，可以直接new，也可以通过Builder()来增加各种配置
  * 配置cache缓存，这个默认是关闭的，只支持Get不支持Post，传入缓存文件以及大小就可以
  * 配置超市时间
  * 配置拦截器
* 创建Request 通过Builder来创建Request，可以增加各种配置
* 然后使用得到的okHttpClient 调用newcall方法将request传进去就得到一个Call对象
* 然后执行得到的call对象就可以
  * 通过execute来执行，是同步执行，同步执行直接可以得到一个结果
  * 通过enqueue来执行，表示异步执行，异步执行需要传入CallBack
* 然后将Call交给分发器dispatcher去分发执行，dispatcher：内部维护的队列与线程池，完成请求调配。
* 分发执行之后再经过拦截器（完成整个请求过程），就可以得到请求的结果了，不管是成功还是失败。

```kotlin
  val okHttpClient = OkHttpClient.Builder()
//        .cache(Cache(File("/"),1024))  //设置缓存
            // .addInterceptor()  //添加拦截器
            //.callTimeout() 设置请求超时
            //.connectTimeout() 设置连接超时
            //.dispatcher()//设置分发器
            // .dns() 设置自己的dns
            .eventListener(OkHttpListener()) //设置事件监听器
            .build()
        val request = Request.Builder()
            .url("https://www.baidu.com/")  //设置网络连接
//            .addHeader("","")//添加请求头
//            .cacheControl(CacheControl.FORCE_NETWORK) //添加擦车策略
            // .post(RequestBody) //默认是Get请求，如果需要使用post请求，需要设置一个RequestBody
            .build()

        val newCall = okHttpClient.newCall(request)  //得到Call 对象

        //  val result=newCall.execute()//表示同步执行 ,同步可以直接得到一个结果
//         println(result.isSuccessful)   isSuccessful通过这个来判断是否请求成功
//         result.body//  拿到请求体，可以通过流的方式进行读取
//        result.close()

        newCall.enqueue(object : Callback {
            override fun onFailure(call: Call, e: IOException) {
                //请求失败的回调
                Log.e("lpf", "onFailure-------->请求失败")
            }

            override fun onResponse(call: Call, response: Response) {
                //请求成功的回调
                if (response.isSuccessful) {
                    Log.d("lpf", "isSuccessful")
                    response.body?.let { Log.d("lpf", it.string()) }
                } else {
                    Log.e("lpf", "请求失败")
                }

            }

        })   //这个表示异步请求 异步请求需要传入一个回调
//////////////////////////////事件回调////////////////////////////
 class OkHttpListener:EventListener(){
        override fun callStart(call: Call) {
            super.callStart(call)
            Log.e("lpf","callStart-------->")
        }

        override fun callEnd(call: Call) {
            super.callEnd(call)
            Log.e("lpf","callEnd-------->")
        }

        override fun connectStart(
            call: Call,
            inetSocketAddress: InetSocketAddress,
            proxy: Proxy
        ) {
            super.connectStart(call, inetSocketAddress, proxy)
            Log.e("lpf","connectStart-------->")
        }

        override fun connectEnd(
            call: Call,
            inetSocketAddress: InetSocketAddress,
            proxy: Proxy,
            protocol: Protocol?
        ) {
            super.connectEnd(call, inetSocketAddress, proxy, protocol)
            Log.e("lpf","connectEnd-------->")
        }

        override fun dnsStart(call: Call, domainName: String) {
            super.dnsStart(call, domainName)
            Log.e("lpf","dnsStart-------->")
        }

        override fun dnsEnd(
            call: Call,
            domainName: String,
            inetAddressList: List<InetAddress>
        ) {
            super.dnsEnd(call, domainName, inetAddressList)
            Log.e("lpf","dnsEnd-------->")
        }

        override fun requestBodyStart(call: Call) {
            super.requestBodyStart(call)
            Log.e("lpf","requestBodyStart-------->")
        }

        override fun requestBodyEnd(call: Call, byteCount: Long) {
            super.requestBodyEnd(call, byteCount)
            Log.e("lpf","requestBodyEnd-------->")
        }

        override fun responseHeadersStart(call: Call) {
            super.responseHeadersStart(call)
            Log.e("lpf","responseHeadersStart-------->")
        }

        override fun requestHeadersEnd(call: Call, request: Request) {
            super.requestHeadersEnd(call, request)
            Log.e("lpf","requestHeadersEnd-------->")
        }
    }

```

打印的事件回调结果

```shell
2022-05-17 22:53:29.127 8954-8954/com.test.kotlin_test E/lpf: callStart-------->
2022-05-17 22:53:29.132 8954-9191/com.test.kotlin_test E/lpf: dnsStart-------->
2022-05-17 22:53:29.193 8954-9191/com.test.kotlin_test E/lpf: dnsEnd-------->
2022-05-17 22:53:29.197 8954-9191/com.test.kotlin_test E/lpf: connectStart-------->
2022-05-17 22:53:29.289 8954-9191/com.test.kotlin_test E/lpf: connectEnd-------->
2022-05-17 22:53:29.292 8954-9191/com.test.kotlin_test E/lpf: requestHeadersEnd-------->
2022-05-17 22:53:29.319 8954-9191/com.test.kotlin_test E/lpf: responseHeadersStart-------->
2022-05-17 22:53:29.323 8954-9191/com.test.kotlin_test D/lpf: isSuccessful
2022-05-17 22:53:29.327 8954-9191/com.test.kotlin_test E/lpf: callEnd-------->
```



#### 4.OkHttp请求过程源码解析

##### 4.1.创建一个newCall 对象

```kotlin
/** Prepares the [request] to be executed at some point in the future. */
override fun newCall(request: Request): Call = RealCall(this, request, forWebSocket = false)
```



##### 4.2.newCall执行同步或者异步请求，异步请求的主线流程

```kotlin
override fun enqueue(responseCallback: Callback) {
 
  check(executed.compareAndSet(false, true)) { "Already Executed" }
  callStart()  
  //事件分发器dispatcher 要分发任务了（敲黑板，重点在这里）
  client.dispatcher.enqueue(AsyncCall(responseCallback))
}
```

* check：同一个RealCall只能执行一次 ，使用的CAS同步机制的中的原子变量 AtomicBoolean 来实现的
* callStart：这里就是事件回调 如果设置的事件回调接口就能收到事件
* client.dispatcher.enqueue：开始执行事件分发，AsyncCall：是一个Runnable，内部记录一个成员变量callsPerHost，类型是AtomicInteger，这个变量是用来实现限制同一个Host的同时请求的数量，默认是5。



##### 4.3.Dispatcher的enqueue方法

```kotlin
//事件分发
internal fun enqueue(call: AsyncCall) {
  synchronized(this) {
    //将RealCall加入到队列
    readyAsyncCalls.add(call)

    // Mutate the AsyncCall so that it shares the AtomicInteger of an existing running call to
    // the same host.
    if (!call.call.forWebSocket) { //如果不是websocket
      //查找之前请求的call的host跟我们这次请求的是否是一样的
      val existingCall = findExistingCallWithHost(call.host)
      //如果存在就拿到之前的call的host
      if (existingCall != null) call.reuseCallsPerHostFrom(existingCall)
    }
  }
  promoteAndExecute()
}
```

* 保证线程安全，加了内置锁synchronized（不同维度上来说也属于非公平锁，也属于悲观锁，重量级锁）
* 将RealCall添加到等待队列
* 如果不是websocket，查看当前newCall的host跟正在运行的host是否一样，如果一样就将callsPerHost +1，是为了实现同一个Host的请求数量限制功能。

这个方法就是将newCall添加到了等待队列，然后将Host的数量做了运算。



##### 4.4.promoteAndExecute方法

```kotlin
/**
 * 将符合条件的调用从 [readyAsyncCalls] 提升到 [runningAsyncCalls] 并在 executor 服务上运行它们。
 * 不能同步调用，因为执行调用可以调用用户代码。
 * 如果调度程序当前正在运行调用，则为 true
 */
private fun promoteAndExecute(): Boolean {
  this.assertThreadDoesntHoldLock()
  //定义一个临时缓存
  val executableCalls = mutableListOf<AsyncCall>()
  val isRunning: Boolean
  synchronized(this) {
    val i = readyAsyncCalls.iterator()
      //遍历等待连接池中的RealCall
    while (i.hasNext()) {
      val asyncCall = i.next()

      //限制 最多同时执行的任务数量
      if (runningAsyncCalls.size >= this.maxRequests) break // Max capacity.
      //同一个Host同时执行的任务数量最多是5个 如果超过5个了，continue
      if (asyncCall.callsPerHost.get() >= this.maxRequestsPerHost) continue // Host max capacity.

      i.remove() 
      asyncCall.callsPerHost.incrementAndGet()
      executableCalls.add(asyncCall)  //将newCall添加到缓存数组里边
      runningAsyncCalls.add(asyncCall) //添加到正在运行的队列
    }
    isRunning = runningCallsCount() > 0
  }
	
    //遍历缓存数组
  for (i in 0 until executableCalls.size) {
    val asyncCall = executableCalls[i]
    asyncCall.executeOn(executorService)
  }

  return isRunning
}
```

* 遍历等待队列中的call，等待队列使用的ArrayDeque，在没有外部同步的情况下，它们不支持多线程并发访问。所以加了内置锁synchronized
* runningAsyncCalls这个队列的数量如果已经等于maxRequests默认是64，直接break，就是说同时请求的任务最多是64个
* callsPerHost.get()同一个Host请求的数量如果等于maxRequestsPerHost默认是5，直接continue，同时请求同一个Host的数量不能大于5个
* 后边就是将call任务从等待任务移除，并添加到临时缓存数组和正在运行的队列，并遍历缓存数组，执行call的executeOn方法



##### 4.5.executeOn方法

```kotlin
fun executeOn(executorService: ExecutorService) {
  client.dispatcher.assertThreadDoesntHoldLock()

  var success = false
  try {
    //将call放置到线程池 就执行run方法了
    executorService.execute(this)
    success = true
  } catch (e: RejectedExecutionException) {
    val ioException = InterruptedIOException("executor rejected")
    ioException.initCause(e)
    noMoreExchanges(ioException)
    responseCallback.onFailure(this@RealCall, ioException)
  } finally {
    if (!success) { //当这个call执行完成了，最终会执行这里
      client.dispatcher.finished(this) // This call is no longer running!
    }
  }
}
```

* 将RealCall放置到线程池去执行，接下来就是执行Runnable的run方法
* client.dispatcher.finished(this) 执行完成会调用分发器的finish方法，将将callsPerHost减去1，runningAsyncCalls减去1，这样就可以执行其他的任务了



##### 4.6.RealCall的run方法

```kotlin
 override fun run() {
    threadName("OkHttp ${redactedUrl()}") {
      var signalledCallback = false
      timeout.enter()
      try {
        //通过执行getResponseWithInterceptorChain 获取到请求的结果（重点--拦截器来了）
        val response = getResponseWithInterceptorChain()
        signalledCallback = true
        responseCallback.onResponse(this@RealCall, response)
      } catch (e: IOException) {
        if (signalledCallback) {
          // Do not signal the callback twice!
          Platform.get().log("Callback failure for ${toLoggableString()}", Platform.INFO, e)
        } else {
          responseCallback.onFailure(this@RealCall, e)
        }
      } catch (t: Throwable) {
        cancel()
        if (!signalledCallback) {
          val canceledException = IOException("canceled due to $t")
          canceledException.addSuppressed(t)
          responseCallback.onFailure(this@RealCall, canceledException)
        }
        throw t
      } finally {
        client.dispatcher.finished(this)
      }
    }
  }
}
```

* 通过执行getResponseWithInterceptorChain 获取到请求的结果，这个方法很关键，dispatcher分发完成之后，就需要拦截器了，这里就是调用拦截器。
* 下面是异常的情况的回调处理。
* 不管成功还是失败，最终还是会调用 client.dispatcher.finished(this) ，让同一个host的请求数量和请求队列数量减去一个



##### 4.7.getResponseWithInterceptorChain方法

```kotlin
  @Throws(IOException::class)
  internal fun getResponseWithInterceptorChain(): Response {
    // Build a full stack of interceptors.
    val interceptors = mutableListOf<Interceptor>()
    //添加我们自定义的 Interceptor 添加到请求器数组里边
    interceptors += client.interceptors
    //添加重试和跟进拦截器
    interceptors += RetryAndFollowUpInterceptor(client)
    //桥梁拦截器 ：应用程序代码到网络代码的桥梁
    interceptors += BridgeInterceptor(client.cookieJar)
    //缓存拦截器 处理来自缓存的请求并将响应写入缓存
    interceptors += CacheInterceptor(client.cache)
    //连接拦截器 打开到目标服务器的连接并继续到下一个拦截器。网络可能用于返回的响应，或使用条件 GET 验证缓存的响应。
    interceptors += ConnectInterceptor
    if (!forWebSocket) {
      //添加我们自定义的NetworkInterceptor
      interceptors += client.networkInterceptors
    }
    //这是链中的最后一个拦截器。它对服务器进行网络调用
    interceptors += CallServerInterceptor(forWebSocket)

    //将拦截器构建成 拦截器链RealInterceptorChain
    val chain = RealInterceptorChain(
      call = this,
      interceptors = interceptors,
      index = 0,
      exchange = null,
      request = originalRequest,
      connectTimeoutMillis = client.connectTimeoutMillis,
      readTimeoutMillis = client.readTimeoutMillis,
      writeTimeoutMillis = client.writeTimeoutMillis
    )

    var calledNoMoreExchanges = false
    try {
      //通过责任链的方式，proceed request 得到response
      val response = chain.proceed(originalRequest)
      if (isCanceled()) {
        response.closeQuietly()
        throw IOException("Canceled")
      }
      return response
    } catch (e: IOException) {
      calledNoMoreExchanges = true
      throw noMoreExchanges(e) as Throwable
    } finally {
      if (!calledNoMoreExchanges) {
        noMoreExchanges(null)
      }
    }
  }
```

* interceptors：添加我们自定义的 Interceptor 添加到请求器数组里边
* **RetryAndFollowUpInterceptor**(client)：添加重试重定向拦截器
* **BridgeInterceptor**：桥接拦截器 ：应用程序代码到网络代码的桥梁，处理Header和Body
* **CacheInterceptor**：缓存拦截器 处理来自缓存的请求并将响应写入缓存
* **ConnectInterceptor**：如果缓存未命中，连接拦截器，专门用来管理跟服务端的连接的
* networkInterceptors：如果不是webSocket，添加我们自定义的NetworkInterceptor
* **CallServerInterceptor**：这是链中的最后一个拦截器。拿到上一步跟服务器的连接，跟服务端进行交互，并得到请求的结果。
* 构建拦截器链RealInterceptorChain，并拦截器链调用proceed方法执行originalRequest，并返回结果

OkHttp默认有5个拦截器，通过拦截器真正的执行网络请求，并返回数据。这块采用的是责任链的设计模式来设计的。



##### 4.8.同步的情况就比较简单了，execute方法

```
override fun execute(): Response {
  check(executed.compareAndSet(false, true)) { "Already Executed" }

  timeout.enter()
  callStart()
  try {
    client.dispatcher.executed(this)
    return getResponseWithInterceptorChain()
  } finally {
    client.dispatcher.finished(this)
  }
}
```

* 检测realCall是否已经执行过了
* executed直接加到运行队列中，同步的情况就没有等待队列了
* 然后执行拦截器的相关逻辑就拿到结果了



#### 5.OkHttp重要点总结：

##### 5.1.OkHttp请求的流程概述

* 构建一个OkHttpClient,构建一个Request，通过调用newcall方法将request当作参数传进去，得到一个readlCall对象
* 执行realcall的enqueue方法（异步）或者execute（同步），不管是同步还是异步call只能执行一次
* 异步将realcall封装成AsynCall，执行分发器dispatcher的enqueue方法
* 先添加到等待队列，然后检查是否可以将任务添加到正在执行的队列中去
  * 当前正在执行的异步任务不得大于64
  * 从ready中迭代得到的等待执行的任务host在正在进行执行的任务不得大于5个
* 然后放置任务到线程池去执行
* 异步任务的执行需要交给拦截器去执行，拦截器是责任链的设计模式，最终得到结果，系统默认提供了5大拦截器
* 同步执行方式就比较简单了，任务之间添加到正在运行的队列中分发器只是对任务做一个记录，执行完了就会删除，同步任务同样需要经过拦截器。

##### 5.2.OkHttp分发器的执行流程

* 分发器dispatcher对同步任务的执行很简单，只是对任务做一个记录的过程，执行完成就会移除
* 分发器对于异步任务添加到等待队列，然后检查是否可以将任务添加到正在执行的队列中去，检查的条件就是：
  * 当前正在执行的异步任务不得大于64
  * 从ready中迭代得到的等待执行的任务host在正在进行执行的任务不得大于5个

##### 5.3.okHttp拦截器的执行流程

采用责任链的设计模式，是一种行为性设计模式，让我们的任务请求者和发起者进行解耦合，我们不需要关心责任链是如何处理的。

##### 5.4.应用拦截器和网络拦截器的异同点

* 应用拦截器是添加在拦截器的最前面，应用拦截器最先接到执行拦截，最后得到结果反馈，在应用拦截器中拿到的是用户发送的请求数据，这些数据未被其他拦截器处理
* 网络拦截器是添加在CallServer拦截器之前的，会倒数第二个接到请求，第二个接到结果，在网络拦截器中我们可以拿到真正发给服务端的数据