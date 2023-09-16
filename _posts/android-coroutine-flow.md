---
layout: post
title:  "Android : 從 Codelabs 學 Coroutine Flow"
date:   2022-08-08 16:15:07 +0800
categories: [android, flow]
---

# 背景

會來看這裡的應該已經知道如何使用 **Coroutine** 了。

所以我們就從 [codelabs](https://developer.android.com/codelabs/advanced-kotlin-coroutines#0) 來學 Flow 的使用方法。

這篇文章將會針對 codelabs 的實作來分析。 然後還會解析一下 **Flow** 的實作為何。

# Codelabs 的分析

這個 codelabs 主要會做的是與 **Room** 的互動以及資料的顯示。

我們會從資料的建立到資料的顯示與更新來一一講解。

## 從伺服器取得資料

為了建立資料庫，我們需要從 github 取得：

```kotlin
interface SunflowerService {
    @GET("googlecodelabs/kotlin-coroutines/master/advanced-coroutines-codelab/sunflower/src/main/assets/plants.json")
    suspend fun getAllPlants() : List<Plant>

    @GET("googlecodelabs/kotlin-coroutines/master/advanced-coroutines-codelab/sunflower/src/main/assets/custom_plant_sort_order.json")
    suspend fun getCustomPlantSortOrder() : List<Plant>
}
```

### 進行連線

然後我們需要使用 **Retrofit** 進行連結 ：

```kotlin
private val retrofit = Retrofit.Builder()
    .baseUrl("https://raw.githubusercontent.com/")
    .client(OkHttpClient())
    // A converter which uses Gson for JSON.
    .addConverterFactory(GsonConverterFactory.create())
    .build()

private val sunflowerService = retrofit.create(SunflowerService::class.java)
```

通過 `getAllPlants` 、 `getCustomPlantSortOrder` 與 `customPlantSortOrder` 我們可以從 github 中的 JSON 取得資料，並進行轉換：

```kotlin
suspend fun allPlants(): List<Plant> = withContext(Dispatchers.Default) {
    val result = sunflowerService.getAllPlants()
    result.shuffled()
}

suspend fun plantsByGrowZone(growZone: GrowZone) = withContext(Dispatchers.Default) {
    val result = sunflowerService.getAllPlants()
    result.filter { it.growZoneNumber == growZone.number }.shuffled()
}

suspend fun customPlantSortOrder(): List<String> = withContext(Dispatchers.Default) {
    val result = sunflowerService.getCustomPlantSortOrder()
    result.map { plant -> plant.plantId }
}
```


#### GsonConverterFactory 的實作

其中的 **GsonConverterFactory** 會實作 `responseBodyConverter` 與 `requestBodyConverter` 如下：

```java
@Override
public Converter<ResponseBody, ?> responseBodyConverter(
    Type type, Annotation[] annotations, Retrofit retrofit) {
  TypeAdapter<?> adapter = gson.getAdapter(TypeToken.get(type));
  return new GsonResponseBodyConverter<>(gson, adapter);
}

@Override
public Converter<?, RequestBody> requestBodyConverter(
    Type type,
    Annotation[] parameterAnnotations,
    Annotation[] methodAnnotations,
    Retrofit retrofit) {
  TypeAdapter<?> adapter = gson.getAdapter(TypeToken.get(type));
  return new GsonRequestBodyConverter<>(gson, adapter);
}
```

而 **GsonRequestBodyConverter** 則會實作 `convert` ：

```java
@Override
public T convert(ResponseBody value) throws IOException {
  JsonReader jsonReader = gson.newJsonReader(value.charStream());
  try {
    T result = adapter.read(jsonReader);
    if (jsonReader.peek() != JsonToken.END_DOCUMENT) {
      throw new JsonIOException("JSON document was not fully consumed.");
    }
    return result;
  } finally {
    value.close();
  }
}
```

## 資料庫的建立與更新

資料庫的建立會在進入 **Fragment** 時經由 **ViewModelFactory** 產生：

```kotlin
class PlantListViewModelFactory(
    private val repository: PlantRepository
) : ViewModelProvider.NewInstanceFactory() {

    @Suppress("UNCHECKED_CAST")
    override fun <T : ViewModel> create(modelClass: Class<T>) = PlantListViewModel(repository) as T
}
```

有趣的是， **PlantListViewModelFactory** 是由 **DefaultViewModelProvider** 用注入的方式創建：

```kotlin
interface ViewModelFactoryProvider {
    fun providePlantListViewModelFactory(context: Context): PlantListViewModelFactory
}


private object DefaultViewModelProvider: ViewModelFactoryProvider {
    private fun getPlantRepository(context: Context): PlantRepository {
        return PlantRepository.getInstance(
            plantDao(context),
            plantService()
        )
    }

    private fun plantService() = NetworkService()

    private fun plantDao(context: Context) =
        AppDatabase.getInstance(context.applicationContext).plantDao()

    override fun providePlantListViewModelFactory(context: Context): PlantListViewModelFactory {
        val repository = getPlantRepository(context)
        return PlantListViewModelFactory(repository)
    }
}
```

當 **PlantListViewModel** 被創建時，他會調用 `init` 並通過 `clearGrowZoneNumber` 調用 repository 的 `tryUpdateRecentPlantsCache`：

```kotlin
init {
    // When creating a new ViewModel, clear the grow zone and perform any related udpates
    clearGrowZoneNumber()
}

fun clearGrowZoneNumber() {
    growZone.value = NoGrowZone

    // initial code version, will move during flow rewrite
    launchDataLoad { plantRepository.tryUpdateRecentPlantsCache() }
}

private fun launchDataLoad(block: suspend () -> Unit): Job {

    return viewModelScope.launch {
        try {
            _spinner.value = true
            block()
        } catch (error: Throwable) {
            _snackbar.value = error.message
        } finally {
            _spinner.value = false
        }
    }
}

/*
    viewModelScope 會創建以下 CoroutineScope，
    因此會確保在 Main Thread 執行

    CloseableCoroutineScope(SupervisorJob() + Dispatchers.Main.immediate)

    配合 SupervisorJob，我們不用擔心如果這些 Jobs 失敗時會導致全部 Job 被取消。
    我們也可以客製化 policy for handling failures of its children。

    對 ChildHandleNode 而言，當自己被 Cancel 時，他會試著取消 parentJob ：

    override fun childCancelled(cause: Throwable): Boolean = job.childCancelled(cause)

    但 SupervisorJob 則會回傳 false。
*/
```

當我們調用 `tryUpdateRecentPlantsCache` 時，我們會進行以下行為：

```kotlin
suspend fun tryUpdateRecentPlantsCache() {
    // 1. 得到 true
    if (shouldUpdatePlantsCache())
        // 2. 進行 fetch
        fetchRecentPlants()
}

private suspend fun fetchRecentPlants() {
    val plants = plantService.allPlants()
    plantDao.insertAll(plants)
}

// NetworkService.kt
suspend fun allPlants(): List<Plant> = withContext(Dispatchers.Default) {
    delay(1500)
    val result = sunflowerService.getAllPlants()
    result.shuffled()
}

```

### 從線上取得資料

由於 `allPlants` 會在 **Coroutine** 中執行，因此他會通過 **DispatchedTask** 的 `run` 然後 `invokeSuspend` 進行。

一旦進行，`java.lang.reflect` 的 **Proxy** 會調用 `invoke` ：

```java
// Android-added: Helper method invoke(Proxy, Method, Object[]) for ART native code.
private static Object invoke(Proxy proxy, Method method, Object[] args) throws Throwable {
    InvocationHandler h = proxy.h;
    return h.invoke(proxy, method, args);
}
```

這個方法主要是用來尋找 `method` 的實作。 此時的 `method` 是 `getAllPlants`。

另外，有趣的是，此時 `proxy` 的架構會是如下：

<center>
<img src = "/images/posts/jekyll/codelab/advanced-kotlin-coroutines/retrofit_proxy_invoke.png"/>
</center>

因此，當調用 `h.invoke(...)` 我們會進入 **Retrofit** 的 **InvocationHandler** ：

```kotlin
new InvocationHandler() {
  private final Platform platform = Platform.get();
  private final Object[] emptyArgs = new Object[0];

  @Override
  public @Nullable Object invoke(Object proxy, Method method, @Nullable Object[] args)
      throws Throwable {
    // If the method is a method from Object then defer to normal invocation.

    // 1. 若調用這方法的 類別 是 物件，那就直接調用方法
    if (method.getDeclaringClass() == Object.class) {
      return method.invoke(this, args);
    }

    // 2. 若是 interface (如同 SunflowerService)，那就會進入這裡
    args = args != null ? args : emptyArgs;
    return platform.isDefaultMethod(method)
        ? platform.invokeDefaultMethod(method, service, proxy, args)
        : loadServiceMethod(method).invoke(args);
  }
});
```

從以上方法會讓我們得到 `HttpServiceMethod$SuspendForBody` 的 **ServiceMethod<?>** 類別

```kotlin
// retrofit2
abstract class HttpServiceMethod<ResponseT, ReturnT> extends ServiceMethod<ReturnT>

// 其中的類別
static final class SuspendForBody<ResponseT> extends HttpServiceMethod<ResponseT, Object>
```

並調用他的 `invoke(args)`。

此時的 `args` 如下：

<center>
<img src = "/images/posts/jekyll/codelab/advanced-kotlin-coroutines/suspendForBody-invoke-arg-all-plants.png"/>
</center>

<br>

而 **SuspendForBody** 的 `invoke` 實作是建立 **OkHttpCall**。 裡面包含了 **RequestFactory**、 **OkHttpClient** 、 **GsonResponseBodyConverter** 與 `args`：

```kotlin
@Override
final @Nullable ReturnT invoke(Object[] args) {
  Call<ResponseT> call = new OkHttpCall<>(requestFactory, args, callFactory, responseConverter);
  return adapt(call, args);
}

@Override
protected Object adapt(Call<ResponseT> call, Object[] args) {

  /*
    1. callAdapter 是一個 DefaultCallAdpaterFactory

       當 adapt 被調用時，由於其中的 executor: Executor 為 null
       所以 `adpat(call)` 會直接得到傳入的 `call`

       也就是剛剛創建的 OkHttpCall
  */
  call = callAdapter.adapt(call);

  //noinspection unchecked Checked by reflection inside RequestFactory.

  /*
    2. 從 args 取得最後一個 Continuation

       以下是我們這次得到的：
       Continuation at com.example.android.advancedcoroutines.NetworkService$allPlants$2.invokeSuspend(NetworkService.kt:21)
  */
  Continuation<ResponseT> continuation = (Continuation<ResponseT>) args[args.length - 1];

  // Calls to OkHttp Call.enqueue() like those inside await and awaitNullable can sometimes
  // invoke the supplied callback with an exception before the invoking stack frame can return.
  // Coroutines will intercept the subsequent invocation of the Continuation and throw the
  // exception synchronously. A Java Proxy cannot throw checked exceptions without them being
  // declared on the interface method. To avoid the synchronous checked exception being wrapped
  // in an UndeclaredThrowableException, it is intercepted and supplied to a helper which will
  // force suspension to occur so that it can be instead delivered to the continuation to
  // bypass this restriction.
  try {
    /*
      3. isNullable 此次為 false
         因此會調用 KotlinExtensions.await

         此 call 為 OkHttpCall
         而 continuation 是 allPlants 的

         由於這個方法對網路連線挺重要的，所以我們在下面特別去看。
    */
    return isNullable
        ? KotlinExtensions.awaitNullable(call, continuation)
        : KotlinExtensions.await(call, continuation);
  } catch (Exception e) {
    return KotlinExtensions.suspendAndThrow(e, continuation);
  }
}
```

以下是 `KotlinExtensions.await` 的實作：

```kotlin
suspend fun <T : Any> Call<T>.await(): T {

  /*
      1. 建立 suspendCancellableCoroutine
         suspendCancellableCoroutine 會在 suspendCoroutineUninterceptedOrReturn
         中建立 CancellableContinuationImpl
         並帶入 block 中： block(cancel)
  */

  return suspendCancellableCoroutine { continuation ->
    continuation.invokeOnCancellation {
      cancel()
    }

    /*
      OkHttpCall 的 enqueue 會做以下事 ：
        1. 建立 okhttp3.Call ：
           okhttp3.Call call = callFactory.newCall(requestFactory.create(args));

           其中的 requestFactory.create 會建立 RequestBuilder
           並由此得到  okhttp3.Request

           最後會通過 newCall 將 Request 由 RealCall 他包起來。
           並用來做最後連線。

        2. 最後會調用 Recall call 的 enqueue
           並在其中建立一個新的 okhttp3.Callback 並帶入其中。

           這個 okhttp3.Callback 主要是處理：
            a. 進行 parseResponse
            b. 調用 callback.onResponse

           接著此 Callback 會被 AsyncCall: NamedRunnable 包起來，
           並 enqueue 到 client.dispatcher() 中。

           而 Dispatcher 會限制 maxRequest 為 64 以及 maxRequestPerHost == 5

           最後 AsyncCall 會被放入
           Deque<AsyncCall> readyAsyncCalls 中。 等著 promoteAndExecute 被調用並進行。

           所有的 AsyncCall 都會經由 ExecutorService 進行：
           asyncCall.executeOn(executorService());

           executorService = new ThreadPoolExecutor(0, Integer.MAX_VALUE, 60, TimeUnit.SECONDS,
                new SynchronousQueue<>(), Util.threadFactory("OkHttp Dispatcher", false));

          經由 Util.threadFactory 所創建的執行緒是
          Thread[OkHttp Dispatcher,5,main]
          [ 當然這與 withContext(Dispatchers.Default)所使用的執行緒並不相同 ]
    */
    enqueue(object : Callback<T> {
      override fun onResponse(call: Call<T>, response: Response<T>) {
        if (response.isSuccessful) {
          val body = response.body()
          if (body == null) {
            val invocation = call.request().tag(Invocation::class.java)!!
            val method = invocation.method()
            val e = KotlinNullPointerException("Response from " +
                method.declaringClass.name +
                '.' +
                method.name +
                " was null but response body type was declared as non-null")
            continuation.resumeWithException(e)
          } else {
            continuation.resume(body)
          }
        } else {
          continuation.resumeWithException(HttpException(response))
        }
      }

      override fun onFailure(call: Call<T>, t: Throwable) {
        continuation.resumeWithException(t)
      }
    })
  }
}
```

接下來，我們看看 `executeOn` 之後會發生什麼事吧。

```kotlin
void executeOn(ExecutorService executorService) {
    assert (!Thread.holdsLock(client.dispatcher()));
    boolean success = false;
    try {
        /*
          此時的 this 是 RealCall$AsyncCall。
          executorService 是 ThreadPoolExecutor
          而在調用 execute 後會進行以下行為：

          1. AsyncCall 會被 ThreadPoolExecutor 的 Worker 包裹起來。
             private final class Worker extends AbstractQueuedSynchronizer implements Runnable

             創建 Worker 的同時也會經由
             ThreadPoolExecutor.this.getThreadFactory().newThread(this);
             建立新的執行緒。

             而所創建的執行緒是：
             Thread[OkHttp Dispatcher,5,main]

             要注意的是， withContext (Dispatchers.Default)
             會讓我們得到 Thread[DefaultDispatcher-worker-1,5,main]

          2. 再來就是要將 Worker 加入
             HashSet<Worker> workers

             但這只會在 thread != null 的情況下發生。

             當我們要將 Worker 加入時，我們需要用到 ReentrantLock mainLock。

             ReentrantLock 與 synchronized 很類似。
             主要差異就是 ReentrantLock 需要手動進行 lock 與 unlock。

             也就是說，每次只會有一條執行緒進來修改 workers。

          3. 在將 Worker 加入後，就會調用此 Worker 的 thread.start()。
             ( 這裡沒找到做了什麼 )

        */
        executorService.execute(this);
        success = true;
    } catch (RejectedExecutionException e) {
        InterruptedIOException ioException = new InterruptedIOException("executor rejected");
        ioException.initCause(e);
        transmitter.noMoreExchanges(ioException);
        responseCallback.onFailure(RealCall.this, ioException);
    } finally {
        if (!success) {
          client.dispatcher().finished(this); // This call is no longer running!
        }
    }
}
```

到目前為止，我們只是將 `allPlants` 中的 `getAllPlants` 變成 **Runnable** 並經過一連串的包裝後讓 executorService 操作她。

接下來我們只能等待系統通知這個 Runnable 可以執行。也就是當 **CoroutineScheduler$Worker** 被調用 `run` 後的行為。

```kotlin
// Thread.class
/*
  1. 此時的 this == Thread[OkHttp Dispatcher,5,main]
     而 task 為 java.util.concurrent.ThreadPoolExecutor$Worker@23a48b01[State = 1, empty queue]
*/
public void run() {
    if (this.target != null) {
        this.target.run();
    }
}

// ThreadPoolExecutor.class

public void run() {
    ThreadPoolExecutor.this.runWorker(this);
}

/*
  2.
*/

final void runWorker(Worker w) {
    /*
      1. 取得 Thread[OkHttp Dispatcher,5,main]
    */
    Thread wt = Thread.currentThread();

    /*
      2. 取得之前得到的 RealCall$AsyncCall
    */
    Runnable task = w.firstTask;

    w.firstTask = null;
    w.unlock();
    boolean completedAbruptly = true;

    try {
        while(task != null || (task = this.getTask()) != null) {
            w.lock();
            if ((runStateAtLeast(this.ctl.get(), 536870912) || Thread.interrupted() && runStateAtLeast(this.ctl.get(), 536870912)) && !wt.isInterrupted()) {
                wt.interrupt();
            }

            try {
                this.beforeExecute(wt, task);

                try {

                    /*
                      3. 由於 task 是 AsyncCall :> NamedRunnable

                         所以當調用 `run` 時，會調用 NamedRunnable 的 `run`。
                         然後會「暫時更新」Thread 的名稱成 OkHttp https://raw.githubusercontent.com/...

                      4. 之後便會調用此 RealCall 的 execute()
                         這裡會做很多的事，所以得在下面分開講。

                    */
                    task.run();
                    this.afterExecute(task, (Throwable)null);
                } catch (Throwable var14) {
                    this.afterExecute(task, var14);
                    throw var14;
                }
            } finally {
                task = null;
                ++w.completedTasks;
                w.unlock();
            }
        }

        completedAbruptly = false;
    } finally {
        this.processWorkerExit(w, completedAbruptly);
    }

}
```

我們來看看 **RealCall** 的 `execute()` 吧：


```kotlin
@Override protected void execute() {
  boolean signalledCallback = false;
  transmitter.timeoutEnter();
  try {
    /*
      1. 這方法會做以下：
         a. 將多個 Interceptor，包括
            client.interceptors() [size == 0]、
            RetryAndFollowUpInterceptor 、
            BridgeInterceptor 、
            CacheInterceptor 、
            ConnectInterceptor 與
            client.networkInterceptors() 、
            CallServerInterceptor
            加在 List<Interceptor> interceptors 中。

        b. 創建 Interceptor.Chain ：
           RealInterceptorChain(interceptors, transmitter, null, 0,
                                originalRequest, this, client.connectTimeoutMillis(),
                                client.readTimeoutMillis(), client.writeTimeoutMillis());
        c. 通過 chain.proceed(originalRequest)
           [Request{method=GET, url=https://raw.githubusercontent.com/googlecodelabs/kotlin-coroutines/master/advanced-coroutines-codelab/sunflower/src/main/assets/plants.json, tags={class retrofit2.Invocation=com.example.android.advancedcoroutines.SunflowerService.getAllPlants() []}}]
           取得 Response 。

        d. 建立 RealInterceptorChain next
           此時 index == index + 1

           然後從 interceptors.get(index) // index == 0
           中取得 RetryAndFollowUpInterceptor 並調用：
           Response response = interceptor.intercept(next);

        e. 在 intercept(next) 時，會先取得 transmitter 並調用
           transmitter.prepareToConnect(request);

           並建立 ExchangeFinder 來協助連線。
           其中的 RouteSelector routeSelector 會紀錄當下可進行連線的路徑。
           他會在創建 ExchangeFinder 時建立：
           this.routeSelector = new RouteSelector( address, connectionPool.routeDatabase, call, eventListener);

        f. 回到 intercept(next) 後， 便會調用
           response = realChain.proceed(request, transmitter, null);

           此時會再次將 index + 1，創建下一個 RealInterceptorChain
           然後取得下一個 Interceptor ： BridgeInterceptor

           並調用 Response response = interceptor.intercept(next)
           建立 Request.Builder，將 header, content ... 設定。

           然後通過
           Response networkResponse = chain.proceed(requestBuilder.build());

           來建立 Request(this) 並傳入 chain.proceed 。

        g. 這次進入 proceed 會取得 CacheInterceptor
           進行 Response response = interceptor.intercept(next)

           經由
           CacheStrategy strategy = new CacheStrategy.Factory(now, chain.request(), cacheCandidate).get();
           Request networkRequest = strategy.networkRequest;
           Response cacheResponse = strategy.cacheResponse;

           最後調用
           Response networkResponse = chain.proceed(networkRequest);

           networkRequest = Request{method=GET, url=https://raw.githubusercontent.com/googlecodelabs/kotlin-coroutines/master/advanced-coroutines-codelab/sunflower/src/main/assets/plants.json, tags={class retrofit2.Invocation=com.example.android.advancedcoroutines.SunflowerService.getAllPlants() []}}

        h. 取得 ConnectInterceptor 並進入 intercept(Chain chain)

           接著從 (RealInterceptorChain) chain
           取得 request 與 transmitter

           再從 transmitter 建立 Exchange
           Exchange exchange = transmitter.newExchange(chain, doExtensiveHealthChecks);

           newExchange 主要功能是找尋 healthy connection。
           這會通過 ExchangeFinder 看看 pool 中是否有連線可用。

           這是由 ExchangeFinder 中的 RouteSelector 和 RouteSelection 取得最後要使用的 connection ：
           result = new RealConnection(connectionPool, selectedRoute);
           connectingConnection = result;

           這個是可以用來對之後要進行的 handshake 進行非同步取消。

               通過以下可以進行 TCP/TLS handshake
               result.connect(connectTimeout, readTimeout, writeTimeout, pingIntervalMillis,
                              connectionRetryEnabled, call, eventListener);

               這會進行 Protocol 的設定並通過 RealConnection 中的 startHttp2
               建立 Http2Connection
               http2Connection = new Http2Connection.Builder(true)
                  .socket(socket, route.address().url().host(), source, sink)
                  .listener(this)
                  .pingIntervalMillis(pingIntervalMillis)
                  .build();
              http2Connection.start()

              start 會使用
              final ReaderRunnable readerRunnable;
              創建執行緒並進行 start。

          並記錄起來
          connectionPool.routeDatabase.connected(result.route());

          result 也會儲存在 connectionPool 中。

              儲存之前會先調用
              executor.execute(cleanupRunnable)
              [executor : ThreadPoolExecutor]
              來將 RealConnectionPool 中 以超時 或 沒有在運作的 connection 清理。
              這行為也會在新的執行緒執行。

          再來就是通過
          transmitter.acquireConnectionNoEvents(result)
          在 result: RealConnection 中的 transmitters 新增一個 callStackTrace 的 Transmitter 。

          然後就會將 result 從 ExchangeFinder$findConnection
          回傳至 ExchangeFinder$findHealthyConnection 中。

          並再次回傳至 ExchangeFinder 的 ExchangeCodec find 方法。
          並回傳
          [this : RealConnection]
          Http2ExchangeCodec(client, this, chain, http2Connection);

          終於，我們回到 newExchange 的方法
          並建立
          new Exchange(this, call, eventListener, exchangeFinder, codec);
          且回傳至 此次的 intercept 方法。

          最後便會調用
          realChain.proceed(request, transmitter, exchange);

        i. 取得 CallServerInterceptor 再次調用 intercept。
           這方法會做以下的事：

           - 通過 codec 取得 Response.Builder
             Response.Builder result = codec.readResponseHeaders(expectContinue);
           - 此 Response.Builder 會用來取得 Response
             Response response = responseBuilder
                // 設定 Builder 的 request
                .request(request)

                // handshake = Handshake{tlsVersion=TLS_1_2 cipherSuite=TLS_ECDHE_RSA_WITH_AES_128_GCM_SHA256 peerCertificates=[CN=*.github.io, O="GitHub, Inc.", L=San Francisco, ST=California, C=US, CN=DigiCert TLS RSA SHA256 2020 CA1, O=DigiCert Inc, C=US] localCertificates=[]}
                .handshake(exchange.connection().handshake())

                .sentRequestAtMillis(sentRequestMillis)
                .receivedResponseAtMillis(System.currentTimeMillis())

                .build();
             此時的 response 的 body 是 null ，但 status code 卻會是 200。
           - 接下來便是針對 status code 的行為：
             -> 若是 100 - continue : 便會再次取得 response 。
             -> 若是 101 並且是 WebSocket，
                那就需要進行 upgrade
                response = response.newBuilder()
                                   .body(Util.EMPTY_RESPONSE)
                                   .build()
               否則便進行讀取並更新 body : RealResponseBody
               response = response.newBuilder()
                                  .body(exchange.openResponseBody(response))
                                  .build()
          - response 會回傳到 CacheInterceptor 並創建 Response
            Response response = networkResponse.newBuilder()
                .cacheResponse(stripBody(cacheResponse))
                .networkResponse(stripBody(networkResponse))
                .build();
            這行為主要是更新 Response 的 cacheResponse 與 networkResponse。

          - response 又繼續回傳至 BridgeInterceptor 並建立 Response.Builder
            Response.Builder responseBuilder =
                networkResponse.newBuilder()
                               .request(userRequest);

            然後會更新 body
            responseBuilder.body(new RealResponseBody(contentType, -1L, Okio.buffer(responseBody)));

            最後回傳 responseBuilder.build()

          - 終於我們又回到 RealCall$AsyncCall 的 execute。
    */
    Response response = getResponseWithInterceptorChain();
    signalledCallback = true;

    /*
      2. 接下來並調用 OkHttpCall 的 onResponse
         responseCallback.onResponse(RealCall.this, response);

         這方法會間接調用 OkHttp 的 parseResponse。
         若 code < 200 || code >= 300 會回傳 Response.error
         而 code == 204 || code == 205 則會回傳 Response.success(body: null, rawResponse)

         再來就是建立 ExceptionCatchingResponseBody catchingBody
         然後經由 GsonResponseBodyConverter 將 responseBody 讀取出來
         T body = responseConverter.convert(catchingBody);

         終於，這個 body 就是我們想要的結果

         然後 RealResponseBody 便會調用 close() 將 BufferedSource 關掉。

         最後便回傳
         Response.success(body, rawResponse);

         然後在 OkHttpCall 中的 enqueue 的 call.enqueue 中的
         callback.onResponse(OkHttpCall.this, response);

         再來就是調用 fun <T : Any> Call<T>.await(): T
         的 enqueue 物件：
         enqueue(object : Callback<T> {...})

         並調用 continuation.resume(body)
         將 body 以此形式
         Result.success(outcome)
         傳回去。

    */
    responseCallback.onResponse(RealCall.this, response);
  } catch (IOException e) {
    if (signalledCallback) {
      // Do not signal the callback twice!
      Platform.get().log(INFO, "Callback failure for " + toLoggableString(), e);
    } else {
      responseCallback.onFailure(RealCall.this, e);
    }
  } catch (Throwable t) {
    cancel();
    if (!signalledCallback) {
      IOException canceledException = new IOException("canceled due to " + t);
      canceledException.addSuppressed(t);
      responseCallback.onFailure(RealCall.this, canceledException);
    }
    throw t;
  } finally {
    client.dispatcher().finished(this);
  }
}
```

### 將線上資料存入 Room

```kotlin
private suspend fun fetchRecentPlants() {
    val plants = plantService.allPlants()
    plantDao.insertAll(plants)
}
```
`plantDao` 所調用的方法都會在 **PlantDao** 介面被定義：

```kotlin
@Dao
interface PlantDao {
    @Query("SELECT * FROM plants ORDER BY name")
    fun getPlants(): LiveData<List<Plant>>

    @Query("SELECT * FROM plants WHERE growZoneNumber = :growZoneNumber ORDER BY name")
    fun getPlantsWithGrowZoneNumber(growZoneNumber: Int): LiveData<List<Plant>>

    @Insert(onConflict = OnConflictStrategy.REPLACE)
    suspend fun insertAll(plants: List<Plant>)
}
```

如果我們看編輯器產生的實作，我們可以看到 `insertAll` 的實作很直接。 而且還會傳入 `continuation` 來進行 `CoroutinesRoom.execute` ：

```java
@Override
public Object insertAll(final List<Plant> plants, final Continuation<? super Unit> continuation) {
  return CoroutinesRoom.execute(__db, true, new Callable<Unit>() {
    @Override
    public Unit call() throws Exception {
      __db.beginTransaction();
      try {
        __insertionAdapterOfPlant.insert(plants);
        __db.setTransactionSuccessful();
        return Unit.INSTANCE;
      } finally {
        __db.endTransaction();
      }
    }
  }, continuation);
}
```

至於 `getPlants`，實作蠻長的，所以我們就看看 `getPlants` 是如何取得 **LiveData** 好了：

```java
@Override
  public LiveData<List<Plant>> getPlants() {
    final String _sql = "SELECT * FROM plants ORDER BY name";
    final RoomSQLiteQuery _statement = RoomSQLiteQuery.acquire(_sql, 0);
    return __db.getInvalidationTracker().createLiveData(new String[]{"plants"}, false, new Callable<List<Plant>>() {
      // 取得資料時會調用 call
      @Override
      public List<Plant> call() throws Exception { ... }

      @Override
      protected void finalize() { ... }
    }
  }
}
```

#### 使用 Glide 顯示圖像


```kotlin
/*
  xml 會通過
  app:imageFromUrl="@{plant.imageUrl}"
  調用此方法
*/

@BindingAdapter("imageFromUrl")
fun bindImageFromUrl(view: ImageView, imageUrl: String?) {
    if (!imageUrl.isNullOrEmpty()) {
        Glide.with(view.context)
                .load(imageUrl)
                .transition(DrawableTransitionOptions.withCrossFade())
                .into(view)
    }
}
```

我們來一段一段地看這些碼是如何進行圖像的取得與顯示。

```kotlin
Glide.with(view.context) // 取得 RequestManager
```

通過 `Glide.with` 我們會創建 **Glide**，並將以下變數一同設定：

```java
GlideExecutor sourceExecutor
GlideExecutor diskCacheExecutor
GlideExecutor animationExecutor

MemorySizeCalculator memorySizeCalculator
DefaultConnectivityMonitorFactory connectivityMonitorFactory

BitmapPool bitmapPool  // 可以是 LruBitmapPool 或 BitmapPoolAdapter
LruArrayPool arrayPool
LruResourceCache memoryCache

// 預設為 250 MB
InternalCacheDiskCacheFactory diskCacheFactory

Engine engine
```

另外，**Glide** 也會成為 **ApplicationContext** 的 **ComponentCallbacks** 來處理記憶體不足的狀況。

至於 **Glide** 的暫存預設是放在內存的 `image_manager_disk_cache` 中，而預設大小是 **250 MB**。

建立好 **RequestManager** 後，我們就要進行設定了。

首先我們使用 `load` 定義了圖像的來源 ：

```kotlin
Glide.with(view.context)
        .load(imageUrl)
```

`RequestManager.load` 方法其實很簡單，他只是設定 **RequestBuilder** 的 `model` 變數。

<br>

<center>
<img src = "/images/posts/jekyll/codelab/advanced-kotlin-coroutines/glide_load_url.svg"/>
</center>

<br>

這裡的 `imageUrl` 並不只是用於資料的索取，還會被用來暫存時的 `key`。 這之後會提到。

接下來的 `RequestBuilder.transition` 也有類似的作用：

```java
@NonNull
@CheckResult
public RequestBuilder<TranscodeType> transition(
    @NonNull TransitionOptions<?, ? super TranscodeType> transitionOptions) {
  if (isAutoCloneEnabled()) {
    return clone().transition(transitionOptions);
  }
  this.transitionOptions = Preconditions.checkNotNull(transitionOptions);
  isDefaultTransitionOptionsSet = false;
  return selfOrThrowIfLocked();
}
```

在 `into` 方法之前都是在對 **RequestBuilder** 的設定。

```kotlin
Glide.with(view.context)
        .load(imageUrl)
        .transition(DrawableTransitionOptions.withCrossFade())
        .into(view)
```

一旦進入 `into` 時，我們就會開始建立圖像與 Thumbnail 的 **Request**。

```java
Request request = buildRequest(
    target,             // 這是通過 glideContext.buildImageViewTarget(view, transcodeClass),
                        // 建立的 ViewTarget

    targetListener,     // 此時為 null
    options,            // 這是 this : RequestBuilder
    callbackExecutor);  // 這是 Executors.mainThreadExecutor()
```

之所以需要 `target` 參數是因為 **ViewTarget** 可以幫我們紀錄 **Request**。

而這個 `request` 最後會由 **SingleRequest** 創建：

```java
SingleRequest<>(
      context,
      glideContext,
      requestLock,
      model,
      transcodeClass,
      requestOptions,
      overrideWidth,
      overrideHeight,
      priority,
      target,
      targetListener,
      requestListeners,
      requestCoordinator,
      engine,
      animationFactory,
      callbackExecutor);
}
```





```java
// ResourceCacheGenerator.java

@Override
public boolean startNext() {
  //...
  /*
    1. 創建 key 時， sourceId 便是我們的 URL
  */
  currentKey =
        new ResourceCacheKey( // NOPMD AvoidInstantiatingObjectsInLoops
            helper.getArrayPool(),
            sourceId,
            helper.getSignature(),
            helper.getWidth(),
            helper.getHeight(),
            transformation,
            resourceClass,
            helper.getOptions());

    /*
      2. 使用 currentKey 從 cache 中取得目標檔案
    */
    cacheFile = helper.getDiskCache().get(currentKey);
}
```

```java



```




當我們調用 `helper.getDiskCache().get` 時，以下會發生：

```java
/*
  1. 使用 SafeKeyGenerator safeKeyGenerator
     並調用 getSafeKey
*/
String safeKey = safeKeyGenerator.getSafeKey(key);

/*
  2. 通過 calculateHexStringDigest 來取得 key
*/
String safeKey = calculateHexStringDigest(key);

/*
  3. 取得 SafeKeyGenerator.PoolableDigestContainer
*/
PoolableDigestContainer container = Preconditions.checkNotNull(digestPool.acquire())

/*
  4.
*/
key.updateDiskCacheKey(container.messageDigest);


```



 **ResourceCacheKey**


但是我們還可以使用 `signature(Key)` 來修改 key 最後的值。

```kotlin

```

`signature` 的作用會在 `dispatchOnPreDraw` 時發生。





### Cache 起來

這段源碼主要是要將 Plant 的 id 存起來。

```kotlin
private var plantsListSortOrderCache =
    CacheOnSuccess(onErrorFallback = { listOf<String>() }) {
        plantService.customPlantSortOrder()
    }

suspend fun customPlantSortOrder(): List<String> = withContext(Dispatchers.Default) {

    // 1. 從線上取得資料 List<Plant>
    val result = sunflowerService.getCustomPlantSortOrder()

    // 2. 回傳 List<String>
    result.map { plant -> plant.plantId }
}
```

但是重點是在 **CacheOnSuccess** 的功能是在多線程的情況下建立 **Deferred** 並調用 `block`。

```kotlin
// 由於 customPlantSortOrder 回傳的是 List<String>
// 所以 T : List<String>

class CacheOnSuccess<T: Any>(
    private val onErrorFallback: (suspend () -> T)? = null,
    private val block: suspend () -> T
) {
    private val mutex =  Mutex()

    // 設為 Volatile 並配合 mutex
    // 來確保 thread-safe
    // 因為 Volatile 適合一個寫一個讀
    // 但若需要進行 non-atomic 讀寫，那就要使用 lock 來確保 atomic operation。
    @Volatile
    private var deferred: Deferred<T>? = null

    suspend fun getOrAwait(): T {

        /*
          supervisorScope 會取得外面的 Continuation
          並在其中增加 SupervisorCoroutine
          最後會在此 Coroutine 中執行 block

          public suspend fun <R> supervisorScope(block: suspend CoroutineScope.() -> R): R
        */
        return supervisorScope {
            // This function is thread-safe _iff_ deferred is @Volatile and all reads and writes
            // hold the mutex.

            // 確保只有一個 Coroutine 可調用 block
            val currentDeferred =
            /*
              withLock 會將 owner lock 起來
              並進行 action() 。

              但預設 owner == null

              但 mutex.withLock 本身就會確保
              block 只能有單一執行緒調用
              https://kotlinlang.org/docs/shared-mutable-state-and-concurrency.html#actors
             */
             mutex.withLock {

                // 如果有 deferred 便將其回傳
                deferred?.let { return@withLock it }

                // 若無 deferred 便使用 async 來創建 deferred
                async {
                    // Note: mutex is not held in this async block
                    block()
                }.also {
                    // Note: mutex is held here
                    deferred = it
                }
            }

            // await the result, with our custom error handling
            // 這會調用到下面的延伸方法
            // 而這方法會回傳給
            currentDeferred.safeAwait()
        }
    }

    //
    private suspend fun Deferred<T>.safeAwait(): T {
        try {
            // Note: this call to await will always throw if this coroutine is cancelled

            /*
              對於 Deferred， await 的實作為：
              override suspend fun await(): T = awaitInternal() as T

              而 awaitInternal 會使用 while(true) 進行以下行為:

              1. 如果 state 已經結束，最後會將 state : Any? 進行 unbox ：
                 return state.unboxState()
              2. 若尚未結束便會進行 state 的更新。

              在更新完 state 之後便會調用 awaitSuspend() 。
              並在 suspendCoroutineUninterceptedOrReturn 中建立
              AwaitContinuation : CancellableContinuationImpl

              之後會在此 DispatchTask 中設定 DisposeHandler。
              如此一來，若有 exception 便會由 continuation.resumeWithException 傳回去。

              最後會調用 AwaitContinuation 的 getResult() 來取得最後結果。
            */
            return await()
        } catch (throwable: Throwable) {
            // If deferred still points to `this` instance of Deferred, clear it because we don't
            // want to cache errors

            // 若有錯誤發生，便將 deferred 移除
            mutex.withLock {
                if (deferred == this) {
                    deferred = null
                }
            }

            // never consume cancellation
            if (throwable is CancellationException) {
                throw throwable
            }

            // return fallback if provided
            onErrorFallback?.let { fallback -> return fallback() }

            // if we get here the error fallback didn't provide a fallback result, so throw the
            // exception to the caller
            throw throwable
        }
    }
}
```

範例使用方法：

```kotlin
/**
 * Cache the first non-error result from an async computation passed as [block].
 *
 * Usage:
*/

  val cachedSuccess: CacheOnSuccess<Int> = CacheOnSuccess(onErrorFallback = { 3 }) {
      delay(1_000) // compute value using coroutines
      5
   }

  // get the result from the cache,
  // calling [block], or fallback on exception
  cachedSuccess.getOrAwait()
```

基本上這就是核心運作了。

接下來，我們更新一下 `getPlants` 與 `getPlantsWithGrowZone`。

# 新增排序行為

在 PlantRepository 中，我們先定義 `applySort` 方法：

```kotlin
private fun List<Plant>.applySort(customSortOrder: List<String>): List<Plant> {

    /*
        這法會使用 customSortOrder 將每個 plant 都創建一個 ComparablePair
        這類別會被用在 sortedBy 中 的 compareBy

        public inline fun <T, R : Comparable<R>> Iterable<T>.sortedBy(crossinline selector: (T) -> R?): List<T> {
            return sortedWith(compareBy(selector))
        }

        Comparator { a, b -> compareValuesBy(a, b, selector) }
    */
    return sortedBy { plant ->
        val positionForItem = customSortOrder.indexOf(plant.plantId).let { order ->
            if (order > -1) order else Int.MAX_VALUE
        }

        // ComparablePair 會記錄 位置 與 名稱
        // 最後 sortedBy 會用這個來進行排序
        ComparablePair(positionForItem, plant.name)
    }
}
```

接下來我們需要做兩件事：

1. 從 Room 取得 Plants ： **LiveData<List\<Plant>>**
2. 從 Network 取得已排序好的 Plant ID ： **List\<String>**

```kotlin
// 1
val plantsLiveData = plantDao.getPlants()
// 2
val customSortOrder = plantsListSortOrderCache.getOrAwait()
```

再來就是按照 Plant ID 的排序將 Plants 排序好。

但這要如何做到呢？
其實我們可以使用 `map` 方法：

```kotlin
plantsLiveData.map {
    plantList -> plantList.applySort(customSortOrder)
}
```

<u>接下來要如何將新的值還給 `plants` 呢？</u>

由於 `plantsListSortOrderCache.getOrAwait` 是一個 suspend function，因此必須由另外一個 suspend function 調動。

另外，`plantsListSortOrderCache.getOrAwait` 需要在 `plantsLiveData` 更新後再執行，所以我們可以用 **MediatorLiveData** ：

```kotlin
val plantsLiveData = plantDao.getPlants()

val plants = MediatorLiveData<List<Plant>>().apply {
    addSource(plantsLiveData) {
        val context = EmptyCoroutineContext
        val supervisorJob = SupervisorJob(context[Job])
        val scope = CoroutineScope(Dispatchers.Main.immediate + context + supervisorJob)
        scope.launch {
            value = it.applySort(withContext(Dispatchers.IO) {
                plantsListSortOrderCache.getOrAwait()
            })
        }
    }
}
```

雖然這個方法可以順利執行，但我們還可以將他簡化。 只要使用依舊是實驗 API 的 `livedata` 方法：

```kotlin
@OptIn(ExperimentalTypeInference::class)
public fun <T> liveData(
    context: CoroutineContext = EmptyCoroutineContext,
    // 預設 5 秒
    timeoutInMs: Long = DEFAULT_TIMEOUT,
    @BuilderInference block: suspend LiveDataScope<T>.() -> Unit
): LiveData<T> = CoroutineLiveData(context, timeoutInMs, block)
```

其中的 `@BuilderInference` 讓我們在 block 中定義 `T` 的類型。

[BuilderInference 的使用](https://juejin.cn/post/7209625823582027832)

而 **CoroutineLiveData** 其實是 **MediatorLiveData** ：
當中會使用 `blockRunner` 來存放 `block`，並在 **LiveData** 更新時被啟動。


```kotlin
internal class CoroutineLiveData<T>(
    context: CoroutineContext = EmptyCoroutineContext,
    timeoutInMs: Long = DEFAULT_TIMEOUT,
    block: Block<T>
) : MediatorLiveData<T>() {

    private var blockRunner: BlockRunner<T>?
    private var emittedSource: EmittedSource? = null

    init {
        val supervisorJob = SupervisorJob(context[Job])
        val scope = CoroutineScope(Dispatchers.Main.immediate + context + supervisorJob)

        blockRunner = BlockRunner(
            liveData = this,
            block = block,
            timeoutInMs = timeoutInMs,
            scope = scope
        ) {
            blockRunner = null
        }
    }

    internal suspend fun emitSource(source: LiveData<T>): DisposableHandle {
        clearSource()
        val newSource = addDisposableSource(source)
        emittedSource = newSource
        return newSource
    }

    internal suspend fun clearSource() {
        emittedSource?.disposeNow()
        emittedSource = null
    }

    // 當 LiveData 更新時便會調用
    override fun onActive() {
        super.onActive()
        blockRunner?.maybeRun()
    }

    override fun onInactive() {
        super.onInactive()
        blockRunner?.cancel()
    }
}
```

至於 **BlockRunner** 與 **LiveData** 的互動便要看 **BlockRunner** 的源碼：

當 `blockRunner` 的 `maybeRun` 被調用時，他會經在 `scope.launch` 中建立 **LiveDataScopeImpl** 並傳入 `block`，也就是 **CoroutineLiveData** 中的 `block`。

```kotlin
// Handles running a block at most once to completion.

internal class BlockRunner<T>(
    private val liveData: CoroutineLiveData<T>,
    private val block: Block<T>,
    private val timeoutInMs: Long,
    private val scope: CoroutineScope,
    private val onDone: () -> Unit
) {
    // currently running block job.
    private var runningJob: Job? = null

    // cancelation job created in cancel.
    private var cancellationJob: Job? = null

    @MainThread
    fun maybeRun() {
        cancellationJob?.cancel()
        cancellationJob = null
        if (runningJob != null) {
            return
        }
        runningJob = scope.launch {

            // 建立 LiveDataScopeImpl
            val liveDataScope = LiveDataScopeImpl(liveData, coroutineContext)
            block(liveDataScope)
            onDone()
        }
    }

    @MainThread
    fun cancel() {
        if (cancellationJob != null) {
            error("Cancel call cannot happen without a maybeRun")
        }
        cancellationJob = scope.launch(Dispatchers.Main.immediate) {
            delay(timeoutInMs)
            if (!liveData.hasActiveObservers()) {
                runningJob?.cancel()
                runningJob = null
            }
        }
    }
}
```

通過 `liveData` 我們可以將

```kotlin
val plantsLiveData = plantDao.getPlants()

val plants = MediatorLiveData<List<Plant>>().apply {
    addSource(plantsLiveData) {
        val context = EmptyCoroutineContext
        val supervisorJob = SupervisorJob(context[Job])
        val scope = CoroutineScope(Dispatchers.Main.immediate + context + supervisorJob)
        scope.launch {
            value = it.applySort(withContext(Dispatchers.IO) {
                plantsListSortOrderCache.getOrAwait()
            })
        }
    }
}

// ------ 改成
val plants: LiveData<List<Plant>> = liveData<List<Plant>> {
    val plantsLiveData = plantDao.getPlants()
    val customSortOrder = plantsListSortOrderCache.getOrAwait()
    plantsLiveData.value?.applySort(customSortOrder)
}
```

但很可惜以上的方法並不會有任何作用，畢盡在 **BlockRunner** 中，我們必須要通知 **LiveDataScopeImpl** 才能更新。

所以上面的編碼可以改為：

```kotlin
val plants: LiveData<List<Plant>> = liveData<List<Plant>> {
    val plantsLiveData = plantDao.getPlants()
    val customSortOrder = plantsListSortOrderCache.getOrAwait()
    emitSource(plantsLiveData.map { plantList ->
       plantList.applySort(customSortOrder)
    })
}
```

除了 `plants` 外，`plantsByGrowZone` 也可以有相同的實作：

```kotlin
fun getPlantsWithGrowZone(growZone: GrowZone) =
        plantDao.getPlantsWithGrowZoneNumber(growZone.number)

// 新增排序行為
// 這方法會得到 LiveData<List<Plant>>
fun getPlantsWithGrowZone(growZone: GrowZone) = liveData {
    // 1. 從 Room 取得 LiveData<List<Plant>>
    val plantsGrowZoneLiveData = plantDao.getPlantsWithGrowZoneNumber(growZone.number)

    // 2. 從 Network 取得 List<Plant.id>
    val customSortOrder = plantsListSortOrderCache.getOrAwait()

    // 3. 用 customSortOrder 來排序並傳遞出去
    emitSource(plantsGrowZoneLiveData.map { plantList ->
        plantList.applySort(customSortOrder)
    })
}
```

## emitSource 的流程

如果你試著跑以上的程式，並停在 `emitSource` 時，泥會發現 `plantsLiveData` 或 `customSortOrder` 的值都會是 `null`。

當然，這很正常，因為這兩者皆為 suspend 方法。

那到底 `emitSource` 是如何在 `plantsLiveData` 更新後調用 `plantList.applySort`呢？

這是因為當我們調用 `emitSource` 時，我們會做以下的事：

```kotlin
// CoroutineLiveData :> MediatorLiveData
internal suspend fun emitSource(source: LiveData<T>): DisposableHandle {
    clearSource()

    /*
          將 source 記錄在 mSources 中，
          並將 source 與 this 包裹成 EmittedSource 再回傳。

          此時的 source 是：

          plantsLiveData.map { plantList ->
             plantList.applySort(customSortOrder)
          }
    */
    val newSource = addDisposableSource(source)
    emittedSource = newSource
    return newSource
}
```

當中最重要的是 `addDisposableSource`。
通過這方法， `source` 會被記錄在 `mSources` ：

```kotlin
private SafeIterableMap<LiveData<?>, Source<?>> mSources = new SafeIterableMap<>();
```
當 **Lifecycle.State** 更新時，以下的事會發生：

<center>
<img src = "/images/posts/jekyll/codelab/advanced-kotlin-coroutines/emitSource_onActive.svg"/>
</center>

<br>

當 `onActive` 被調用後，**MediatorLiveData** 會進行監控：

```kotlin
source.getValue().plug();

void plug() {
    mLiveData.observeForever(this);
}
```

這時就等 `mLiveData` 進行更新。
一旦更新，他就會調用 `setValue` 並調用 **Observer** 的 `onChange` 來通知所有的觀察者們。


## 使用 emit

剛剛我們使用了 `emitSource (LiveData)` 但其實 **LiveDataScopeImpl** 還有一個方法： `emit(T)` ：


```kotlin
override suspend fun emit(value: T) = withContext(coroutineContext) {
    target.clearSource()
    target.value = value
}
```

如果我們想要使用 `emit`，那我們必須要將 `plantsLiveData` 的 `value` 取出，並傳入 `emit`。


```kotlin
val plants: LiveData<List<Plant>> = liveData<List<Plant>> {
    val plantsLiveData = plantDao.getPlants()
    val customSortOrder = plantsListSortOrderCache.getOrAwait()
    emitSource(plantsLiveData.map { plantList ->
       plantList.applySort(customSortOrder)
    })
}

// 首次嘗試
val plants: LiveData<List<Plant>> = liveData<List<Plant>> {
    val plantsLiveData = plantDao.getPlants()
    val customSortOrder = plantsListSortOrderCache.getOrAwait()
    emit(plantsLiveData.map { plantList ->
       plantList.applySort(customSortOrder)
    }.value ? listOf())
}
```

如果我們使用上述的方法，我們不會得到任何結果。 這是因為 `emit` 並沒有對 `plantsLiveData` 進行監控。 因此，當 `plantsLiveData` 更新後，`emit` 並不會再次被調用。

所以我們需要將 `plantsLiveData` 放在外面，如此一來， 一旦 `plantsLiveData` 更新後內部的方法也會被調用：

```kotlin
val plants: LiveData<List<Plant>> =
    plantDao.getPlants().switchMap { plantList->
        liveData {
            withContext(defaultDispatcher) {
                val customSortOrder = plantsListSortOrderCache.getOrAwait()
                emit(plantList.applySort(customSortOrder))
            }
        }
    }
```


之所以使用 `switchMap` 而非 `map` 是因為其中 `transform` 的類型：

```kotlin
// transform: (X) -> LiveData<Y>
public inline fun <X, Y> LiveData<X>.switchMap(
    crossinline transform: (X) -> LiveData<Y>
): LiveData<Y> = Transformations.switchMap(this) { transform(it) }

// transform: (X) -> Y
public inline fun <X, Y> LiveData<X>.map(
    crossinline transform: (X) -> Y
): LiveData<Y> =Transformations.map(this) { transform(it) }
```

相同的，我們也可以將 `getPlantsWithGrowZone` 方法改成：

```kotlin
@AnyThread
suspend fun List<Plant>.applyMainSafeSort(customSortOrder: List<String>) =
    withContext(defaultDispatcher) {
        this@applyMainSafeSort.applySort(customSortOrder)
    }

fun getPlantsWithGrowZone(growZone: GrowZone) =
    plantDao.getPlantsWithGrowZoneNumber(growZone.number)
      // retrieve LiveData<List<Plant>>
      .switchMap { plantList ->
            liveData {
                val customSortOrder = plantsListSortOrderCache.getOrAwait()
                emit(plantList.applyMainSafeSort(customSortOrder))
            }
      }
```


# Flow

終於我們要進入 **Flow** 了。

><br>
>
>Flow 是非同步化的 [Sequence](https://kotlinlang.org/docs/reference/sequences.html)，他並不會一次性地生成全部所需要的值，而是一個接一個地產生。
><br><br/>

## 簡單範例

我們首先來看看一些簡單的範例：

```kotlin
fun testFlow() {
    val context = EmptyCoroutineContext
    val supervisorJob = SupervisorJob(context[Job])
    val scope = CoroutineScope(context + supervisorJob)

    fun makeFlow() = flow {
        println("sending first value")
        emit(1)
        println("first value collected, sending another value")
        emit(2)
        println("second value collected, sending a third value")
        emit(3)
        println("done")
    }

    scope.launch {
        makeFlow().collect { value ->
            println("got $value")
        }
        println("flow is completed")
    }
}

/**
結果

sending first value
got 1
first value collected, sending another value
got 2
second value collected, sending a third value
got 3
done
flow is completed

**/
```

從結果上，很顯然 `flow` 裡面是同步的行為。

當中的 `emit` 與 `collect` 則是相互搭配。
通過 `emit` 我們可以將值傳送出去，並經由 `collect` 將其收集。

因此我們可以説 `flow` 是 **Producer** 而 `collect` 是 **Consumer**。

接下來，我們來看看 `flow` 的源碼吧。

### flow 的源碼

```kotlin
public fun <T> flow(
    @BuilderInference block: suspend FlowCollector<T>.() -> Unit): Flow<T> =
        SafeFlow(block)

// FlowCollector
public fun interface FlowCollector<in T> {
    public suspend fun emit(value: T)
}

private class SafeFlow<T>(private val block: suspend FlowCollector<T>.() -> Unit) : AbstractFlow<T>() {
    override suspend fun collectSafely(collector: FlowCollector<T>) {
        collector.block()
    }
}
```

根據源碼，我們知道當我們調用 `flow` 時，我們其實是在建立一個 **AbstractFlow** 物件：

```kotlin
fun makeFlow() : AbstractFlow { ... }
```

而當我們在調用 `makeFlow().collect` 時，我們其實是在調用 **AbstractFlow** 中的 `collect` 並間接調用 `collectSafely`：

```kotlin
@FlowPreview
public abstract class AbstractFlow<T> : Flow<T>, CancellableFlow<T> {

    public final override suspend fun collect(collector: FlowCollector<T>) {
        val safeCollector = SafeCollector(collector, coroutineContext)
        try {
            collectSafely(safeCollector)
        } finally {
            safeCollector.releaseIntercepted()
        }
    }

    /**
     * Accepts the given [collector] and [emits][FlowCollector.emit] values into it.
     *
     * A valid implementation of this method has the following constraints:
     * 1) It should not change the coroutine context (e.g. with `withContext(Dispatchers.IO)`) when emitting values.
     *    The emission should happen in the context of the [collect] call.
     *    Please refer to the top-level [Flow] documentation for more details.
     * 2) It should serialize calls to [emit][FlowCollector.emit] as [FlowCollector] implementations are not
     *    thread-safe by default.
     *    To automatically serialize emissions [channelFlow] builder can be used instead of [flow]
     *
     * @throws IllegalStateException if any of the invariants are violated.
     */
    public abstract suspend fun collectSafely(collector: FlowCollector<T>)
}
```

但在調用 `collectSafely` 之前，會先創建 **SafeCollector** 並傳入 `collectSafely`。

如此一來，當我們調用 `collectSafely` 時，我們會調用 `collector.block()` 。 此時的 `collector` 就是 **SafeCollector**。

而 `block` 則是：

```kotlin
{
    println("sending first value")
    emit(1)
    println("first value collected, sending another value")
    emit(2)
    println("second value collected, sending a third value")
    emit(3)
    println("done")
}
```

由於 `emit` 調用的其實是 **SafeCollector** 中的方法，所以接下來我們需要看看 **SafeCollector** 到底有什麼功能了。

#### SafeCollector 的作用

所謂的 **SafeCollector** 其實是一個 **Continuation**。

```kotlin
@Suppress("CANNOT_OVERRIDE_INVISIBLE_MEMBER", "INVISIBLE_MEMBER", "INVISIBLE_REFERENCE", "UNCHECKED_CAST")
internal actual class SafeCollector<T> actual constructor(
    @JvmField internal actual val collector: FlowCollector<T>,
    @JvmField internal actual val collectContext: CoroutineContext
) : FlowCollector<T>, ContinuationImpl(NoOpContinuation, EmptyCoroutineContext), CoroutineStackFrame
```

而當我們調用 `emit` 時，我們其實是在調用以下：

```kotlin
/**
 * This is a crafty implementation of state-machine reusing.
 * First it checks that it is not used concurrently (which we explicitly prohibit) and
 * then just cache an instance of the completion in order to avoid extra allocation on each emit,
 * making it effectively garbage-free on its hot-path.
 */
override suspend fun emit(value: T) {
    return suspendCoroutineUninterceptedOrReturn sc@{ uCont ->
        try {
            emit(uCont, value)
        } catch (e: Throwable) {
            // Save the fact that exception from emit (or even check context) has been thrown
            lastEmissionContext = DownstreamExceptionElement(e)
            throw e
        }
    }
}
```

通過 `suspendCoroutineUninterceptedOrReturn` 我們可以取得 suspend 方法的 `Continuation`。

此 `uCont` 便會被傳入 `emit` ：

```kotlin

private fun emit(uCont: Continuation<Unit>, value: T): Any? {
    val currentContext = uCont.context
    // 1. 檢查 CoroutineContext 是否還在運作
    currentContext.ensureActive()
    // This check is triggered once per flow on happy path.
    // 2. lastEmissionContext 預設為 null
    val previousContext = lastEmissionContext
    if (previousContext !== currentContext) {
        // 3. 更新 lastEmissionContext = currentContext
        checkContext(currentContext, previousContext, value)
    }
    completion = uCont

    // 4. 調用 emitFun 並回傳
    return emitFun(collector as FlowCollector<Any?>, value, this as Continuation<Unit>)
}
```

最後調用的 `emitFun` 其實是 ：

```kotlin
@Suppress("UNCHECKED_CAST")
private val emitFun =
    FlowCollector<Any?>::emit as Function3<FlowCollector<Any?>, Any?, Continuation<Unit>, Any?>
```

雖然從源碼中看不出來 `emitFun` 到底是做什麼，但如果單單將他轉換成 Java 後我們就可以看出端倪了：

```java
private final Function3 emitFun = (Function3)TypeIntrinsics.beforeCheckcastToFunctionOfArity(new Function3() {
    // $FF: synthetic method
    // $FF: bridge method
    public Object invoke(Object var1, Object var2, Object var3) {
       return this.invoke((FlowCollector)var1, var2, (Continuation)var3);
    }

    @Nullable
    public final Object invoke(@NotNull FlowCollector p1, @Nullable Object p2, @NotNull Continuation continuation) {
       return p1.emit(p2, continuation);
    }
 }, 3);
```

而最後面的 `p1.emit` 便是關鍵所在。

說到這裡可能會比較容易混淆，所以下面會做個總整並說明 `p1.emit` 做了什麼。

#### 總整

目前的流程是如下：

><br>
>
>1. 使用 `flow` 建立 **Flow** 或 **SafeFlow**
><br><br/>

```kotlin
public fun <T> flow(@BuilderInference block: suspend FlowCollector<T>.() -> Unit): Flow<T> = SafeFlow(block)

// SafeFlow :> AbstractFlow :> Flow
fun makeFlow() : AbstractFlow = flow { ... }
```

><br>
>
>2. 調用 **SafeFlow** 的 `collect`
><br><br/>

```kotlin
scope.launch {
  makeFlow()
      .collect { value ->
      println("got $value")
  }
  println("flow is completed")
}
```

此時我們會調用 **AbstractFlow** 中的 `collect`：

<center>
<img src = "/images/posts/jekyll/codelab/advanced-kotlin-coroutines/flow-to-flowcollector.png"/>
</center>

<br>

調用 `collect` 的最後會調用 `collector.block()`。

此時的 `collector` 是 **SafeCollector**，而所謂的 `block` 其實是：

```kotlin
{
    println("sending first value")
    emit(1)
    println("first value collected, sending another value")
    emit(2)
    println("second value collected, sending a third value")
    emit(3)
    println("done")
}
```

><br>
>
>4. 通過 `collect` 啟動 `block`
><br><br/>

當我們調用 `block` 時，雖然我們只看到 `flow` 但其實更完整的型態會是如此：

```kotlin
flow { flowCollector: FlowCollector ->
    ...
}
```

因此，當我們在 `block` 中調用 `emit` 時，其實是調用 **SafeCollector** 的 `emit`：

```kotlin
override suspend fun emit(value: T) {
    return suspendCoroutineUninterceptedOrReturn sc@{ uCont ->
        try {
            emit(uCont, value)
        } catch (e: Throwable) {
            // Save the fact that exception from emit (or even check context) has been thrown
            lastEmissionContext = DownstreamExceptionElement(e)
            throw e
        }
    }
}
```

而 `emit(uCont, value)` 會調用 `emitFun` ：


```kotlin
private fun emit(uCont: Continuation<Unit>, value: T): Any? {
    val currentContext = uCont.context
    currentContext.ensureActive()
    // This check is triggered once per flow on happy path.
    val previousContext = lastEmissionContext
    if (previousContext !== currentContext) {
        checkContext(currentContext, previousContext, value)
    }
    completion = uCont
    return emitFun(collector as FlowCollector<Any?>, value, this as Continuation<Unit>)
}
```

><br>
>
>5. 通過 `emitFun` 啟動 `p1.emit`
><br><br/>

這就回到我們之前說的， `emitFun` 轉換成 Java 後會變成：

```java
private final Function3 emitFun = (Function3)TypeIntrinsics.beforeCheckcastToFunctionOfArity(new Function3() {
    // $FF: synthetic method
    // $FF: bridge method
    public Object invoke(Object var1, Object var2, Object var3) {
       return this.invoke((FlowCollector)var1, var2, (Continuation)var3);
    }

    @Nullable
    public final Object invoke(@NotNull FlowCollector p1, @Nullable Object p2, @NotNull Continuation continuation) {
       return p1.emit(p2, continuation);
    }
 }, 3);
```

所以這裡的 `p1` 其實是 `collect` 中的那段：

```kotlin
makeFlow()
    .collect { value ->
        // 這段
        println("got $value")
    }

// 寫更清楚一點點
makeFlow().collect(object: FlowCollector<Int> {
    override suspend fun emit(value: Int) {
        println("got $value")
    }
})
```

現在看懂了嗎？

其實 `p1.emit` 就是將資料傳給 `collect` 中的 **FlowCollector**。


><br>
>
>**總結**
>`flow` 的 `emit` 只會在 `collect` 有存在時才會有反應 ... 也就是所謂的 **Cold Streams**。
><br><br/>

# Flow 的特性

## 無法跨執行緒

**Flow** 主要的限制便是 **not thread safe**，無法跨執行緒執行。

```kotlin
flow {
    emit(1) // Ok
    withContext(Dispatchers.IO) {
        emit(2) // Will fail with ISE
    }
}
```

以上的程式碼會拋出以下的錯誤訊息：

```shell
java.lang.IllegalStateException: Flow invariant is violated:
    Flow was collected in [StandaloneCoroutine{Active}@13789336, Dispatchers.Default],
    but emission happened in [DispatchedCoroutine{Active}@18bde50d, Dispatchers.IO].
    Please refer to 'flow' documentation or use 'flowOn' instead
```

### flowOn

按照提示，他建議我們使用 `flowOn`。

但 `flowOn` 其實只能將當下的 `flow` 在某 Coroutine 中進行 。

在沒使用 `flowOn` 的情況下：

```kotlin
fun testFlow() {
    val context = EmptyCoroutineContext
    val supervisorJob = SupervisorJob(context[Job])
    val scope = CoroutineScope(context + supervisorJob)

    fun testFlow() = flow {
        println("THREAD: ${Thread.currentThread()} emit")
        emit(1)
    }

    scope.launch{
        println("THREAD: ${Thread.currentThread()} run test")
        testFlow().collect {
            println("THREAD: ${Thread.currentThread()} got $it")
        }
    }
}

/*
    我們會得到
    THREAD: Thread[DefaultDispatcher-worker-2,5,main] run test
    THREAD: Thread[DefaultDispatcher-worker-2,5,main] emit
    THREAD: Thread[DefaultDispatcher-worker-2,5,main] got 1
*/
```

但使用 `flowOn` 後：

```kotlin
fun testFlow() {
    val context = EmptyCoroutineContext
    val supervisorJob = SupervisorJob(context[Job])
    val scope = CoroutineScope(context + supervisorJob)

    fun testFlow() = flow {
        println("THREAD: ${Thread.currentThread()} emit")
        emit(1)
    }.flowOn(Dispatchers.Main)

    scope.launch {
        println("THREAD: ${Thread.currentThread()} run test")
        testFlow().collect {
            println("THREAD: ${Thread.currentThread()} got $it")
        }
    }
}

/*
    我們會得到
    THREAD: Thread[DefaultDispatcher-worker-2,5,main] run test
    THREAD: Thread[main,5,main] emit
    THREAD: Thread[DefaultDispatcher-worker-1,5,main] got 1
*/
```

從結果可以知道，原來 `flowOn` 會讓 `emit` 或 `block` 由指定的 **Dispatcher** 中進行。

#### flowOn 的原理

`flowOn` 的功能會依照傳入的 `context` 將 **Flow** 以不同的類別包裹起來。

```kotlin
public fun <T> Flow<T>.flowOn(context: CoroutineContext): Flow<T> {
    checkFlowContext(context)
    return when {
        // 1. 若 context 為 EmptyCoroutineContext
        //    便不會改變
        context == EmptyCoroutineContext -> this

        // 2. 若 Flow 為 FusibleFlow
        //    那便使用 context 重新創造一個新的 Flow
        this is FusibleFlow -> fuse(context = context)

        // 3. 回傳 ChannelFlowOperatorImpl
        else -> ChannelFlowOperatorImpl(this, context = context)
    }
}
```

從我們剛剛的源碼，我們知道我們此時的 **Flow** 是 **SafeFlow**，因此會進入 `else` 的情況。

所以 `testFlow()` 會得到 **ChannelFlowOperatorImpl** 或更精準 `ChannelFlowOperatorImpl(SafeFlow, context)`：

```kotlin
internal class ChannelFlowOperatorImpl<T>(
    flow: Flow<T>,
    context: CoroutineContext = EmptyCoroutineContext,
    capacity: Int = Channel.OPTIONAL_CHANNEL,
    onBufferOverflow: BufferOverflow = BufferOverflow.SUSPEND
) : ChannelFlowOperator<T, T>(flow, context, capacity, onBufferOverflow) {

    override fun create(context: CoroutineContext, capacity: Int, onBufferOverflow: BufferOverflow): ChannelFlow<T> =
        ChannelFlowOperatorImpl(flow, context, capacity, onBufferOverflow)

    // 若回傳 null 就表示 ChannelFlow 必須使用 Channel
    override fun dropChannelOperators(): Flow<T> = flow

    override suspend fun flowCollect(collector: FlowCollector<T>) =
        flow.collect(collector)
}
```

這便是 `testFlow()` 所得到的類別。

接下來，我們需要查看 `testFlow().collect` 的效果了。

### flowOn.collect 的效果

我們無法在 **ChannelFlowOperatorImpl** 中看到 `collect` 的調用後的結果。 我們需要進一步看 **ChannelFlowOperator** 的部分源碼：

```kotlin
// ChannelFlow implementation that operates on another flow before it
internal abstract class ChannelFlowOperator<S, T>(
    @JvmField protected val flow: Flow<S>,
    context: CoroutineContext,
    capacity: Int,
    onBufferOverflow: BufferOverflow
) : ChannelFlow<T>(context, capacity, onBufferOverflow) {

    /*  ...  */

    // Optimizations for fast-path when channel creation is optional
    override suspend fun collect(collector: FlowCollector<T>) {
        // Fast-path: When channel creation is optional (flowOn/flowWith operators without buffer)
        if (capacity == Channel.OPTIONAL_CHANNEL) {

            /*
                1. 預設會更新 context
            */
            val collectContext = coroutineContext
            val newContext = collectContext + context // compute resulting collect context
            // #1: If the resulting context happens to be the same as it was -- fallback to plain collect
            if (newContext == collectContext)
                return flowCollect(collector)
            // #2: If we don't need to change the dispatcher we can go without channels
            if (newContext[ContinuationInterceptor] == collectContext[ContinuationInterceptor])
                return collectWithContextUndispatched(collector, newContext)
        }
        // Slow-path: create the actual channel
        super.collect(collector)
    }
}
```

這裡的 **FlowCollector** 是 `collect` 中的那個 block ：

```kotlin
testFlow().collect {
    // 這裡定義了 FlowCollector
    // 並實作 emit 行為
    println("THREAD: ${Thread.currentThread()} got $it")
}
```

預設的情況下，且 **CoroutineContext** 沒有變化時，就會有以下行為：

<center>
<img src = "/images/posts/jekyll/codelab/advanced-kotlin-coroutines/flowOn-default-same-context.svg"/>
</center><br>

<u>但如果 **CoroutineContext** 有更新，但兩個 **ContinuationInterceptor** 不變</u> :

那便會調用 `collectWithContextUndispatched` 並做以下的事：

1. 將的 `collect` 中的 `block: FlowCollector` 由 **UndispatchedContextCollector** 包裹起來。 但若 `block` 並非 **FlowCollector** 而是 **SendingCollector** 或 **NopCollector** 那便不需要包裹。

2. 調用 `withContextUndispatched` 並使用 `withCoroutineContext` 讓 `flowCollect(it)` 在新的 **CoroutineContext** 中進行。


最後，若如同此次情況：
<u>**CoroutineContext** 有更新，且 **ContinuationInterceptor** 也不同，那就會調用 **ChannelFlow** 的 `collect`</u>。


```kotlin
@InternalCoroutinesApi
public abstract class ChannelFlow<T>(
    // upstream context
    @JvmField public val context: CoroutineContext,
    // buffer capacity between upstream and downstream context
    @JvmField public val capacity: Int,
    // buffer overflow strategy
    @JvmField public val onBufferOverflow: BufferOverflow
) : FusibleFlow<T> {

    // ...

    override suspend fun collect(collector: FlowCollector<T>): Unit =
        coroutineScope {
            collector.emitAll(produceImpl(this))
        }

    /*
        創建 ReceiveChannel
    */
    public open fun produceImpl(scope: CoroutineScope): ReceiveChannel<T> =
        scope.produce(context, produceCapacity, onBufferOverflow, start = CoroutineStart.ATOMIC, block = collectToFun)
}
```

`emitAll` 所傳入的值是 `produceImpl` 所生成的 **ReceiveChannel**。

`produceImpl` 主要會做兩件事：

   1. 建立 **Channel**。 由於 `capacity` 是 `-2` 或 `BUFFERED` 因此會創建 **ArrayChannel**。
      ```kotlin
      BUFFERED -> ArrayChannel( // uses default capacity with SUSPEND
        if (onBufferOverflow == BufferOverflow.SUSPEND) CHANNEL_DEFAULT_CAPACITY else 1,
        onBufferOverflow, onUndeliveredElement
      )
      ```
   2. 建立、 記錄 並 回傳 **ProducerCoroutine**
      ```kotlin
      val coroutine = ProducerCoroutine(newContext, channel)
      coroutine.start(start, coroutine, block)
      return coroutine

      // ProducerCoroutine 的定義
      private class ProducerCoroutine<E>(
          parentContext: CoroutineContext, channel: Channel<E>
      ) : ChannelCoroutine<E>(parentContext, channel, true, active = true), ProducerScope<E>
      ```
<br>

由於我們調用了 `coroutine.start` 因此我們的 `block` 行為會被記錄在 **Task** 中，並會在 **BaseContinuationImpl** 的 `resumeWith` 被調用時被調用。

這裡的 `block` 則是 **ChannelFlow** 的：

```kotlin
internal val collectToFun: suspend (ProducerScope<T>) -> Unit
    get() = { collectTo(it) }
```

我們先來看看 `collectTo` 的作用吧。

#### collectTo 的作用

通過 `collectTo` **ChannelFlowOperator** 會使用 **ProducerCoroutine** 來創建 **SendingCollector** 並傳入 `flowCollect`：

```kotlin
// Slow path when output channel is required
protected override suspend fun collectTo(scope: ProducerScope<T>) =
    flowCollect(SendingCollector(scope))
```

`flowCollect` 會有以下行為：

<center>
<img src = "/images/posts/jekyll/codelab/advanced-kotlin-coroutines/flowOn-collectTo.svg"/>
</center>

<br>

從之前一樣，通過 `block` 中的 `emit` 最後便會調用 `emitFun`。

但此時的 `emitFun` 所會調用的是 **SendingCollector** 的 `emit` ：

```kotlin
@InternalCoroutinesApi
public class SendingCollector<T>(
    private val channel: SendChannel<T>
) : FlowCollector<T> {
    override suspend fun emit(value: T): Unit = channel.send(value)
}
```

`emit` 最後會調用 **ProducerCoroutine** 的 `send`。
但因為 **ProducerCoroutine** 會 delegate (委託) 給 `channel`。 因此我們需要看 **ArrayChannel** 的 `send`，或真正實作者 **AbstractSendChannel** 的 `send` ：

```kotlin
public final override suspend fun send(element: E) {
    // fast path -- try offer non-blocking
    if (offerInternal(element) === OFFER_SUCCESS) return
    // slow-path does suspend or throws exception
    return sendSuspend(element)
}
```

通過 `offerInternal` 以下事情會發生：
1. 通過 `updateBufferSize` 更新 `buffer` 大小
2. 如果當下的 `size` 為 **0**：
    1. `element` 或 `emit(1)` 中的 **1** 會通過 **ReceiveElement** 的 `tryResumeReceive` 讓 **CancellableContinuationImpl** 更新當中的 `_state` 並讓 `resume`。
   2. 從 `tryResumeReceive` 得到 `RESUME_TOKEN`。 若確認無誤，便會直接回傳 `OFFER_SUCCESS`。
3. 若當下 `size != 0`，那便會調用 `enqueueElement` 將 `element` 存入 **ArrayChannel** 的 `buffer` 中。

```kotlin
private var buffer: Array<Any?> = arrayOfNulls<Any?>(min(capacity, 8)).apply { fill(EMPTY) }
```

若以上都順利進行，最後會回傳 `OFFER_SUCCESS` 至 `send` 方法。

`buffer` 會在之後通過 `receiveCatching` 讀取。

但若值沒有存放在 `buffer` 中，那便會由 **CancellableContinuationImpl** 中的 `_state` 讀取。

這便是 `collectTo` 的完整流程。

#### 回到 emitAll

得到 **ProducerCoroutine** 後，他會經由 `emitAll` 傳送出去。

`emitAll` 的實作最後會調用 `emitAllImpl` ：

```kotlin
public suspend fun <T> FlowCollector<T>.emitAll(channel: ReceiveChannel<T>): Unit =
    emitAllImpl(channel, consume = true)

private suspend fun <T> FlowCollector<T>.emitAllImpl(channel: ReceiveChannel<T>, consume: Boolean) {
    ensureActive()
    // Manually inlined "consumeEach" implementation that does not use iterator but works via "receiveCatching".
    // It has smaller and more efficient spilled state which also allows to implement a manual kludge to
    // fix retention of the last emitted value.
    // See https://youtrack.jetbrains.com/issue/KT-16222
    // See https://github.com/Kotlin/kotlinx.coroutines/issues/1333
    var cause: Throwable? = null
    try {
        while (true) {

            val result = run { channel.receiveCatching() }
            if (result.isClosed) {
                result.exceptionOrNull()?.let { throw it }
                break // returns normally when result.closeCause == null
            }
            emit(result.getOrThrow())
        }
    } catch (e: Throwable) {
        cause = e
        throw e
    } finally {
        if (consume) channel.cancelConsumed(cause)
    }
}
```

通過 `emitAll`，我們會調用 **ReceiveChannel** 或 **AbstractChannel** 的 `receiveCatching` 並從 **ArrayChannel** 的 `pollInternal` 來取得 `buffer` 中的值。

`buffer` 中的值就是通過 `emit` 傳進來的值。

最後再由 **FlowCollector** 的 `emit` 傳至 `collect` 中的 `block`：

```kotlin
testFlow().collect {
    println("THREAD: ${Thread.currentThread()} got $it")
}
```

但若無法通過 `pollInternal` 從 `buffer` 中無法取得任何的值，那便會調用 `receiveSuspend`。

```kotlin
@Suppress("UNCHECKED_CAST")
private suspend fun <R> receiveSuspend(receiveMode: Int): R = suspendCancellableCoroutineReusable sc@ { cont ->
    val receive = if (onUndeliveredElement == null)
        ReceiveElement(cont as CancellableContinuation<Any?>, receiveMode) else
        ReceiveElementWithUndeliveredHandler(cont as CancellableContinuation<Any?>, receiveMode, onUndeliveredElement)
    while (true) {
        if (enqueueReceive(receive)) {
            removeReceiveOnCancel(cont, receive)
            return@sc
        }
        // hm... something is not right. try to poll
        val result = pollInternal()
        if (result is Closed<*>) {
            receive.resumeReceiveClosed(result)
            return@sc
        }
        if (result !== POLL_FAILED) {
            cont.resume(receive.resumeValue(result as E), receive.resumeOnCancellationFun(result as E))
            return@sc
        }
    }
}
```

`receiveSuspend` 中的最重要的方法是 `suspendCancellableCoroutineReusable` ：

```kotlin
internal suspend inline fun <T> suspendCancellableCoroutineReusable(
    crossinline block: (CancellableContinuation<T>) -> Unit
): T = suspendCoroutineUninterceptedOrReturn { uCont ->
    val cancellable = getOrCreateCancellableContinuation(uCont.intercepted())
    block(cancellable)
    cancellable.getResult()
}
```



`suspendCancellableCoroutineReusable` 會通過 `getOrCreateCancellableContinuation` 來取得  **CancellableContinuation** 並傳入 `block` 中。





## channelFlow

如果我們想要

## hot Flow

## Going async with flow

```kotlin
// This code is a simplified version of how Room implements flow
fun <T> createFlow(query: Query, tables: List<Tables>): Flow<T> = flow {
    val changeTracker = tableChangeTracker(tables)

    while(true) {
        emit(suspendQuery(query))
        changeTracker.suspendUntilChanged()
    }
}
```

`suspendQuery`
a main-safe function that runs a regular Room suspend query.

`suspendUntilChanged`
a function that suspends the coroutine until one of the tables changes.


# 轉換成 Flow

```kotlin
@Query("SELECT * FROM plants ORDER BY name")
fun getPlants(): LiveData<List<Plant>>

@Query("SELECT * FROM plants WHERE growZoneNumber = :growZoneNumber ORDER BY name")
fun getPlantsWithGrowZoneNumber(growZoneNumber: Int): LiveData<List<Plant>>

// 轉換成
@Query("SELECT * from plants ORDER BY name")
fun getPlantsFlow(): Flow<List<Plant>>

@Query("SELECT * from plants WHERE growZoneNumber = :growZoneNumber ORDER BY name")
fun getPlantsWithGrowZoneNumberFlow(growZoneNumber: Int): Flow<List<Plant>>
```

By specifying a Flow return type, Room executes the query with the following characteristics:

Main-safety
Queries with a Flow return type always run on the Room executors, so they are always main-safe. You don't need to do anything in your code to make them run off the main thread.

Observes changes
Room automatically observes changes and emits new values to the flow.

Async sequence
Flow emits the entire query result on each change, and it won't introduce any buffers. If you return a Flow<List<T>>, the flow emits a List<T> that contains all rows from the query result. It will execute just like a sequence – emitting one query result at a time and suspending until it is asked for the next one.

Cancellable
When the scope that's collecting these flows is cancelled, Room cancels observing this query.




# Reference

[Wayne's Talk](https://waynestalk.com/en/kotlin-coroutine-flow-tutorial-en/)





<br><br><br><br><br><br>
