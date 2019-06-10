[TOC]

# 简单解析 okhttp3

* 

## 基本用法

1. 导库，网络权限
2. 通过 RequestBuilder 通过建造者模式来构建我们的请求，然后 build() 得到 Request 对象，
3. 通过 OkhttpClient 对象来 newCall(reqquest) 构建 Call 对象
4. 最后就是调用 Call 对象的 enqueue(异步，且回调并非在 UI 线程) 或者 execute（同步，不给回调） 方法

* 整个流程是:通过 OkHttpClient 将构建的 Request 转换为 Call ，然后在 RealCall 中选择进行异步或同步任务，最后通过一些的拦截器链 interceptor 发出网络请求和得到返回的 response 。

## 首先先解析execute同步

* 通过 call.excute() 方法来提交同步请求，这种方式会阻塞线程，而为了避免 ANR 异常,而且安卓也不允许在主线程里访问网络.所以要开启子线程来同步访问网络，要操作 UI 就直接 handler 机制咯。

* 是 Call 对象执行的，所以我们先找到 Call 对象

  ~~~java
  @Override 
  public Call newCall(Request request) {
      return RealCall.newRealCall(this, request, false /* for web socket */);
  }
  
  static RealCall newRealCall(OkHttpClient client, Request originalRequest, boolean forWebSocket) {
      // Safely publish the Call instance to the EventListener.
      RealCall call = new RealCall(client, originalRequest, forWebSocket);
      call.eventListener = client.eventListenerFactory().create(call);
      return call;
  }
  
  private RealCall(OkHttpClient client, Request originalRequest, boolean forWebSocket) {
      this.client = client;
      this.originalRequest = originalRequest;
      this.forWebSocket = forWebSocket;
      this.retryAndFollowUpInterceptor = new RetryAndFollowUpInterceptor(client, forWebSocket);
  }
  //直接去 Call 里找到 execute  执行方法，返回值为 Response 对象，我们拿到这个对象后可以进行一系列操作
  @Override public Response execute() throws IOException {
      synchronized (this) {
        if (executed) throw new IllegalStateException("Already Executed");
        executed = true;
      }
      captureCallStackTrace();
      eventListener.callStart(this);
      try {
          //先去调度器里 executed，我们先去看下做了什么
        client.dispatcher().executed(this);
          //加进调度器里的同步请求队列后就调用下面这个方法拿到最终的请求结果 Response 对象
        Response result = getResponseWithInterceptorChain();
        if (result == null) throw new IOException("Canceled");
        return result;
      } catch (IOException e) {
        eventListener.callFailed(this, e);
        throw e;
      } finally {
          //执行完后，就去调度器里 把这个 call 从队列里移除掉
        client.dispatcher().finished(this);
      }
  }
  
  /** Used by {@code Call#execute} to signal it is in-flight. */
    synchronized void executed(RealCall call) {
        //将 call 加进正在运行的同步请求队列中，这里只是为了让 调度器可以管理这些请求，比如可以在中途通过调度器来中断请求
      runningSyncCalls.add(call);
  }
  /** Running synchronous calls. Includes canceled calls that haven't finished yet. */
  private final Deque<RealCall> runningSyncCalls = new ArrayDeque<>();
  ~~~

* 丢到 Dispatcher 调度器对象里的同步请求队列里后，执行 getResponseWithInterceptorChain()最重要的一个方法;

  ~~~java
  Response getResponseWithInterceptorChain() throws IOException {
      // Build a full stack of interceptors.首先先创建拦截器列表
      List<Interceptor> interceptors = new ArrayList<>();
      //第一个拦截器就是 client.interceptors()，用户可通过OkhttpClient.Builder()构建这个 Client 对象的时候传进的自定义拦截器
      interceptors.addAll(client.interceptors());
      // retryAndFollowUpInterceptor（失败重连，重定向）
      interceptors.add(retryAndFollowUpInterceptor);
      //接下来是 装换用户的请求，转换服务器的响应
      interceptors.add(new BridgeInterceptor(client.cookieJar()));
      //读取缓存，修改缓存
      interceptors.add(new CacheInterceptor(client.internalCache()));
      //与服务器建立连接
      interceptors.add(new ConnectInterceptor(client));
      if (!forWebSocket) {//forWebSocket此时为false
          //外部配置的 network 拦截器
        interceptors.addAll(client.networkInterceptors());
      }
      //最后是 发起网络请求接收数据
      interceptors.add(new CallServerInterceptor(forWebSocket));
  //创建拦截器链，传进 拦截器链
      Interceptor.Chain chain = new RealInterceptorChain(interceptors, null, null, null, 0,
          originalRequest, this, eventListener, client.connectTimeoutMillis(),
          client.readTimeoutMillis(), client.writeTimeoutMillis());
  //然后执行 process 方法，我们到这个方法里看
      return chain.proceed(originalRequest);
    }
  ~~~

* 现在先去看看这个请求拦截链怎么创建的

  ~~~java
  public RealInterceptorChain(List<Interceptor> interceptors, StreamAllocation streamAllocation,
        HttpCodec httpCodec, RealConnection connection, int index, Request request, Call call,
        EventListener eventListener, int connectTimeout, int readTimeout, int writeTimeout) {
      this.interceptors = interceptors;
      this.connection = connection;
      this.streamAllocation = streamAllocation;
      this.httpCodec = httpCodec;
      this.index = index;
      this.request = request;
      this.call = call;
      this.eventListener = eventListener;
      this.connectTimeout = connectTimeout;
      this.readTimeout = readTimeout;
      this.writeTimeout = writeTimeout;
  }
  ~~~

* 再看 proceed 方法

  ~~~java
  @Override public Response proceed(Request request) throws IOException {
      return proceed(request, streamAllocation, httpCodec, connection);
  }
  
  public Response proceed(Request request, StreamAllocation streamAllocation, HttpCodec httpCodec,
        RealConnection connection) throws IOException {
      
      if (index >= interceptors.size()) throw new AssertionError();
  
      calls++;
  
      // If we already have a stream, confirm（验证） that the incoming request will use it.
      if (this.httpCodec != null && !this.connection.supportsUrl(request.url())) {
        throw new IllegalStateException("network interceptor " + interceptors.get(index - 1)
            + " must retain the same host and port");
      }
  
      // If we already have a stream, confirm that this is the only call to chain.proceed().
      if (this.httpCodec != null && calls > 1) {
        throw new IllegalStateException("network interceptor " + interceptors.get(index - 1)
            + " must call proceed() exactly once");
      }
  
      // Call the next interceptor in the chain.创建一个 下一个RealInterceptorChain 对象
      RealInterceptorChain next = new RealInterceptorChain(interceptors, streamAllocation, httpCodec,
          connection, index + 1, request, call, eventListener, connectTimeout, readTimeout,
          writeTimeout);
      Interceptor interceptor = interceptors.get(index);
      //这里返回了请求结果 Response 对象，现在去拦截器的intercept（方法），这里可以看出应该使用了责任链模式
      Response response = interceptor.intercept(next);
  
      // Confirm that the next interceptor made its required call to chain.proceed().
      if (httpCodec != null && index + 1 < interceptors.size() && next.calls != 1) {
        throw new IllegalStateException("network interceptor " + interceptor
            + " must call proceed() exactly once");
      }
  
      // Confirm that the intercepted response isn't null.
      if (response == null) {
        throw new NullPointerException("interceptor " + interceptor + " returned null");
      }
  
      if (response.body() == null) {
        throw new IllegalStateException(
            "interceptor " + interceptor + " returned a response with no body");
      }
  
      return response;
  }
  ~~~

  **OkHttp拦截器分析**

  * 责任链模式

  先列出之前在 getResponseWithInterceptorChain 方法中添加的各Interceptor，概括一下它们分别负责什么功能： 

  * client.interceptors()用户自定义的Interceptor，能拦截到所有的请求 (我们可以在开始拦截前传入自定义的拦截器)

  * RetryAndFollowUpInterceptor负责失败重连和重定向相关 

  * BridgeInterceptor负责配置请求的头信息，比如Keep-Alive、gzip、Cookie等可以优化请求 

  * CacheInterceptor负责缓存管理，使用DiskLruCache做本地缓存，CacheStrategy决定缓存策略 

  * ConnectInterceptor开始与目标服务器建立连接，获得RealConnection 

  * client.networkInterceptors()用户自定义的Interceptor，仅在生产网络请求时生效  (我们可以在开始拦截最后传入自定义的拦截器)

  * CallServerInterceptor向服务器发出一次网络请求的地方。

    

  * RetryAndFollowUpInterceptor

  ```java
  public Response intercept(Chain chain) throws IOException {
      Request request = chain.request();
      //streamAllocation对象是在这个Interceptor中创建的,
  streamAllocation = new StreamAllocation(
      client.connectionPool(), createAddress(request.url()), callStackTrace);
  
  int followUpCount = 0;
  Response priorResponse = null;
  while (true) {//这里是个循环，如果一次循环后需要重连或者重定向，那么继续轮训重走网络请求的流程
    if (canceled) {
      streamAllocation.release();
      throw new IOException("Canceled");
    }
  
    Response response = null;
    boolean releaseConnection = true;
    try {//这个拦截器不做拦截的什么操作，这个拦截器是等最后拿到 response 来判断 请求是否失败，是否需要重连，重定向
      response = ((RealInterceptorChain) chain).proceed(request, streamAllocation, null, null);
      releaseConnection = false;
    } catch (RouteException e) {
      // 尝试通过路由连接失败。请求将不会被发送
      if (!recover(e.getLastConnectException(), false, request)) {
        throw e.getLastConnectException();
      }
      releaseConnection = false;
      continue;
    } catch (IOException e) {
      // 尝试与服务器通信失败。请求可能已发送。
      boolean requestSendStarted = !(e instanceof ConnectionShutdownException);
      if (!recover(e, requestSendStarted, request)) throw e;
      releaseConnection = false;
      continue;
    } finally {
      // 我们正在抛出未经检查的异常。释放所有资源。
      if (releaseConnection) {
        streamAllocation.streamFailed(null);
        streamAllocation.release();
      }
    }
  
    // 附加先前的响应（如果存在）。不添加body
    if (priorResponse != null) {
      response = response.newBuilder()
          .priorResponse(priorResponse.newBuilder()
                  .body(null)
                  .build())
          .build();
    }
  //在followUpRequest方法中判断了是否需要重定向以及是否需要重连，需要重连时会返回一个request,request为null说明不需要重连，则直接返回response
    Request followUp = followUpRequest(response);
  
    if (followUp == null) {
      if (!forWebSocket) {
        streamAllocation.release();
      }
      return response;
    }
  
    closeQuietly(response.body());
  
    if (++followUpCount > MAX_FOLLOW_UPS) {
      streamAllocation.release();
      throw new ProtocolException("Too many follow-up requests: " + followUpCount);
    }
  
    if (followUp.body() instanceof UnrepeatableRequestBody) {
      streamAllocation.release();
      throw new HttpRetryException("Cannot retry streamed HTTP body", response.code());
    }
  
    if (!sameConnection(response, followUp.url())) {
      streamAllocation.release();
      streamAllocation = new StreamAllocation(
          client.connectionPool(), createAddress(followUp.url()), callStackTrace);
    } else if (streamAllocation.codec() != null) {
      throw new IllegalStateException("Closing the body of " + response
          + " didn't close its backing stream. Bad interceptor?");
    }
  
    request = followUp;
    priorResponse = response;
  }
  ```
  * BridgeInterceptor

  ```java
  public Response intercept(Chain chain) throws IOException {
      Request userRequest = chain.request();
      Request.Builder requestBuilder = userRequest.newBuilder();
  RequestBody body = userRequest.body();
  if (body != null) {
    MediaType contentType = body.contentType();
    if (contentType != null) {
      requestBuilder.header("Content-Type", contentType.toString());
    }
  
    long contentLength = body.contentLength();
    if (contentLength != -1) {
      requestBuilder.header("Content-Length", Long.toString(contentLength));
      requestBuilder.removeHeader("Transfer-Encoding");
    } else {
      requestBuilder.header("Transfer-Encoding", "chunked");
      requestBuilder.removeHeader("Content-Length");
    }
  }
  
  if (userRequest.header("Host") == null) {
    requestBuilder.header("Host", hostHeader(userRequest.url(), false));
  }
  
  if (userRequest.header("Connection") == null) {
    requestBuilder.header("Connection", "Keep-Alive");
  }
  
  // 如果我们添加一个“accept-encoding:gzip”头字段，我们还负责解压缩传输流。
  boolean transparentGzip = false;
  if (userRequest.header("Accept-Encoding") == null && userRequest.header("Range") == null) {
    transparentGzip = true;
    requestBuilder.header("Accept-Encoding", "gzip");
  }
  
  List<Cookie> cookies = cookieJar.loadForRequest(userRequest.url());
  if (!cookies.isEmpty()) {
      //传进用户设置的 cookies
    requestBuilder.header("Cookie", cookieHeader(cookies));
  }
  
  if (userRequest.header("User-Agent") == null) {
    requestBuilder.header("User-Agent", Version.userAgent());
  }
  
      //上面添加完请求头信息后，就访问下一条链了
  Response networkResponse = chain.proceed(requestBuilder.build());
  //上面返回后，就拿到头部返回的cookie取出来存进本地 cookie 保存
  HttpHeaders.receiveHeaders(cookieJar, userRequest.url(), networkResponse.headers());
  //下面就开始组装 response 返回给上一层处理
  Response.Builder responseBuilder = networkResponse.newBuilder()
      .request(userRequest);
  
  if (transparentGzip
      && "gzip".equalsIgnoreCase(networkResponse.header("Content-Encoding"))
      && HttpHeaders.hasBody(networkResponse)) {
    GzipSource responseBody = new GzipSource(networkResponse.body().source());
    Headers strippedHeaders = networkResponse.headers().newBuilder()
        .removeAll("Content-Encoding")
        .removeAll("Content-Length")
        .build();
    responseBuilder.headers(strippedHeaders);
    responseBuilder.body(new RealResponseBody(strippedHeaders, Okio.buffer(responseBody)));
  }
  
  return responseBuilder.build();
  ```
  * BridgeInteceptor的Interceptor基本上都是添加请求的头信息，例如启动是否使用长连接Keep-Alive，设置Cookie，启动压缩与解压gzip等。

    ------

    

  * CacheInterceptor：缓存拦截器

  ```java
  public Response intercept(Chain chain) throws IOException {
      //先去缓存里按request取出对应response缓存
      Response cacheCandidate = cache != null
          ? cache.get(chain.request())
          : null;
          long now = System.currentTimeMillis();
  //再用缓存策略去处理
  CacheStrategy strategy = new CacheStrategy.Factory(now, chain.request(), cacheCandidate).get();
  Request networkRequest = strategy.networkRequest;
  Response cacheResponse = strategy.cacheResponse;
  
  if (cache != null) {
    cache.trackResponse(strategy);
  }
  
  if (cacheCandidate != null && cacheResponse == null) {
    closeQuietly(cacheCandidate.body()); // 缓存候选不适用。关闭它。
  }
  
  // 如果我们被禁止使用网络，并且缓存不足，则失败。
  if (networkRequest == null && cacheResponse == null) {
    return new Response.Builder()
        .request(chain.request())
        .protocol(Protocol.HTTP_1_1)
        .code(504)
        .message("Unsatisfiable Request (only-if-cached)")
        .body(Util.EMPTY_RESPONSE)
        .sentRequestAtMillis(-1L)
        .receivedResponseAtMillis(System.currentTimeMillis())
        .build();
  }
  
  // 没有网络，但是有缓存，就可以返回了缓存了，直接处理了，不再走下一个节点
  if (networkRequest == null) {
    return cacheResponse.newBuilder()
        .cacheResponse(stripBody(cacheResponse))
        .build();
  }
  
  Response networkResponse = null;
  try {
      //有网络无缓存就需要去网络访问了，到下一个节点去
    networkResponse = chain.proceed(networkRequest);
  } finally {
    if (networkResponse == null && cacheCandidate != null) {
      closeQuietly(cacheCandidate.body());
    }
  }
  
      //当之前是有缓存时
  if (cacheResponse != null) {
    if (networkResponse.code() == HTTP_NOT_MODIFIED) {//http_not_modified
      Response response = cacheResponse.newBuilder()
          .headers(combine(cacheResponse.headers(), networkResponse.headers()))
          .sentRequestAtMillis(networkResponse.sentRequestAtMillis())
          .receivedResponseAtMillis(networkResponse.receivedResponseAtMillis())
          .cacheResponse(stripBody(cacheResponse))
          .networkResponse(stripBody(networkResponse))
          .build();
      networkResponse.body().close();
  
      //在合并头之后但在剥离内容编码头之前更新缓存（由initcontentstream（）执行）
      cache.trackConditionalCacheHit();
        //更新了缓存cacheResponse
      cache.update(cacheResponse, response);
      return response;
    } else {
      closeQuietly(cacheResponse.body());
    }
  }
  //当之前没有缓存时
  Response response = networkResponse.newBuilder()
      .cacheResponse(stripBody(cacheResponse))
      .networkResponse(stripBody(networkResponse))
      .build();
  
  if (cache != null) {
    if (HttpHeaders.hasBody(response) && CacheStrategy.isCacheable(response, networkRequest)) {
      // Offer this request to the cache.
      CacheRequest cacheRequest = cache.put(response);
      return cacheWritingResponse(cacheRequest, response);
    }
  
    if (HttpMethod.invalidatesCache(networkRequest.method())) {
      try {
        cache.remove(networkRequest);
      } catch (IOException ignored) {
        // The cache cannot be written.
      }
    }
  }
  
  return response;
  ```
  * 首先根据 request 从 cache 中取 response
    将 request  和 respnse 传入 CacheStrategy 根据缓存策略（比如仅适用网络加载，仅适用缓存，缓存时效等）得到有策略处理后的 networkRequest 和 cacheResponse，根据处理后的 networkRequest 和 cacheResponse 来判断进行是否访问下一个节点的操作

    ------

    

  * ConnectInterceptor，到达这里就是准备访问网络了，

  ```java
  public Response intercept(Chain chain) throws IOException {
      RealInterceptorChain realChain = (RealInterceptorChain) chain;
      Request request = realChain.request();
      StreamAllocation streamAllocation = realChain.streamAllocation();
  
  boolean doExtensiveHealthChecks = !request.method().equals("GET");
  HttpCodec httpCodec = streamAllocation.newStream(client, doExtensiveHealthChecks);
  RealConnection connection = streamAllocation.connection();
  
  return realChain.proceed(request, streamAllocation, httpCodec, connection);
  }  
  ```
  * 主要为下一步最终进行网络请求做铺垫，这里获得了HttpCodec 和 RealConnection，然后将这些参数传入下一个Chain。
    在newStream方法中会去先尝试从RealConnectionPool中寻找已经存在的连接，若未命中则创建一个连接并与服务器握手对接。（也就是说在这个拦截器，先与服务器创建连接）
    在完成连接后会将Socket对象通过Okio封装成BufferedSource和BufferedSink，并将两者传入HttpCodec，在下一个拦截器网络请求时会用到。
    source = Okio.buffer(Okio.source(rawSocket));
    sink = Okio.buffer(Okio.sink(rawSocket));

    ------

    

  * CallServerInterceptor

  ```java
  public Response intercept(Chain chain) throws IOException {
      RealInterceptorChain realChain = (RealInterceptorChain) chain;
      HttpCodec httpCodec = realChain.httpStream();
      StreamAllocation streamAllocation = realChain.streamAllocation();
      RealConnection connection = (RealConnection) realChain.connection();
      Request request = realChain.request();
  long sentRequestMillis = System.currentTimeMillis();
      //这里很重要，这里将请求的请求头部分写进了httpCodec里的请求里面，先只拿着请求头去访问 
  httpCodec.writeRequestHeaders(request);
  
  Response.Builder responseBuilder = null;
  if (HttpMethod.permitsRequestBody(request.method()) && request.body() != null) {
    // 如果请求上有“expect:100 continue”头，请等待“http/1.1 100 continue”响应，然后再传输请求主体。如果我们没有得到响应，返回我们得到的内容（例如4xx响应），而不传输请求体（这里可能是先判断是否连接成功的，不然要是连接不成功，但是我们还把请求体发出去不是很浪费吗，还有一个可能是为了安全问题）
    if ("100-continue".equalsIgnoreCase(request.header("Expect"))) {
      httpCodec.flushRequest();
      responseBuilder = httpCodec.readResponseHeaders(true);
    }
  
    if (responseBuilder == null) {
      //如果满足“expect:100 continue”期望，接下来可以编写请求body了。
      Sink requestBodyOut = httpCodec.createRequestBody(request, request.body().contentLength());
      BufferedSink bufferedRequestBody = Okio.buffer(requestBodyOut);
      request.body().writeTo(bufferedRequestBody);
      bufferedRequestBody.close();
    } else if (!connection.isMultiplexed()) {
      //如果不满足“expect:100 continue”的要求，请防止重用HTTP/1连接。
      streamAllocation.noNewStreams();
    }
  }
  
  httpCodec.finishRequest();
  
  if (responseBuilder == null) {
    responseBuilder = httpCodec.readResponseHeaders(false);
  }
  
  Response response = responseBuilder
      .request(request)
      .handshake(streamAllocation.connection().handshake())
      .sentRequestAtMillis(sentRequestMillis)
      .receivedResponseAtMillis(System.currentTimeMillis())
      .build();
  
  int code = response.code();
  if (forWebSocket && code == 101) {
    // 连接正在升级，但我们需要确保拦截器看到非空的响应主体。
    response = response.newBuilder()
        .body(Util.EMPTY_RESPONSE)
        .build();
  } else {
    response = response.newBuilder()
        .body(httpCodec.openResponseBody(response))
        .build();
  }
  
  if ("close".equalsIgnoreCase(response.request().header("Connection"))
      || "close".equalsIgnoreCase(response.header("Connection"))) {
    streamAllocation.noNewStreams();
  }
  
  if ((code == 204 || code == 205) && response.body().contentLength() > 0) {
    throw new ProtocolException(
        "HTTP " + code + " had non-zero Content-Length: " + response.body().contentLength());
  }
  
  return response;
  ```
  * 可以发现具体的实现都交给了HttpCodec，它是对Http协议操作的一种抽象，针对HTTP/1.1与HTTP2有Http1Codec和Http2Codec两种实现。
  * 方法的命名都是read和write，因为HttpCodec中最后的请求和响应是由上一步封装的BufferedSource和BufferedSink来完成的，sink负责输出流，将写入的数据交由socket发出，source负责输入流，从socket中读取响应数据。

  ---------------------
  * 到达最后一个节点后就按责任链的顺序依次返回了，返回个 Response 对象，我们就可以拿到 Response 做一些操作了

## 异步回调

* 异步跟同步的主要区别是，异步我们需要给它设置一个回调后我们就不再理他，它会在线程池执行任务完后用回调我们的回调方法。

* 我们调用 RealCall 的 enqueue 方法

  ~~~java
  @Override public void enqueue(Callback responseCallback) {
      synchronized (this) {
        if (executed) throw new IllegalStateException("Already Executed");
        executed = true;
      }
      captureCallStackTrace();
      eventListener.callStart(this);
      client.dispatcher().enqueue(new AsyncCall(responseCallback));
    }
  ~~~

* 接下来到 调度器的 enqueue 方法，应该和同步操作一样只是把异步任务加进队列吧

  ~~~java
  synchronized void enqueue(AsyncCall call) {
      if (runningAsyncCalls.size() < maxRequests/*64*/ && runningCallsForHost(call) < maxRequestsPerHost/*5*/) {//为了避免太多请求同时执行,如果正在运行的异步任务数小于最大数量且此时的请求同一个服务器连接数低于5个（对于每一个主机host（服务器），同时请求数目不能超过maxRequestsPerHost），就直接加进正在运行的异步任务队列，然后直接丢进线程池里执行即可
        runningAsyncCalls.add(call);
        executorService().execute(call);
      } else {//否则就加进准备异步队列等待
        readyAsyncCalls.add(call);
      }
    }
  //线程池
  public synchronized ExecutorService executorService() {
      if (executorService == null) {
          //创建线程池，0个核心线程，不限制非核心线程，并发执行大量短期的小任务
        executorService = new ThreadPoolExecutor(0, Integer.MAX_VALUE, 60, TimeUnit.SECONDS,//不存储元素的阻塞队列 ,每个插入操作必须等到另一个线程调用移除操作，否则插入操作一直处于阻塞,这个队列的作用就是传递任务，并不会保存。
            new SynchronousQueue<Runnable>(), Util.threadFactory("OkHttp Dispatcher", false));
      }
      return executorService;
  }
  ~~~

* 丢到线程池里执行，执行的应该是 running 方法啊，我们去找到它

  ~~~java
  //继承了 NamedRunnable ，去里面找找
  final class AsyncCall extends NamedRunnable {
      private final Callback responseCallback;
  
    }
  ~~~

* NamedRunnable

  ~~~java
  public abstract class NamedRunnable implements Runnable {
    protected final String name;
  
    public NamedRunnable(String format, Object... args) {
      this.name = Util.format(format, args);
    }
    @Override public final void run() {
      String oldName = Thread.currentThread().getName();
      Thread.currentThread().setName(name);
      try {
          //这里就直接调用了 execute()，我们可以看到这里不是主线程执行，我们不可以这里操作UI，除非在 execute 里做了 handler 机制才有可能操作 UI，我们去看一下有没有
        execute();
      } finally {
        Thread.currentThread().setName(oldName);
      }
    }
  
    protected abstract void execute();
  }
  ~~~

* 现在就到 NamedRunnable 复写的 execute()

  ~~~java
  @Override protected void execute() {
      //可以看到下面并没有做 handler 机制来让用户复写的 回调执行在主线程里面，也就是说，如果用户要在回调里操作 UI，那么用户需要直接在回调的方法里使用 handler 机制。
        boolean signalledCallback = false;
        try {
            //可以看到和同步请求一样会走 getResponseWithInterceptorChain(); 也就是说会走 请求的拦截链，这里过程上面已经分析过了就不再分析
          Response response = getResponseWithInterceptorChain();
          if (retryAndFollowUpInterceptor.isCanceled()) {
            signalledCallback = true;
              //看这里是直接执行用户传进来的回调，这里可以想：并不是每个异步请求都是需要回到主线程操作 UI 的
            responseCallback.onFailure(RealCall.this, new IOException("Canceled"));
          } else {
            signalledCallback = true;
            responseCallback.onResponse(RealCall.this, response);
          }
        } catch (IOException e) {
          if (signalledCallback) {
            // Do not signal the callback twice!
            Platform.get().log(INFO, "Callback failure for " + toLoggableString(), e);
          } else {
            eventListener.callFailed(RealCall.this, e);
            responseCallback.onFailure(RealCall.this, e);
          }
        } finally {
            //请求最终结束后就去调度器里的对应队列移除掉这个请求，异步操作还有一个操作，就是继续去准备队列中取出任务到线程池里执行
          client.dispatcher().finished(this);
        }
  }
   /** Used by {@code AsyncCall#run} to signal completion. */
  void finished(AsyncCall call) {
      finished(runningAsyncCalls, call, true);
  }
  private <T> void finished(Deque<T> calls, T call, boolean promoteCalls) {
      int runningCallsCount;
      Runnable idleCallback;
      synchronized (this) {
        if (!calls.remove(call)) throw new AssertionError("Call wasn't in-flight!");
        if (promoteCalls) //此时为true
            promoteCalls();//我们去下面看一下这个方法
        runningCallsCount = runningCallsCount();
        idleCallback = this.idleCallback;
      }
  
      if (runningCallsCount == 0 && idleCallback != null) {
        idleCallback.run();
      }
  }
  private void promoteCalls() {
      if (runningAsyncCalls.size() >= maxRequests) return; // Already running max capacity.如果已经满负荷就自然退出
      if (readyAsyncCalls.isEmpty()) return; // No ready calls to promote.如果准备队列里已经没有别的请求了，那么自然就退出
      
  	//如果还没有超过最大请求数，准备队列也还有请求，那么自然就要取出异步请求来执行
      for (Iterator<AsyncCall> i = readyAsyncCalls.iterator(); i.hasNext(); ) {
        AsyncCall call = i.next();
  
        if (runningCallsForHost(call) < maxRequestsPerHost) {
          i.remove();
          runningAsyncCalls.add(call);
          executorService().execute(call);
        }
        if (runningAsyncCalls.size() >= maxRequests) return; // Reached max capacity.
      }
  }
  ~~~

  

## 总结

1. 我们不可以在异步请求的回调里操作 UI
2. 同步请求是需要我们自己创建线程去执行的
3. 同时异步请求数默认为 64 ,如果有需要是可以去修改这个值
4. 拦截链的部分是最主要的部分，封装了访问网络的操作，比如连接网络，缓存机制，失败重连各种