---
title: OkHttp全解析之Http & Websocket
date: 2023-02-15 19:38:26
tags:
- android
- okhttp
categories:
- android
- 网络
---



# Okhttp-http



## 基础信息

> - 依赖信息
>
>   ```groovy
>   implementation("com.squareup.okhttp3:okhttp:4.10.0")
>   ```
>
> - 作用
>
>   可用于http及websocket的Http Client请求框架



## 关键概念



> HTTP网络请求中涉及2个重要概念
>
> - Request
> - Response
>
> 即请求和响应

>Okhttp的设计上为了贴合现实的网络请求，规划了5个类
>
>- `OkHttpClient`
>
>  客户端实体，可以进行网络请求细节的配置
>
>- `Request`
>
>  客户端网络请求报文
>
>- `Response`
>
>  服务端网络请求响应报文
>
>- `Call`
>
>  HTTP请求
>
>- `WebSocket`
>
>  Websocket请求

> Okhttp做的很好的一点是使用者基本上只需要和上面几个类打交道
>
> - http请求
>
> ```kotlin
> val request = Request.Builder()
>     .url("https://www.example.com")
>     .headers(Headers.headersOf())
>     .method("get","This is body".toRequestBody())
>     .build()
> 
> val client = OkHttpClient()
> val call = client.newCall(request)
> val response = call.execute()
> ```
>
> - websocket
>
> ```kotlin
> val client = OkHttpClient()
> 
> val request = Request.Builder()
>     .url("ws://www.example.com")
>     .build()
> 
> val webSocket = client.newWebSocket(request, object : WebSocketListener() {
>     //...
> })
> 
> val isSuccess = webSocket.send("hello from client")
> ```



## Http



### Request 

> An HTTP request. Instances of this class are immutable if their body is null or itself immutable.
>
> 一个HTPP的请求，该实例是不可变的。

> 一个Request由5部分组成
>
> - url
>
>   网络请求的url值
>
> - method
>
>   网络请求方法类型
>
> - headers
>
>   网络请求head参数
>
> - body
>
>   网络请求体
>
> - tags
>
>   request上的标记信息，**不用于网络请求**。
>
> - cacheControl
>
>   网络请求缓存策略
>
> - isHttps
>
>   网络请求是否是https

```kotlin
class Request internal constructor(
  @get:JvmName("url") val url: HttpUrl,
  @get:JvmName("method") val method: String,
  @get:JvmName("headers") val headers: Headers,
  @get:JvmName("body") val body: RequestBody?,
  internal val tags: Map<Class<*>, Any>
) {
    val cacheControl: CacheControl
		get() {
          var result = lazyCacheControl
          if (result == null) {
            result = CacheControl.parse(headers)
            lazyCacheControl = result
          }
          return result
        }
    
  	val isHttps: Boolean
    	get() = url.isHttps
    
}
```



> Request的创建不支持直接new，由于需要的配置比较多，所以采用了Builder设计模式

```kotlin
Request.Builder()
    .url("url地址")
    .addHeader(name = "key", value = "value")
    .method(method = "get", body = null)
    .tag(type = String::class.java, tag = "tag")
    .cacheControl(
        CacheControl.Builder()
            .noCache()
            .build()
    )
```



#### url

> url的配置有三种方法

- way1

```kotlin
open fun url(url: String): Builder {
      // Silently replace web socket URLs with HTTP URLs.
      // 获取url
      val finalUrl: String = when {
          //websocket协议的建立需要先进行http请求协议升级。
        url.startsWith("ws:", ignoreCase = true) -> {
          "http:${url.substring(3)}"
        }
        url.startsWith("wss:", ignoreCase = true) -> {
          "https:${url.substring(4)}"
        }
        else -> url
      }
	  // 将url转化为HttpUrl类
      return url(finalUrl.toHttpUrl())
    }

//解析并构建
fun String.toHttpUrl(): HttpUrl = Builder().parse(null, this).build()
```

- way2

```kotlin
open fun url(url: URL) = url(url.toString().toHttpUrl())
```

- way3

```kotlin
open fun url(url: HttpUrl): Builder
```



#### header

> 能够对header操作的有如下

- way1

```kotlin
open fun addHeader(name: String, value: String) = apply {
  //internal var headers: Headers.Builder
  headers.add(name, value)
}

//headers
fun add(name: String, value: String) = apply {
      checkName(name)
      checkValue(value, name)
    //添加k-v
      addLenient(name, value)
}

internal fun addLenient(name: String, value: String) = apply {
    //internal val namesAndValues: MutableList<String> = ArrayList(20)
    //由于header的k是可以重复的，所以采用了一个数组装配
      namesAndValues.add(name)
      namesAndValues.add(value.trim())
}
```

- way2

```kotlin
open fun removeHeader(name: String) = apply {
  headers.removeAll(name)
}
// 移除所有k匹配的k-v键值对
fun removeAll(name: String) = apply {
      var i = 0
      while (i < namesAndValues.size) {
        if (name.equals(namesAndValues[i], ignoreCase = true)) {
          namesAndValues.removeAt(i) // name
          namesAndValues.removeAt(i) // value
          i -= 2
        }
        i += 2
      }
}
```

- way3

```kotlin
open fun header(name: String, value: String) = apply {
      headers[name] = value
}

// 将key对于的value设置为新的值
operator fun set(name: String, value: String) = apply {
      // 校验合法性
      checkName(name)
      checkValue(value, name)
      // 移除后再添加
      removeAll(name)
      addLenient(name, value)
}
```

- way4

```kotlin
// 前面的几个方法封装的已经很好了，基本上也用不上。
open fun headers(headers: Headers) = apply {
  this.headers = headers.newBuilder()
}
```



#### method

> 方法有两类，一类是安全的，内置的

- way1

```kotlin
 open fun get() = method("GET", null)

 open fun head() = method("HEAD", null)

 open fun post(body: RequestBody) = method("POST", body)

 @JvmOverloads
 open fun delete(body: RequestBody? = EMPTY_REQUEST) = method("DELETE", body)

 open fun put(body: RequestBody) = method("PUT", body)

 open fun patch(body: RequestBody) = method("PATCH", body)
```

- way2

```kotlin
open fun method(method: String, body: RequestBody?): Builder = apply {
  require(method.isNotEmpty()) {
    "method.isEmpty() == true"
  }
  if (body == null) {
    require(!HttpMethod.requiresRequestBody(method)) {
      "method $method must have a request body."
    }
  } else {
    require(HttpMethod.permitsRequestBody(method)) {
      "method $method must not have a request body."
    }
  }
  this.method = method
  this.body = body
}
```



#### body

> body即`RequestBody`

> 类关系图如下

![image-20230216170810826](https://typora-blog-picture.oss-cn-chengdu.aliyuncs.com/blog/image-20230216170810826.png)



> RequestBody

> 一个抽象模板类

```kotlin
abstract class RequestBody {

  /** 返回该body的content-type类型 */
  abstract fun contentType(): MediaType?

  /**
  	body的长度（单位为字节），如果不确定长度返回-1
   */
  @Throws(IOException::class)
  open fun contentLength(): Long = -1L

  /** 向body内写入内容（BufferedSink类似于OutputStream，着属于okio的内容了。） */
  @Throws(IOException::class)
  abstract fun writeTo(sink: BufferedSink)

  /**
	http是否是双工的。
	http/2以下都是半双工，只能由client向服务端发送，而不能由server主动向client发送。
	http/2以后加入了服务端推送的特性，可以实现服务端主动向客户端推送消息。
   */
  open fun isDuplex(): Boolean = false

  /**
	
   */
  open fun isOneShot(): Boolean = false

  // 创建responseBody的扩展方法
  // 其实也就是new一个匿名类返回。
  companion object {
    @JvmStatic
    @JvmName("create")
    fun String.toRequestBody(contentType: MediaType? = null): RequestBody {
      var charset: Charset = UTF_8
      var finalContentType: MediaType? = contentType
      if (contentType != null) {
        val resolvedCharset = contentType.charset()
        if (resolvedCharset == null) {
          charset = UTF_8
          finalContentType = "$contentType; charset=utf-8".toMediaTypeOrNull()
        } else {
          charset = resolvedCharset
        }
      }
      val bytes = toByteArray(charset)
      return bytes.toRequestBody(finalContentType, 0, bytes.size)
    }

    /** Returns a new request body that transmits this. */
    @JvmStatic
    @JvmName("create")
    fun ByteString.toRequestBody(contentType: MediaType? = null): RequestBody {
      return object : RequestBody() {
        override fun contentType() = contentType

        override fun contentLength() = size.toLong()

        override fun writeTo(sink: BufferedSink) {
          sink.write(this@toRequestBody)
        }
      }
    }

    /** Returns a new request body that transmits this. */
    @JvmOverloads
    @JvmStatic
    @JvmName("create")
    fun ByteArray.toRequestBody(
      contentType: MediaType? = null,
      offset: Int = 0,
      byteCount: Int = size
    ): RequestBody {
      checkOffsetAndCount(size.toLong(), offset.toLong(), byteCount.toLong())
      return object : RequestBody() {
        override fun contentType() = contentType

        override fun contentLength() = byteCount.toLong()

        override fun writeTo(sink: BufferedSink) {
          sink.write(this@toRequestBody, offset, byteCount)
        }
      }
    }

    /** Returns a new request body that transmits the content of this. */
    @JvmStatic
    @JvmName("create")
    fun File.asRequestBody(contentType: MediaType? = null): RequestBody {
      return object : RequestBody() {
        override fun contentType() = contentType

        override fun contentLength() = length()

        override fun writeTo(sink: BufferedSink) {
          source().use { source -> sink.writeAll(source) }
        }
      }
    }
   
  }
}
```



> FormBody

> 也是采用了Builder设计模式

> 只能通过builder才能进行类实例的创建

```kotlin
val formBody = FormBody.Builder(charset = UTF_8.INSTANCE)
    .add(name = "", value = "")
    .addEncoded(name = "", value = "")
    .build()
```



> 构造的时候可以指定一个Charset

> 再添加表单参数的时候可以选择使用编码对String进行编码

```kotlin
class Builder @JvmOverloads constructor(private val charset: Charset? = null) {
  private val names = mutableListOf<String>()
  private val values = mutableListOf<String>()

  fun add(name: String, value: String) = apply {
    names += name.canonicalize(
        encodeSet = FORM_ENCODE_SET,
        plusIsSpace = true,
        charset = charset
    )
    values += value.canonicalize(
        encodeSet = FORM_ENCODE_SET,
        plusIsSpace = true,
        charset = charset
    )
  }

  fun addEncoded(name: String, value: String) = apply {
    names += name.canonicalize(
        encodeSet = FORM_ENCODE_SET,
        alreadyEncoded = true,
        plusIsSpace = true,
        charset = charset
    )
    values += value.canonicalize(
        encodeSet = FORM_ENCODE_SET,
        alreadyEncoded = true,
        plusIsSpace = true,
        charset = charset
    )
  }

  fun build(): FormBody = FormBody(names, values)
}
```



> MultiPartBody

```kotlin
val multiPart = MultipartBody.Builder()
        .addPart("part1".toRequestBody())
        .addPart("part2".toRequestBody())
        .build()
```

> Builder内部封装了一个list，addPart就是往里找个集合里面塞东西

```kotlin
class Builder @JvmOverloads constructor(boundary: String = UUID.randomUUID().toString()) {
  private val boundary: ByteString = boundary.encodeUtf8()
  private var type = MIXED
  private val parts = mutableListOf<Part>()

  /**
   * Set the MIME type. Expected values for `type` are [MIXED] (the default), [ALTERNATIVE],
   * [DIGEST], [PARALLEL] and [FORM].
   */
  fun setType(type: MediaType) = apply {
    require(type.type == "multipart") { "multipart != $type" }
    this.type = type
  }

  /** Add a part to the body. */
  fun addPart(body: RequestBody) = apply {
    addPart(Part.create(body))
  }

  /** Add a part to the body. */
  fun addPart(headers: Headers?, body: RequestBody) = apply {
    addPart(Part.create(headers, body))
  }

  /** Add a form data part to the body. */
  fun addFormDataPart(name: String, value: String) = apply {
    addPart(Part.createFormData(name, value))
  }

  /** Add a form data part to the body. */
  fun addFormDataPart(name: String, filename: String?, body: RequestBody) = apply {
    addPart(Part.createFormData(name, filename, body))
  }

  /** Add a part to the body. */
  fun addPart(part: Part) = apply {
    parts += part
  }

  /** Assemble the specified parts into a request body. */
  fun build(): MultipartBody {
    check(parts.isNotEmpty()) { "Multipart body must have at least one part." }
    return MultipartBody(boundary, type, parts.toImmutableList())
  }
}
```



#### tag

> tag本质上就是一种标记，网络请求用不上，这个标记是用来标识request的。

> 在拦截器里面做处理的时候可能会依据不同的请求做特殊处理，这个tag可能就有用武之地了

```kotlin
open fun tag(tag: Any?): Builder = tag(Any::class.java, tag)

open fun <T> tag(type: Class<in T>, tag: T?) = apply {
  if (tag == null) {
    // internal var tags: MutableMap<Class<*>, Any> = mutableMapOf()
    tags.remove(type)
  } else {
    if (tags.isEmpty()) {
      tags = mutableMapOf()
    }
    tags[type] = type.cast(tag)!! // Force-unwrap due to lack of contracts on Class#cast()
  }
}
```



#### cache



> CacheControl是网络缓存的选项

> 而且它是通过Request的header指定的

> 它只是提供了更好的api支持

```kotlin
open fun cacheControl(cacheControl: CacheControl): Builder {
  val value = cacheControl.toString()
  return when {
    value.isEmpty() -> removeHeader("Cache-Control")
    else -> header("Cache-Control", value)
  }
}
```

> Cache-Control也只能通过使用Builder来实例化

```kotlin
CacheControl.Builder()
    .noCache()
    .noTransform()
    .build()
```

![image-20230218102959969](https://typora-blog-picture.oss-cn-chengdu.aliyuncs.com/blog/image-20230218102959969.png)







### Response

> Response可以通过Builder构建，但是一般是内部使用，或者测试Mock数据的时候

![image-20230218105712006](https://typora-blog-picture.oss-cn-chengdu.aliyuncs.com/blog/image-20230218105712006.png)



#### head

> 有三种方法（同Reqeust）

- way1

```kotlin
open fun addHeader(name: String, value: String)
```

- way2

```kotlin
open fun header(name: String, value: String)
```

- way3

```kotlin
open fun headers(headers: Headers)
```



#### status

- way1

> 设置status code整型数据

```kotlin
open fun code(code: Int)
```

- way2

> 设置消息，比如200，表示success，504 Gateway timeout等

```kotlin
open fun message(message: String)
```





#### body

> 即`ResponseBody`

> 类结构图如下

![image-20230218110829678](https://typora-blog-picture.oss-cn-chengdu.aliyuncs.com/blog/image-20230218110829678.png)

> `ResponseBody`

> 从使用上看到了`RequestBody`的影子，实际上原理也类似

```kotlin
val r = Response.Builder()
    .body("a".toResponseBody())
```

```kotlin
val r = Response.Builder() 
	.body(ByteArray(10).toResponseBody())
```



> `CahchedResponseBody`

> 其实也是io的包装，而而且还是private，说明是内部使用的。

```kotlin
private class CacheResponseBody(
  val snapshot: DiskLruCache.Snapshot,
  private val contentType: String?,
  private val contentLength: String?
) : ResponseBody() {
  private val bodySource: BufferedSource

  init {
    val source = snapshot.getSource(ENTRY_BODY)
    bodySource = object : ForwardingSource(source) {
      @Throws(IOException::class)
      override fun close() {
        snapshot.close()
        super.close()
      }
    }.buffer()
  }

  override fun contentType(): MediaType? = contentType?.toMediaTypeOrNull()

  override fun contentLength(): Long = contentLength?.toLongOrDefault(-1L) ?: -1L

  override fun source(): BufferedSource = bodySource
}
```

> `RealResponseBody`
>
> 查包名`okhttp3.internal.http`就知道是内部使用的，api相对来说没那么好用。

```kotlin
class RealResponseBody(
  /**
   * Use a string to avoid parsing the content type until needed. This also defers problems caused
   * by malformed content types.
   */
  private val contentTypeString: String?,
  private val contentLength: Long,
  private val source: BufferedSource
) : ResponseBody() {

  override fun contentLength(): Long = contentLength

  override fun contentType(): MediaType? = contentTypeString?.toMediaTypeOrNull()

  override fun source(): BufferedSource = source
}
```





#### 其他



- protocol

> 协议类型

```kotlin
open fun protocol(protocol: Protocol)
```

> 是个枚举类

```kotlin
enum class Protocol(private val protocol: String) {
 
  HTTP_1_0("http/1.0"),

  HTTP_1_1("http/1.1"),

  @Deprecated("OkHttp has dropped support for SPDY. Prefer {@link #HTTP_2}.")
  SPDY_3("spdy/3.1"),
 
  HTTP_2("h2"),

  H2_PRIOR_KNOWLEDGE("h2_prior_knowledge"),

  QUIC("quic");

  override fun toString() = protocol

  companion object {

    @JvmStatic
    @Throws(IOException::class)
    fun get(protocol: String): Protocol {
      // Unroll the loop over values() to save an allocation.
      @Suppress("DEPRECATION")
      return when (protocol) {
        HTTP_1_0.protocol -> HTTP_1_0
        HTTP_1_1.protocol -> HTTP_1_1
        H2_PRIOR_KNOWLEDGE.protocol -> H2_PRIOR_KNOWLEDGE
        HTTP_2.protocol -> HTTP_2
        SPDY_3.protocol -> SPDY_3
        QUIC.protocol -> QUIC
        else -> throw IOException("Unexpected protocol: $protocol")
      }
    }
  }
}
```

- handshake

> TLS握手的实体类

```kotlin
open fun handshake(handshake: Handshake?)
```

- networkResponse

> 网络请求获取的原始数据

```kotlin
open fun networkResponse(networkResponse: Response?)
```

- cacheResponse

> 本地缓存获取的原生数据

```kotlin
open fun cacheResponse(cacheResponse: Response?)
```

- priorResponse

> okhttp支持自动重定向，`priorResponse`就是它前一个重定向的请求

```kotlin
open fun priorResponse(priorResponse: Response?)
```

- sentRequestAtMillis

> 请求发送的时间

```kotlin
open fun sentRequestAtMillis(sentRequestAtMillis: Long)
```

- receivedResponseAtMillis

> 请求接受的时间

```kotlin
open fun receivedResponseAtMillis(receivedResponseAtMillis: Long)
```

- request

> 与response对应的wire-level（满足协议要求）的request

```kotlin
open fun request(request: Request)
```





### Call

> Call是即将执行的网络请求

```kotlin
interface Call : Cloneable {
  
  fun request(): Request

  @Throws(IOException::class)
  fun execute(): Response

  fun enqueue(responseCallback: Callback)

  fun cancel()

  fun isExecuted(): Boolean

  fun isCanceled(): Boolean

  fun timeout(): Timeout

  public override fun clone(): Call

  fun interface Factory {
    fun newCall(request: Request): Call
  }
}
```



> Call虽然是一个接口，但是只有一个实现类`RealCall`

```kotlin
class RealCall(
  val client: OkHttpClient,
  /** The application's original request unadulterated by redirects or auth headers. */
  val originalRequest: Request,
  val forWebSocket: Boolean
) 
```



#### timeout

> 获取超时计时器

```kotlin
private val timeout = object : AsyncTimeout() {
    override fun timedOut() {
      cancel()
    }
  }.apply {
    timeout(client.callTimeoutMillis.toLong(), MILLISECONDS)
  }

override fun timeout() = timeout
```



#### clone

> 直接返回结果

```kotlin
override fun clone() = RealCall(client, originalRequest, forWebSocket)
```



#### request

> 返回最新的请求

```kotlin
override fun request(): Request = originalRequest
```



#### cancel

> cancel提供了线程安全的特征，但是这个安全是**有限**

> 能保证线程**可见性**，**禁止指令重排**，但不能实现原子性即可能重复cancel。

> 可能为了保证性能吧，安全性越高，性能就越弱

```kotlin
@Volatile private var canceled = false

override fun cancel() {
  if (canceled) return // Already canceled.

  canceled = true
  exchange?.cancel()
  connectionToCancel?.cancel()

  eventListener.canceled(this)
}
```



#### isCanceld

```kotlin
override fun isCanceled() = canceled
```



#### isExecuted

> 不同于cancel，call必须保证只能执行一次，所以需要保证较强的安全水平。

```kotlin
private val executed = AtomicBoolean()

override fun isExecuted(): Boolean = executed.get()
```



#### execute

> 执行请求

```kotlin
override fun execute(): Response {
  //修改flag，cas确保一次执行
  check(executed.compareAndSet(false, true)) { "Already Executed" }
  //启动计时器，超时取消执行。
  timeout.enter()
  //告知call开始了
  callStart()
  try {
    //通知调度器，该call开始执行
    client.dispatcher.executed(this)
    //执行获取响应结果
    return getResponseWithInterceptorChain()
  } finally {
    //告知调度器该call已经结束。
    client.dispatcher.finished(this)
  }
}
```



> 处理请求

```kotlin
internal fun getResponseWithInterceptorChain(): Response {
    // 构建拦截器链
    val interceptors = mutableListOf<Interceptor>()
    //自定义拦截器
    interceptors += client.interceptors
    //预制拦截器
    interceptors += RetryAndFollowUpInterceptor(client)
    interceptors += BridgeInterceptor(client.cookieJar)
    interceptors += CacheInterceptor(client.cache)
    interceptors += ConnectInterceptor
    //自定义网络拦截器
    if (!forWebSocket) {
      interceptors += client.networkInterceptors
    }
    interceptors += CallServerInterceptor(forWebSocket)

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
      // 启动责任链处理
      val response = chain.proceed(originalRequest)
      // 取消处理
      if (isCanceled()) {
        response.closeQuietly()
        throw IOException("Canceled")
      }
      return response
    } catch (e: IOException) {
      // 异常处理
      calledNoMoreExchanges = true
      throw noMoreExchanges(e) as Throwable
    } finally {
      // 
      if (!calledNoMoreExchanges) {
        noMoreExchanges(null)
      }
    }
  }
```





#### enqueue

> 异步实现请求



```kotlin
override fun enqueue(responseCallback: Callback) {
  // 检验是否已经执行
  check(executed.compareAndSet(false, true)) { "Already Executed" }
	
  callStart()
  // 请求入队
  client.dispatcher.enqueue(AsyncCall(responseCallback))
}
```





### Client

> 客户端请求的实体类，可以用于做网络请求参数的配置





#### Interceptor

> 可以说式okhttp中核心的部分

> Interceptor分为三类
>
> - 内置拦截器
>
>   内部预置的拦截器
>
> - 网络拦截器
>
>   在内置拦截器之后起作用的拦截器
>
> - 普通拦截器
>
>   在内置拦截器之前起作用的拦截器



```kotlin
// Build a full stack of interceptors.
val interceptors = mutableListOf<Interceptor>()
//普通拦截器
interceptors += client.interceptors
// 内置拦截器
interceptors += RetryAndFollowUpInterceptor(client)
interceptors += BridgeInterceptor(client.cookieJar)
interceptors += CacheInterceptor(client.cache)
interceptors += ConnectInterceptor
if (!forWebSocket) {
  //网络拦截器
  interceptors += client.networkInterceptors
}
//内置拦截器
interceptors += CallServerInterceptor(forWebSocket)
```



##### 内置拦截器



###### RetryAndFollowUpInterceptor

> This interceptor recovers from failures and follows redirects as necessary. It may throw an IOException if the call was canceled.
>
> 此拦截器用于从失败的请求中回复并且在有必要的时候进行重定向，当请求取消时抛出IO异常。





> 类声明

```kotlin
class RetryAndFollowUpInterceptor(private val client: OkHttpClient) : Interceptor
```

> 总调度

```kotlin
override fun intercept(chain: Interceptor.Chain): Response {
  val realChain = chain as RealInterceptorChain
  var request = chain.request
  val call = realChain.call
  var followUpCount = 0
  var priorResponse: Response? = null
  var newExchangeFinder = true
  var recoveredFailures = listOf<IOException>()
  while (true) {
    // 为网络请求创建连接
    call.enterNetworkInterceptorExchange(request, newExchangeFinder)

    var response: Response
    var closeActiveExchange = true
    try {
      //处理取消
      if (call.isCanceled()) {
        throw IOException("Canceled")
      }

      try {
        // 向前推进
        response = realChain.proceed(request)
        // 时候开启新的连接
        newExchangeFinder = true
      } catch (e: RouteException) {
        // 尝试恢复连接
        if (!recover(e.lastConnectException, call, request, requestSendStarted = false)) {
          //恢复失败抛异常
          throw e.firstConnectException.withSuppressed(recoveredFailures)
        } else {
          recoveredFailures += e.firstConnectException
        }
        // 不需要重新建立连接
        newExchangeFinder = false
        continue
      } catch (e: IOException) {
        // An attempt to communicate with a server failed. The request may have been sent.
        if (!recover(e, call, request, requestSendStarted = e !is ConnectionShutdownException)) {
          throw e.withSuppressed(recoveredFailures)
        } else {
          recoveredFailures += e
        }
        newExchangeFinder = false
        continue
      }

      // Attach the prior response if it exists. Such responses never have a body.
      // 合并前一个连接
      if (priorResponse != null) {
        response = response.newBuilder()
            .priorResponse(priorResponse.newBuilder()
                .body(null)
                .build())
            .build()
      }

      val exchange = call.interceptorScopedExchange
      // 重定向request 
      val followUp = followUpRequest(response, exchange)

      if (followUp == null) {
        if (exchange != null && exchange.isDuplex) {
          call.timeoutEarlyExit()
        }
        closeActiveExchange = false
        return response
      }

      val followUpBody = followUp.body
      if (followUpBody != null && followUpBody.isOneShot()) {
        closeActiveExchange = false
        return response
      }

      response.body?.closeQuietly()
	
      // 限制重定向次数
      if (++followUpCount > MAX_FOLLOW_UPS) {
        throw ProtocolException("Too many follow-up requests: $followUpCount")
      }
	  // 更新重定向的请求地址
      request = followUp
      // 当前请求作为后一个
      priorResponse = response
    } finally {
      call.exitNetworkInterceptorExchange(closeActiveExchange)
    }
  }
}
```

> 重定向

```kotlin
private fun followUpRequest(userResponse: Response, exchange: Exchange?): Request? {
  val route = exchange?.connection?.route()
  val responseCode = userResponse.code

  val method = userResponse.request.method
  when (responseCode) {
    HTTP_PROXY_AUTH -> {
        // 代理身份验证
        // 407 
      val selectedProxy = route!!.proxy
      if (selectedProxy.type() != Proxy.Type.HTTP) {
        throw ProtocolException("Received HTTP_PROXY_AUTH (407) code while not using proxy")
      }
      return client.proxyAuthenticator.authenticate(route, userResponse)
    }

      //客户端身份验证
      // 401 
    HTTP_UNAUTHORIZED -> return client.authenticator.authenticate(route, userResponse)

      //重定向
      /// 30X
    HTTP_PERM_REDIRECT, HTTP_TEMP_REDIRECT, HTTP_MULT_CHOICE, HTTP_MOVED_PERM, HTTP_MOVED_TEMP, HTTP_SEE_OTHER -> {
      return buildRedirectRequest(userResponse, method)
    }

      //客户端超时 408
    HTTP_CLIENT_TIMEOUT -> {
      // 408's are rare in practice, but some servers like HAProxy use this response code. The
      // spec says that we may repeat the request without modifications. Modern browsers also
      // repeat the request (even non-idempotent ones.)
      if (!client.retryOnConnectionFailure) {
        // The application layer has directed us not to retry the request.
        return null
      }

      val requestBody = userResponse.request.body
      if (requestBody != null && requestBody.isOneShot()) {
        return null
      }
      val priorResponse = userResponse.priorResponse
      if (priorResponse != null && priorResponse.code == HTTP_CLIENT_TIMEOUT) {
        // We attempted to retry and got another timeout. Give up.
        return null
      }

      if (retryAfter(userResponse, 0) > 0) {
        return null
      }

      return userResponse.request
    }

      //服务不用
    HTTP_UNAVAILABLE -> {
      val priorResponse = userResponse.priorResponse
      if (priorResponse != null && priorResponse.code == HTTP_UNAVAILABLE) {
        // We attempted to retry and got another timeout. Give up.
        return null
      }

      if (retryAfter(userResponse, Integer.MAX_VALUE) == 0) {
        // specifically received an instruction to retry without delay
        return userResponse.request
      }

      return null
    }

      //There are too many connections from your internet address  421
    HTTP_MISDIRECTED_REQUEST -> {
      // OkHttp can coalesce HTTP/2 connections even if the domain names are different. See
      // RealConnection.isEligible(). If we attempted this and the server returned HTTP 421, then
      // we can retry on a different connection.
      val requestBody = userResponse.request.body
      if (requestBody != null && requestBody.isOneShot()) {
        return null
      }

      if (exchange == null || !exchange.isCoalescedConnection) {
        return null
      }

      exchange.connection.noCoalescedConnections()
      return userResponse.request
    }

    else -> return null
  }
}
```

> 该拦截器处理了如下status code，并在合适的时候进行retry
>
> 身份验证
>
> - 407
>
>   Proxy Authentication Required
>
> - 401
>
>   Unauthorized
>
> 重定向
>
> - 307/308
>
>   Temporary Redirect.
>
> - 300
>
>   Multiple Choices
>
> - 301
>
>   Moved Permanently.
>
> - 302
>
>   Temporary Redirect.
>
> - 303
>
>   See Other
>
> 其他
>
> - 408
>
>   Request timeout
>
> - 503
>
>   Service Unavailable.
>
> - 421
>
>   There are too many connections from your internet address





###### BridgeInterceptor

> 类型声明

> cookieJar用于管理cookie

```kotlin
class BridgeInterceptor(private val cookieJar: CookieJar) : Interceptor
```

> 核心实现



```kotlin
override fun intercept(chain: Interceptor.Chain): Response {
  // 获取请求
  val userRequest = chain.request()
  val requestBuilder = userRequest.newBuilder()
  // 获取body
  val body = userRequest.body
  if (body != null) {
    val contentType = body.contentType()
    if (contentType != null) { // content-type
      requestBuilder.header("Content-Type", contentType.toString())
    }

    val contentLength = body.contentLength()
    if (contentLength != -1L) { // 指定了content-length
      requestBuilder.header("Content-Length", contentLength.toString())
      requestBuilder.removeHeader("Transfer-Encoding") 
    } else { // 没有指定content-length，使用Transfer-Encoding： chunked
      requestBuilder.header("Transfer-Encoding", "chunked")
      requestBuilder.removeHeader("Content-Length")
    }
  }

  if (userRequest.header("Host") == null) { // 主机名
    requestBuilder.header("Host", userRequest.url.toHostHeader())
  }

  if (userRequest.header("Connection") == null) { // 长连接
    requestBuilder.header("Connection", "Keep-Alive")
  }

  // If we add an "Accept-Encoding: gzip" header field we're responsible for also decompressing
  // the transfer stream.
  var transparentGzip = false
  if (userRequest.header("Accept-Encoding") == null && userRequest.header("Range") == null) { // 没设置编码，并且不是分块传输，默认采用gzip
    transparentGzip = true
    requestBuilder.header("Accept-Encoding", "gzip")
  }
  // 价值cookie
  val cookies = cookieJar.loadForRequest(userRequest.url)
  if (cookies.isNotEmpty()) {
    requestBuilder.header("Cookie", cookieHeader(cookies))
  }
  // 默认ua
  // const val userAgent = "okhttp/${OkHttp.VERSION}"
  if (userRequest.header("User-Agent") == null) {
    requestBuilder.header("User-Agent", userAgent)
  }
  // 进行后续的传输
  val networkResponse = chain.proceed(requestBuilder.build())
  // 更新cookie
  cookieJar.receiveHeaders(userRequest.url, networkResponse.headers)
  // 重新配置response，关联request
  val responseBuilder = networkResponse.newBuilder()
      .request(userRequest)
  // gzip解码
  if (transparentGzip &&
      "gzip".equals(networkResponse.header("Content-Encoding"), ignoreCase = true) &&
      networkResponse.promisesBody()) {
    val responseBody = networkResponse.body
    if (responseBody != null) {
        // gzip 输出流包装
      val gzipSource = GzipSource(responseBody.source())
        // 移除Content-Encoding Content-Length
      val strippedHeaders = networkResponse.headers.newBuilder()
          .removeAll("Content-Encoding")
          .removeAll("Content-Length")
          .build()
        // 添加头部
      responseBuilder.headers(strippedHeaders)
        // 替换body
      val contentType = networkResponse.header("Content-Type")
      responseBuilder.body(RealResponseBody(contentType, -1L, gzipSource.buffer()))
    }
  }
  // 返回
  return responseBuilder.build()
}
```



> 该拦截器实现了
>
> - Content-Type/Content-Length/Transfer-Encoding/Host/Connection/Accept-Encoding/User-Agent等头部的校验
> - Cookie的添加和更新
> - 消息压缩（尽可能采用gzip压缩）
>
> 总的来说就是做请求的转接，将应用层的数据整合成符合http规范的报文。



###### CacheInterceptor

> 类型声明

> `Cache`用于做http缓存。

```kotlin
class CacheInterceptor(internal val cache: Cache?) : Interceptor
```





> 先从缓存中获取request
>
> 然后`CacheStrategy`会依据Request和CachedResponse判断缓存是否生效
>
> 若失效交由后续的拦截器处理，命中则直接返回。

```kotlin
override fun intercept(chain: Interceptor.Chain): Response {
  val call = chain.call()
    // 获取缓存
  val cacheCandidate = cache?.get(chain.request())

  val now = System.currentTimeMillis()
	// http 缓存策略
  val strategy = CacheStrategy.Factory(now, chain.request(), cacheCandidate).compute()
    // 网络请求
    //（如果为null，要么是缓存hit，要么是request only-if-cached但无缓存，如果不为null表明该请求可以用于之后的拦截器）
  val networkRequest = strategy.networkRequest
    // 缓存响应（如果为null表示无缓存，不为null表示有命中的缓存）
  val cacheResponse = strategy.cacheResponse
	// 记录缓存命中次数，validation次数
  cache?.trackResponse(strategy)
    // 时间监听器
  val listener = (call as? RealCall)?.eventListener ?: EventListener.NONE
  	// 关闭缓存body（缓存未命中）
  if (cacheCandidate != null && cacheResponse == null) {
    // The cache candidate wasn't applicable. Close it.
    cacheCandidate.body?.closeQuietly()
  }

  // If we're forbidden from using the network and the cache is insufficient, fail.
    // 要么是request only-if-cached但无缓存，返回异常报文
  if (networkRequest == null && cacheResponse == null) {
    return Response.Builder()
        .request(chain.request())
        .protocol(Protocol.HTTP_1_1)
        .code(HTTP_GATEWAY_TIMEOUT)
        .message("Unsatisfiable Request (only-if-cached)")
        .body(EMPTY_RESPONSE)
        .sentRequestAtMillis(-1L)
        .receivedResponseAtMillis(System.currentTimeMillis())
        .build().also {
          listener.satisfactionFailure(call, it)
        }
  }

  // If we don't need the network, we're done.
    // 缓存命中，使用cache，直接返回
  if (networkRequest == null) {
    return cacheResponse!!.newBuilder()
        .cacheResponse(stripBody(cacheResponse))
        .build().also {
          listener.cacheHit(call, it)
        }
  }

  if (cacheResponse != null) { // 通知监听器，缓存命中
    listener.cacheConditionalHit(call, cacheResponse)
  } else if (cache != null) { // 通知监听器缓存miss。
    listener.cacheMiss(call)
  }

  var networkResponse: Response? = null
  try { // 让后续拦截器进行处理
    networkResponse = chain.proceed(networkRequest)
  } finally {
    // If we're crashing on I/O or otherwise, don't leak the cache body.
    if (networkResponse == null && cacheCandidate != null) {
      cacheCandidate.body?.closeQuietly()
    }
  }

  // If we have a cache response too, then we're doing a conditional get.
    // 如果在缓存策略中有匹配的缓存。
  if (cacheResponse != null) {
      // validation结果为没变，使用缓存并更新缓存
    if (networkResponse?.code == HTTP_NOT_MODIFIED) {
      val response = cacheResponse.newBuilder()
          .headers(combine(cacheResponse.headers, networkResponse.headers))
          .sentRequestAtMillis(networkResponse.sentRequestAtMillis)
          .receivedResponseAtMillis(networkResponse.receivedResponseAtMillis)
          .cacheResponse(stripBody(cacheResponse))
          .networkResponse(stripBody(networkResponse))
          .build()

      networkResponse.body!!.close()

      // Update the cache after combining headers but before stripping the
      // Content-Encoding header (as performed by initContentStream()).
      cache!!.trackConditionalCacheHit()
      cache.update(cacheResponse, response)
      return response.also {
        listener.cacheHit(call, it)
      }
    } else { // validation发现资源发生变化，关闭流（以后也用不上了）
      cacheResponse.body?.closeQuietly()
    }
  }

  val response = networkResponse!!.newBuilder()
      .cacheResponse(stripBody(cacheResponse))
      .networkResponse(stripBody(networkResponse))
      .build()
	// 缓存非空，刷新缓存
  if (cache != null) {
      // 有body，并且可以缓存
    if (response.promisesBody() && CacheStrategy.isCacheable(response, networkRequest)) {
      // Offer this request to the cache.
        // 将内容放入缓存
      val cacheRequest = cache.put(response)
      return cacheWritingResponse(cacheRequest, response).also {
        if (cacheResponse != null) {
          // This will log a conditional cache miss only.
          listener.cacheMiss(call)
        }
      }
    }
		// 刷新缓存
    if (HttpMethod.invalidatesCache(networkRequest.method)) {
      try {
        cache.remove(networkRequest)
      } catch (_: IOException) {
        // The cache cannot be written.
      }
    }
  }

  return response
}
```



> 缓存策略

```kotlin
val strategy = CacheStrategy.Factory(now, chain.request(), cacheCandidate).compute()
```



```kotlin
fun compute(): CacheStrategy {
  val candidate = computeCandidate()

  // We're forbidden from using the network and the cache is insufficient.
    // 如果networkRequest不为空，但request中设置了only-if-cached策略
  if (candidate.networkRequest != null && request.cacheControl.onlyIfCached) {
    return CacheStrategy(null, null)
  }

  return candidate
}
```



```kotlin
private fun computeCandidate(): CacheStrategy {
  // No cached response.
    // 如果没有缓存
  if (cacheResponse == null) {
    return CacheStrategy(request, null)
  }

  // Drop the cached response if it's missing a required handshake.
  	// 如果请求式https，握手信息为null
  if (request.isHttps && cacheResponse.handshake == null) {
    return CacheStrategy(request, null)
  }

  // If this response shouldn't have been stored, it should never be used as a response source.
  // This check should be redundant as long as the persistence store is well-behaved and the
  // rules are constant.
    // 如果不可以缓存
  if (!isCacheable(cacheResponse, request)) {
    return CacheStrategy(request, null)
  }

  val requestCaching = request.cacheControl
    // 如果缓存策略式no-cache，或有缓存限定
  if (requestCaching.noCache || hasConditions(request)) {
    return CacheStrategy(request, null)
  }
	
  val responseCaching = cacheResponse.cacheControl
// 当前缓存的请求的存活时间
  val ageMillis = cacheResponseAge()
// 缓存的最大生存时间    
  var freshMillis = computeFreshnessLifetime()

  if (requestCaching.maxAgeSeconds != -1) {
    freshMillis = minOf(freshMillis, SECONDS.toMillis(requestCaching.maxAgeSeconds.toLong()))
  }

  var minFreshMillis: Long = 0
  if (requestCaching.minFreshSeconds != -1) {
    minFreshMillis = SECONDS.toMillis(requestCaching.minFreshSeconds.toLong())
  }

  var maxStaleMillis: Long = 0
  if (!responseCaching.mustRevalidate && requestCaching.maxStaleSeconds != -1) {
    maxStaleMillis = SECONDS.toMillis(requestCaching.maxStaleSeconds.toLong())
  }
// 没有设置no-cache，并且 age + minFresh < freshMills + maxStale (及缓存没有完全失效)
  if (!responseCaching.noCache && ageMillis + minFreshMillis < freshMillis + maxStaleMillis) {
    val builder = cacheResponse.newBuilder()
    if (ageMillis + minFreshMillis >= freshMillis) { // 缓存已经超过了max-age,但是还在maxStale范围内
      builder.addHeader("Warning", "110 HttpURLConnection \"Response is stale\"")
    }
    val oneDayMillis = 24 * 60 * 60 * 1000L
    if (ageMillis > oneDayMillis && isFreshnessLifetimeHeuristic()) { // 无限期缓存，并且缓存已经超过一天了
      builder.addHeader("Warning", "113 HttpURLConnection \"Heuristic expiration\"")
    }
    return CacheStrategy(null, builder.build())
  }

  // Find a condition to add to the request. If the condition is satisfied, the response body
  // will not be transmitted.
  val conditionName: String
  val conditionValue: String?
  when { // 如果有条件判断
    etag != null -> {
      conditionName = "If-None-Match"
      conditionValue = etag
    }

    lastModified != null -> {
      conditionName = "If-Modified-Since"
      conditionValue = lastModifiedString
    }

    servedDate != null -> {
      conditionName = "If-Modified-Since"
      conditionValue = servedDateString
    }
    // 没有条件判断，不考虑缓存
    else -> return CacheStrategy(request, null) // No condition! Make a regular request.
  }

    // 加入相关的条件首部，进行validation
  val conditionalRequestHeaders = request.headers.newBuilder()
  conditionalRequestHeaders.addLenient(conditionName, conditionValue!!)

  val conditionalRequest = request.newBuilder()
      .headers(conditionalRequestHeaders.build())
      .build()
  return CacheStrategy(conditionalRequest, cacheResponse)
}
```



> 简单来说就是http缓存的封装。
>
> 严格遵循RFC



###### ConnectInterceptor

> 连接拦截器

```kotlin
object ConnectInterceptor : Interceptor {
  @Throws(IOException::class)
  override fun intercept(chain: Interceptor.Chain): Response {
      // 获取chain
    val realChain = chain as RealInterceptorChain
     // 初始化Exchange
    val exchange = realChain.call.initExchange(chain)
     // 将连接接入chain
    val connectedChain = realChain.copy(exchange = exchange)
      //  交由后续拦截器操作
    return connectedChain.proceed(realChain.request)
  }
}
```



> 初始化连接
>
> Finds a new or pooled connection to carry a forthcoming request and response
>
> 从连接池中寻找一个可复用的连接，或者是创建一个新的连接

```kotlin
internal fun initExchange(chain: RealInterceptorChain): Exchange {
  synchronized(this) {
    check(expectMoreExchanges) { "released" }
    check(!responseBodyOpen)
    check(!requestBodyOpen)
  }

  val exchangeFinder = this.exchangeFinder!!
  // 寻找网络编码
  val codec = exchangeFinder.find(client, chain)
  // 创建Exchange对象
  val result = Exchange(this, eventListener, exchangeFinder, codec)
  this.interceptorScopedExchange = result
  this.exchange = result
  synchronized(this) {
    this.requestBodyOpen = true
    this.responseBodyOpen = true
  }

  if (canceled) throw IOException("Canceled")
  return result
}
```



> 获取网络编码实体

```kotlin
fun find(
  client: OkHttpClient,
  chain: RealInterceptorChain
): ExchangeCodec {
  try {
      // 寻找可用连接
    val resultConnection = findHealthyConnection(
        connectTimeout = chain.connectTimeoutMillis,
        readTimeout = chain.readTimeoutMillis,
        writeTimeout = chain.writeTimeoutMillis,
        pingIntervalMillis = client.pingIntervalMillis,
        connectionRetryEnabled = client.retryOnConnectionFailure,
        doExtensiveHealthChecks = chain.request.method != "GET"
    )
      // 创建网络编码
    return resultConnection.newCodec(client, chain)
  } catch (e: RouteException) {
    trackFailure(e.lastConnectException)
    throw e
  } catch (e: IOException) {
    trackFailure(e)
    throw RouteException(e)
  }
}
```



> 寻找连接

```kotlin
private fun findHealthyConnection(
  connectTimeout: Int,
  readTimeout: Int,
  writeTimeout: Int,
  pingIntervalMillis: Int,
  connectionRetryEnabled: Boolean,
  doExtensiveHealthChecks: Boolean
): RealConnection {
  while (true) {
      // 寻找连接
    val candidate = findConnection(
        connectTimeout = connectTimeout,
        readTimeout = readTimeout,
        writeTimeout = writeTimeout,
        pingIntervalMillis = pingIntervalMillis,
        connectionRetryEnabled = connectionRetryEnabled
    )

      // 如果连接可用，直接返回
    // Confirm that the connection is good.
    if (candidate.isHealthy(doExtensiveHealthChecks)) {
      return candidate
    }
	// 如果连接不可用，不允许进一步运行
    // If it isn't, take it out of the pool.
    candidate.noNewExchanges()

    // Make sure we have some routes left to try. One example where we may exhaust all the routes
    // would happen if we made a new connection and it immediately is detected as unhealthy.
      // 如果连接不可用，还有尝试的连接，继续寻找
    if (nextRouteToTry != null) continue
		// 如果还存在路由项继续
    val routesLeft = routeSelection?.hasNext() ?: true
    if (routesLeft) continue

    val routesSelectionLeft = routeSelector?.hasNext() ?: true
    if (routesSelectionLeft) continue

    throw IOException("exhausted all routes")
  }
}
```



> 连接search核心逻辑

```kotlin
private fun findConnection(
  connectTimeout: Int,
  readTimeout: Int,
  writeTimeout: Int,
  pingIntervalMillis: Int,
  connectionRetryEnabled: Boolean
): RealConnection {
    // 确保call是有效的
  if (call.isCanceled()) throw IOException("Canceled")

  // Attempt to reuse the connection from the call.
  val callConnection = call.connection // This may be mutated by releaseConnectionNoEvents()!
    // 尝试复用连接
  if (callConnection != null) {
    var toClose: Socket? = null
    synchronized(callConnection) {
      if (callConnection.noNewExchanges || !sameHostAndPort(callConnection.route().address.url)) {
        toClose = call.releaseConnectionNoEvents()
      }
    }
	
    // If the call's connection wasn't released, reuse it. We don't call connectionAcquired() here
    // because we already acquired it.
    if (call.connection != null) {
      check(toClose == null)
      return callConnection
    }

    // The call's connection was released.
    toClose?.closeQuietly()
    eventListener.connectionReleased(call, callConnection)
  }

  // We need a new connection. Give it fresh stats.
  refusedStreamCount = 0
  connectionShutdownCount = 0
  otherFailureCount = 0
	// 尝试从连接池中获取
  // Attempt to get a connection from the pool.
  if (connectionPool.callAcquirePooledConnection(address, call, null, false)) {
    val result = call.connection!!
    eventListener.connectionAcquired(call, result)
    return result
  }

  // Nothing in the pool. Figure out what route we'll try next.
  val routes: List<Route>?
  val route: Route
    // 从其他路由项中寻找
  if (nextRouteToTry != null) {
    // Use a route from a preceding coalesced connection.
    routes = null
    route = nextRouteToTry!!
    nextRouteToTry = null
  } else if (routeSelection != null && routeSelection!!.hasNext()) {
    // Use a route from an existing route selection.
    routes = null
    route = routeSelection!!.next()
  } else {
    // Compute a new route selection. This is a blocking operation!
    var localRouteSelector = routeSelector
    if (localRouteSelector == null) {
      localRouteSelector = RouteSelector(address, call.client.routeDatabase, call, eventListener)
      this.routeSelector = localRouteSelector
    }
    val localRouteSelection = localRouteSelector.next()
    routeSelection = localRouteSelection
    routes = localRouteSelection.routes

    if (call.isCanceled()) throw IOException("Canceled")

    // Now that we have a set of IP addresses, make another attempt at getting a connection from
    // the pool. We have a better chance of matching thanks to connection coalescing.
      // 加入路由进行连接的复用
    if (connectionPool.callAcquirePooledConnection(address, call, routes, false)) {
      val result = call.connection!!
      eventListener.connectionAcquired(call, result)
      return result
    }

    route = localRouteSelection.next()
  }
	// 创建连接
  // Connect. Tell the call about the connecting call so async cancels work.
    // 创建新的连接
  val newConnection = RealConnection(connectionPool, route)
  call.connectionToCancel = newConnection
  try {
    newConnection.connect(
        connectTimeout,
        readTimeout,
        writeTimeout,
        pingIntervalMillis,
        connectionRetryEnabled,
        call,
        eventListener
    )
  } finally {
    call.connectionToCancel = null
  }
  call.client.routeDatabase.connected(newConnection.route())

  // If we raced another call connecting to this host, coalesce the connections. This makes for 3
  // different lookups in the connection pool!
    // 获取多路复用的连接即http/2连接。
  if (connectionPool.callAcquirePooledConnection(address, call, routes, true)) {
      // 获取到了就放弃创建的连接
    val result = call.connection!!
    nextRouteToTry = route
    newConnection.socket().closeQuietly()
    eventListener.connectionAcquired(call, result)
    return result
  }
	// 将新创建的连接放入连接池
  synchronized(newConnection) {
    connectionPool.put(newConnection)
    call.acquireConnectionNoEvents(newConnection)
  }

  eventListener.connectionAcquired(call, newConnection)
  return newConnection
}
```



###### CallServerInterceptor



> This is the last interceptor in the chain. It makes a network call to the server.
>
> 拦截器链中的最后一个，可以用于向server做网络请求。

```kotlin
class CallServerInterceptor(private val forWebSocket: Boolean) : Interceptor {

  @Throws(IOException::class)
  override fun intercept(chain: Interceptor.Chain): Response {
      // 获取必要参数
    val realChain = chain as RealInterceptorChain
    val exchange = realChain.exchange!!
    val request = realChain.request
    val requestBody = request.body
      // request发送时间
    val sentRequestMillis = System.currentTimeMillis()
	  // 写入头
    exchange.writeRequestHeaders(request)
	
    var invokeStartEvent = true
    var responseBuilder: Response.Builder? = null
        // 如果请求类型支持body，并且body不为null，写入body
    if (HttpMethod.permitsRequestBody(request.method) && requestBody != null) {
      // If there's a "Expect: 100-continue" header on the request, wait for a "HTTP/1.1 100
      // Continue" response before transmitting the request body. If we don't get that, return
      // what we did get (such as a 4xx response) without ever transmitting the request body.
      if ("100-continue".equals(request.header("Expect"), ignoreCase = true)) {
        exchange.flushRequest()
        responseBuilder = exchange.readResponseHeaders(expectContinue = true)
        exchange.responseHeadersStart()
        invokeStartEvent = false
      }
        // 写入body
      if (responseBuilder == null) {
        if (requestBody.isDuplex()) { // 如果是request是全双工，不关闭body
          // Prepare a duplex body so that the application can send a request body later.
          exchange.flushRequest()
          val bufferedRequestBody = exchange.createRequestBody(request, true).buffer()
          requestBody.writeTo(bufferedRequestBody)
        } else { // 不是全双工在发送数据后关闭
          // Write the request body if the "Expect: 100-continue" expectation was met.
          val bufferedRequestBody = exchange.createRequestBody(request, false).buffer()
          requestBody.writeTo(bufferedRequestBody)
          bufferedRequestBody.close()
        }
      } else { 
        exchange.noRequestBody()
        if (!exchange.connection.isMultiplexed) {
          // If the "Expect: 100-continue" expectation wasn't met, prevent the HTTP/1 connection
          // from being reused. Otherwise we're still obligated to transmit the request body to
          // leave the connection in a consistent state.
          exchange.noNewExchangesOnConnection()
        }
      }
    } else { // 无body	
      exchange.noRequestBody()
    }
	// 如果body为null或者不是全双工，请求完成
    if (requestBody == null || !requestBody.isDuplex()) {
      exchange.finishRequest()
    }
      // 构建response
    if (responseBuilder == null) {
        // 读响应头
      responseBuilder = exchange.readResponseHeaders(expectContinue = false)!!
      if (invokeStartEvent) { 
        exchange.responseHeadersStart()
        invokeStartEvent = false
      }
    }
      // 装入request，握手，发送时间，接受时间。
    var response = responseBuilder
        .request(request)
        .handshake(exchange.connection.handshake())
        .sentRequestAtMillis(sentRequestMillis)
        .receivedResponseAtMillis(System.currentTimeMillis())
        .build()
      // 读取响应码
    var code = response.code
    if (code == 100) {
      // Server sent a 100-continue even though we did not request one. Try again to read the actual
      // response status.
      responseBuilder = exchange.readResponseHeaders(expectContinue = false)!!
      if (invokeStartEvent) {
        exchange.responseHeadersStart()
      }
        // 重新构建response
      response = responseBuilder
          .request(request)
          .handshake(exchange.connection.handshake())
          .sentRequestAtMillis(sentRequestMillis)
          .receivedResponseAtMillis(System.currentTimeMillis())
          .build()
      code = response.code
    }

    exchange.responseHeadersEnd(response)
	// websocket协议
    response = if (forWebSocket && code == 101) {
      // Connection is upgrading, but we need to ensure interceptors see a non-null response body.
      response.newBuilder()
          .body(EMPTY_RESPONSE)
          .build()
    } else {
        // 读取body
      response.newBuilder()
          .body(exchange.openResponseBody(response))
          .build()
    }
      // 如果有request或者response中有明确声明Connection: close,不再复用连接
    if ("close".equals(response.request.header("Connection"), ignoreCase = true) ||
        "close".equals(response.header("Connection"), ignoreCase = true)) {
      exchange.noNewExchangesOnConnection()
    }
      // 请求码表示无body，但实际有
    if ((code == 204 || code == 205) && response.body?.contentLength() ?: -1L > 0L) {
      throw ProtocolException(
          "HTTP $code had non-zero Content-Length: ${response.body?.contentLength()}")
    }
      // 返回response
    return response
  }
}
```



##### 普通拦截器

> 属于自定义的拦截器

OkHttpClient.Builder

```kotlin
inline fun addInterceptor(crossinline block: (chain: Interceptor.Chain) -> Response) =
    addInterceptor(Interceptor { chain -> block(chain) })
```

```kotlin
fun addInterceptor(interceptor: Interceptor) = apply {
  interceptors += interceptor
}
```



##### 网络拦截器

> 也是属于自定义的拦截器

OkHttpClient.Builder

```kotlin
fun addNetworkInterceptor(interceptor: Interceptor) = apply {
  networkInterceptors += interceptor
}

@JvmName("-addNetworkInterceptor") // Prefix with '-' to prevent ambiguous overloads from Java.
inline fun addNetworkInterceptor(crossinline block: (chain: Interceptor.Chain) -> Response) =
    addNetworkInterceptor(Interceptor { chain -> block(chain) })
```





##### 小结

> 拦截器分为
>
> - 内置拦截器
>
>   完成基本的网络请求功能
>
> - 自定义拦截器
>
>   完成网络请求的扩展需求
>
> 其中自定义拦截器分为
>
> - 普通拦截器
>
>   最早一批处理网络请求的拦截器，在所有内置拦截器之前
>
> - 网络拦截器
>
>   在网络请求发送以前最晚处理的拦截器
>
> 区别不是很大，只是拦截器加入时间的不同。
>
> ```kotlin
> val interceptors = mutableListOf<Interceptor>()
> // 普通拦截器
> interceptors += client.interceptors
> // 内置拦截器
> interceptors += RetryAndFollowUpInterceptor(client)
> interceptors += BridgeInterceptor(client.cookieJar)
> interceptors += CacheInterceptor(client.cache)
> interceptors += ConnectInterceptor
> if (!forWebSocket) {
>     // 网络拦截器
>   interceptors += client.networkInterceptors
> }
> // 内置拦截器
> interceptors += CallServerInterceptor(forWebSocket)
> ```



#### Cache

> 默认情况下Cache对象为null

![image-20230227164826341](https://typora-blog-picture.oss-cn-chengdu.aliyuncs.com/blog/image-20230227164826341.png)



> 回过头来看CacheInterceptor会发现其实没有缓存

> 由于cache为null不会获取到任何缓存

![image-20230227165239908](https://typora-blog-picture.oss-cn-chengdu.aliyuncs.com/blog/image-20230227165239908.png)



##### 类结构

![image-20230227165526085](https://typora-blog-picture.oss-cn-chengdu.aliyuncs.com/blog/image-20230227165526085.png)



##### 构造函数

> 指定缓存存放路径，以及大小（字节）

```kotlin
// 主构造函数为internal，外部无法使用
constructor(directory: File, maxSize: Long) : this(directory, maxSize, FileSystem.SYSTEM)
```



##### 静态工具方法



> key
>
> 根据url获取计算得到key值

```kotlin
@JvmStatic
fun key(url: HttpUrl): String = url.toString().encodeUtf8().md5().hex()
```



> varyMatches
>
> 查看缓存的response的Vary Header是否发生变化

```kotlin
fun varyMatches(
  cachedResponse: Response,
  cachedRequest: Headers,
  newRequest: Request
): Boolean {
    // 获取缓存响应的header参数的Vary头
  return cachedResponse.headers.varyFields().none { // 对Vary头中指定的header参数进行一一比对
      // 如果任何一个不等就返回false
    cachedRequest.values(it) != newRequest.headers(it)
  }
}
```



##### 成员变量



```kotlin
// 存储数据的类
internal val cache = DiskLruCache(
      fileSystem = fileSystem,
      directory = directory,
      appVersion = VERSION,
      valueCount = ENTRY_COUNT,
      maxSize = maxSize,
      taskRunner = TaskRunner.INSTANCE
  )


// 用于数据统计
// 写入成功次数
internal var writeSuccessCount = 0
// 写入失败次数
internal var writeAbortCount = 0
// 网络请求次数
private var networkCount = 0
// 缓存命中次数
private var hitCount = 0
// 请求次数
private var requestCount = 0
```





##### 成员方法



> get
>
> 从cache中获取response

```kotlin
internal fun get(request: Request): Response? {
    // 根据url获取key
  val key = key(request.url)
    // 获取快照
  val snapshot: DiskLruCache.Snapshot = try {
    cache[key] ?: return null
  } catch (_: IOException) {
    return null // Give up because the cache cannot be read.
  }
	// 获取文件
  val entry: Entry = try {
    Entry(snapshot.getSource(ENTRY_METADATA))
  } catch (_: IOException) {
    snapshot.closeQuietly()
    return null
  }

  val response = entry.response(snapshot)
  if (!entry.matches(request, response)) {
    response.body?.closeQuietly()
    return null
  }

  return response
}
```



> Entry

```kotlin
@Throws(IOException::class) constructor(rawSource: Source) {
  rawSource.use {
    val source = rawSource.buffer()
      // 读取url
    val urlLine = source.readUtf8LineStrict()
    // Choice here is between failing with a correct RuntimeException
    // or mostly silently with an IOException
    url = urlLine.toHttpUrlOrNull() ?: throw IOException("Cache corruption for $urlLine").also {
      Platform.get().log("cache corruption", Platform.WARN, it)
    }
      // 读取方法
    requestMethod = source.readUtf8LineStrict()
    val varyHeadersBuilder = Headers.Builder()
      // 读取请求vary header个数
    val varyRequestHeaderLineCount = readInt(source)
      // 添加vary header
    for (i in 0 until varyRequestHeaderLineCount) {
      varyHeadersBuilder.addLenient(source.readUtf8LineStrict())
    }
    varyHeaders = varyHeadersBuilder.build()
		// 读取状态行
    val statusLine = StatusLine.parse(source.readUtf8LineStrict())
    protocol = statusLine.protocol
    code = statusLine.code
    message = statusLine.message
    val responseHeadersBuilder = Headers.Builder()
      // 读取状态行个数
    val responseHeaderLineCount = readInt(source)
    for (i in 0 until responseHeaderLineCount) {
      responseHeadersBuilder.addLenient(source.readUtf8LineStrict())
    }
      // 发送时间
    val sendRequestMillisString = responseHeadersBuilder[SENT_MILLIS]
      // 接受时间
    val receivedResponseMillisString = responseHeadersBuilder[RECEIVED_MILLIS]
    responseHeadersBuilder.removeAll(SENT_MILLIS)
    responseHeadersBuilder.removeAll(RECEIVED_MILLIS)
    sentRequestMillis = sendRequestMillisString?.toLong() ?: 0L
    receivedResponseMillis = receivedResponseMillisString?.toLong() ?: 0L
    responseHeaders = responseHeadersBuilder.build()

    if (isHttps) {
      val blank = source.readUtf8LineStrict() // 空白
      if (blank.isNotEmpty()) {
        throw IOException("expected \"\" but was \"$blank\"")
      }
        // 加密套件类型
      val cipherSuiteString = source.readUtf8LineStrict()
      val cipherSuite = CipherSuite.forJavaName(cipherSuiteString)
        // 服务端证书
      val peerCertificates = readCertificateList(source)
        // 本地证书
      val localCertificates = readCertificateList(source)
      val tlsVersion = if (!source.exhausted()) {
        TlsVersion.forJavaName(source.readUtf8LineStrict()) // 读取tls版本号
      } else {
        TlsVersion.SSL_3_0
      }
      handshake = Handshake.get(tlsVersion, cipherSuite, peerCertificates, localCertificates)
    } else {
      handshake = null
    }
  }
}
```



![image-20230227183310336](https://typora-blog-picture.oss-cn-chengdu.aliyuncs.com/blog/image-20230227183310336.png)







> 向cache中存入内容

> - 只写入了header
> - 返回了一个可以用于写入body的输出流（也不完全是输出流，可以看成是输出流）

> 所以body的写入需要借助外界通过向这个put返回的输出流里面写东西才行

```kotlin
internal fun put(response: Response): CacheRequest? {
    // 获取method
  val requestMethod = response.request.method
	// 判断是否需要刷新缓存
  if (HttpMethod.invalidatesCache(response.request.method)) {
    try {
      remove(response.request)
    } catch (_: IOException) {
      // The cache cannot be written.
    }
    return null
  }
	// 只允许缓存get请求
  if (requestMethod != "GET") {
    // Don't cache non-GET responses. We're technically allowed to cache HEAD requests and some
    // POST requests, but the complexity of doing so is high and the benefit is low.
    return null
  }
	// 如果header有Vary: *，拒绝缓存
  if (response.hasVaryAll()) {
    return null
  }
	// 创建一个Entry
    // 注意Entry之解析了response中的header
  val entry = Entry(response)
  var editor: DiskLruCache.Editor? = null
  try {
      // 获取一个cache修改器
    editor = cache.edit(key(response.request.url)) ?: return null
      // 写入entry（也就是header）
    entry.writeTo(editor)
      // 返回一个CacheRequest（可以理解body的输出流）
      // 注意前面的writeTo只写入了header，到目前为止body是没有写入的
    return RealCacheRequest(editor)
  } catch (_: IOException) {
    abortQuietly(editor)
    return null
  }
}
```





> 写入body

> `CacheInterceptor.intercept`

```java
override fun intercept(chain: Interceptor.Chain): Response {

    // ......
    
    if (cache != null) {
      if (response.promisesBody() && CacheStrategy.isCacheable(response, networkRequest)) {
        // Offer this request to the cache.
          // 存入缓存
        val cacheRequest = cache.put(response)
            // 返回响应体
        return cacheWritingResponse(cacheRequest, response).also {
          if (cacheResponse != null) {
            // This will log a conditional cache miss only.
            listener.cacheMiss(call)
          }
        }
      }

      if (HttpMethod.invalidatesCache(networkRequest.method)) {
        try {
          cache.remove(networkRequest)
        } catch (_: IOException) {
          // The cache cannot be written.
        }
      }
    }

    return response
  }
```

> 查看源码后可以发现Cache body的缓存是在Reponse读取的时候一并操作的，当用户读取Reponse的同时还会向缓存写入。

```kotlin
private fun cacheWritingResponse(cacheRequest: CacheRequest?, response: Response): Response {
  // Some apps return a null body; for compatibility we treat that like a null cache request.
  if (cacheRequest == null) return response
    // 获取cache Sink
  val cacheBodyUnbuffered = cacheRequest.body()
	// 获取网络请求reponse source
  val source = response.body!!.source()
    // cache sink加buffer，减少与磁盘交互次数
  val cacheBody = cacheBodyUnbuffered.buffer()
	// 返回给用户的response的body的source
  val cacheWritingSource = object : Source {
    private var cacheRequestClosed = false

    @Throws(IOException::class)
    override fun read(sink: Buffer, byteCount: Long): Long {
      val bytesRead: Long
      try {
          // 很正常的读取操作
          // 读取网络请求的response
        bytesRead = source.read(sink, byteCount)
      } catch (e: IOException) {
        if (!cacheRequestClosed) {
          cacheRequestClosed = true
          cacheRequest.abort() // Failed to write a complete cache response.
        }
        throw e
      }

      if (bytesRead == -1L) { // 读取完毕
        if (!cacheRequestClosed) {
          cacheRequestClosed = true
            // 关闭cache的sink
          cacheBody.close() // The cache response is complete!
        }
        return -1
      }
		// 将读取的内容写入缓存。
      sink.copyTo(cacheBody.buffer, sink.size - bytesRead, bytesRead)
      cacheBody.emitCompleteSegments()
      return bytesRead
    }

    override fun timeout() = source.timeout()

    @Throws(IOException::class)
    override fun close() {
      if (!cacheRequestClosed &&
          !discard(ExchangeCodec.DISCARD_STREAM_TIMEOUT_MILLIS, MILLISECONDS)) {
        cacheRequestClosed = true
        cacheRequest.abort()
      }
      source.close()
    }
  }

  val contentType = response.header("Content-Type")
  val contentLength = response.body.contentLength()
  return response.newBuilder()
      .body(RealResponseBody(contentType, contentLength, cacheWritingSource.buffer()))
      .build()
}
```





> remove

```kotlin
@Throws(IOException::class)
internal fun remove(request: Request) {
  cache.remove(key(request.url))
}
```



> initialize

```kotlin
fun initialize() {
  cache.initialize()
}
```





> 关闭缓存并删除所有存储的内容，值得注意的是他会删除缓存文件夹下的所有的文件，即使是不归缓存管的文件。

```kotlin
fun delete() {
  cache.delete()
}
```



> 删除缓存中的所有值，但是正在写入的内容会正常结束，但是不会存入缓存中

```kotlin
fun evictAll() {
  cache.evictAll()
}
```



> 返回缓存中的所有的url值

```kotlin
fun urls(): MutableIterator<String> {
  return object : MutableIterator<String> {
    private val delegate: MutableIterator<DiskLruCache.Snapshot> = cache.snapshots()
    private var nextUrl: String? = null
    private var canRemove = false

    override fun hasNext(): Boolean {
      if (nextUrl != null) return true

      canRemove = false // Prevent delegate.remove() on the wrong item!
      while (delegate.hasNext()) {
        try {
          delegate.next().use { snapshot ->
            val metadata = snapshot.getSource(ENTRY_METADATA).buffer()
            nextUrl = metadata.readUtf8LineStrict()
            return true
          }
        } catch (_: IOException) {
          // We couldn't read the metadata for this snapshot; possibly because the host filesystem
          // has disappeared! Skip it.
        }
      }

      return false
    }

    override fun next(): String {
      if (!hasNext()) throw NoSuchElementException()
      val result = nextUrl!!
      nextUrl = null
      canRemove = true
      return result
    }

    override fun remove() {
      check(canRemove) { "remove() before next()" }
      delegate.remove()
    }
  }
}
```



> 数据统计

```kotlin
@Synchronized fun writeAbortCount(): Int = writeAbortCount

@Synchronized fun writeSuccessCount(): Int = writeSuccessCount

@Throws(IOException::class)
fun size(): Long = cache.size()

/** Max size of the cache (in bytes). */
fun maxSize(): Long = cache.maxSize
```



> 刷新缓存

```kotlin
@Throws(IOException::class)
override fun flush() {
  cache.flush()
}
```



> 关闭

```kotlin
@Throws(IOException::class)
override fun close() {
  cache.close()
}
```



##### 小结



- `Cache`大多数的成员都是internal，无法访问，外部能做的要么是获取缓存的统计数据，或者删除缓存等操作，缓存的处理`CacheInterceptor`包揽。

- `Cache`的包含两部分，一为journal文件即缓存的控制文件，二为Response缓存

  - journal文件包含了缓存的内容（也就是缓存的变动历史）
  - Response缓存是以`url`的hash值作为文件名称，并将Response的内容拆分成了两部分，XXX.0与XXX.1文件。其中.0文件是Respone的header，.1文件为Response的body。

- 缓存文件的存入过程如下，先生产XXX.0.tmp，和XXX.1.tmp文件，.0.tmp文件是在put的时候立即生成的，.1.tmp文件是在网络Response读取的时候写入的。由于.1.tmp在.0.tmp之后写入完成。在.1.tmp写入完成以后才会把.0.tmp以及.1.tmp文件命名为.0，.1文件，这样做是为了确保缓存放入的原子性。（防止内容写入一半）

  ```kotlin
  // 在调用string以后response body不仅会被read，背后可能还会将reponse的body存入缓存中。
  okhttpClient.newCall(
      Request.Builder()
          .get()
          .url("your url")
          .build()
  ).execute().body?.string()
  ```





#### Authenticator

> 对身份鉴别过程进行处理

`RetryAndFollowUpInterceptor`

> followUpRequest方法会返回一个重尝试的Request，而就在这个时候会使用到Authenticator

```kotlin
private fun followUpRequest(userResponse: Response, exchange: Exchange?): Request? {
  val route = exchange?.connection?.route()
  val responseCode = userResponse.code
		// ......
  val method = userResponse.request.method
  when (responseCode) {
      // http代理鉴别
    HTTP_PROXY_AUTH -> {
      val selectedProxy = route!!.proxy
      if (selectedProxy.type() != Proxy.Type.HTTP) {
        throw ProtocolException("Received HTTP_PROXY_AUTH (407) code while not using proxy")
      }
      return client.proxyAuthenticator.authenticate(route, userResponse)
    }
		// http鉴别
    HTTP_UNAUTHORIZED -> return client.authenticator.authenticate(route, userResponse)
		// ......
    else -> return null
  }
}
```



> Authenticator接口

```kotlin
fun interface Authenticator {
  /**
   * Returns a request that includes a credential to satisfy an authentication challenge in
   * [response]. Returns null if the challenge cannot be satisfied.
   *
   * The route is best effort, it currently may not always be provided even when logically
   * available. It may also not be provided when an authenticator is re-used manually in an
   * application interceptor, such as when implementing client-specific retries.
   */
    // 
  @Throws(IOException::class)
  fun authenticate(route: Route?, response: Response): Request?

  companion object {
    /** An authenticator that knows no credentials and makes no attempt to authenticate. */
      // 一个什么都不做的鉴别器
    @JvmField
    val NONE: Authenticator = AuthenticatorNone()
    private class AuthenticatorNone : Authenticator {
      override fun authenticate(route: Route?, response: Response): Request? = null
    }

    /** An authenticator that uses the java.net.Authenticator global authenticator. */
      // 基于密码的身份鉴别器
    @JvmField
    val JAVA_NET_AUTHENTICATOR: Authenticator = JavaNetAuthenticator()
  }
}
```



> 设置鉴别器

```kotlin
fun authenticator(authenticator: Authenticator) = apply {
  this.authenticator = authenticator
}
```

```kotlin
fun proxyAuthenticator(proxyAuthenticator: Authenticator) = apply {
  if (proxyAuthenticator != this.proxyAuthenticator) {
    this.routeDatabase = null
  }

  this.proxyAuthenticator = proxyAuthenticator
}
```





#### ConnectionPool

> okhttp的连接池，内部管理了所有的http连接，连接管理都是这个类完成的。



> Manages reuse of HTTP and HTTP/2 connections for reduced network latency. HTTP requests that share the same Address may share a Connection. This class implements the policy of which connections to keep open for future use.
>
> 管理复用http和http/2的连接以减少网络传输的时间，http的请求如果使用了相同的`Address`则可能会共享一个`Connection`，此类实现了将开放的连接保存以备后续之需

```kotlin
class ConnectionPool internal constructor(
  internal val delegate: RealConnectionPool
)
```



##### 类结构

> 三个构造函数，一个成员变量，3个成员方法

![image-20230303110235780](https://typora-blog-picture.oss-cn-chengdu.aliyuncs.com/blog/image-20230303110235780.png)



> 构造函数

```kotlin
// 主构造
class ConnectionPool internal constructor(
  internal val delegate: RealConnectionPool
)

//从构造
constructor(
    maxIdleConnections: Int,
    keepAliveDuration: Long,
    timeUnit: TimeUnit
  ) : this(RealConnectionPool(
      taskRunner = TaskRunner.INSTANCE,
      maxIdleConnections = maxIdleConnections,
      keepAliveDuration = keepAliveDuration,
      timeUnit = timeUnit
  ))

 constructor() : this(5, 5, TimeUnit.MINUTES)
```

> 成员方法

> 肉眼可见的都是委托的代理类

```kotlin
// 获取池子里闲置的连接数
fun idleConnectionCount(): Int = delegate.idleConnectionCount()

// 获取连接池的连接总数
fun connectionCount(): Int = delegate.connectionCount()

// 关闭移除所有的空闲连接
fun evictAll() {
  delegate.evictAll()
}
```



##### RealConnectionPool

> 类结构

> 一个构造构造函数，5个成员变量，8个成员函数

![image-20230303111213005](https://typora-blog-picture.oss-cn-chengdu.aliyuncs.com/blog/image-20230303111213005.png)



> 成员变量

```kotlin
class RealConnectionPool(
  taskRunner: TaskRunner,
  // 最大空闲连接数
  private val maxIdleConnections: Int,
  keepAliveDuration: Long,
  timeUnit: TimeUnit
) {
  // 存活的最长nanoSecond时间
  private val keepAliveDurationNs: Long = timeUnit.toNanos(keepAliveDuration)
  // 用于存放cleaner请求的队列
  private val cleanupQueue: TaskQueue = taskRunner.newQueue()
  // cleaner
  private val cleanupTask = object : Task("$okHttpName ConnectionPool") {
    override fun runOnce() = cleanup(System.nanoTime())
  }
    
  // 所有连接
  private val connections = ConcurrentLinkedQueue<RealConnection>()
    
}
```



> 成员方法

```kotlin
// 以下三个是为ConnectionPool实现的
// connection的calls非空表明此connection处于active状态
fun idleConnectionCount(): Int {
  return connections.count {
    synchronized(it) { it.calls.isEmpty() }
  }
}
// connections是存放connection的容器，所有的connection存放在这里
fun connectionCount(): Int {
  return connections.size
}
// 关闭所有的空闲连接
fun evictAll() {
    val i = connections.iterator()
    while (i.hasNext()) {
      val connection = i.next()
      val socketToClose = synchronized(connection) {
          // 如果是空闲连接
        if (connection.calls.isEmpty()) {
            // 移除
          i.remove()
            // 设置该连接为不可使用
          connection.noNewExchanges = true
            // 获取socket
          return@synchronized connection.socket()
        } else {
          return@synchronized null
        }
      }
        // 关闭socket
      socketToClose?.closeQuietly()
    }
	// 如果connections空了，没执行任何clean操作，直接取消所有的cleaner
    if (connections.isEmpty()) cleanupQueue.cancelAll()
  }


// 存放入连接
  fun put(connection: RealConnection) {
    	// 查看调用的线程是否持有connection的对象锁，保证线程安全
      connection.assertThreadHoldsLock()
	// 添加connection到线程池
    connections.add(connection)
      // 为线程池规划一次清除任务
    cleanupQueue.schedule(cleanupTask)
  }

```

> 尝试获取一个连接

```kotlin
fun callAcquirePooledConnection(
    address: Address,
    call: RealCall,
    routes: List<Route>?,
    requireMultiplexed: Boolean
  ): Boolean {
    // 遍历所有连接
    for (connection in connections) {
      synchronized(connection) {
          // 如果需要http/2连接，connection必须要满足多路复用
        if (requireMultiplexed && !connection.isMultiplexed) return@synchronized
          // 查看连接是否是可以复用的
        if (!connection.isEligible(address, routes)) return@synchronized
          // 获取连接成功，并不发送任何的事件通知（eventListener）
        call.acquireConnectionNoEvents(connection)
        return true
      }
    }
    // 所有连接的遍历完了，没有发现符合要求的。
    return false
  }
```

> Notify this pool that connection has become idle. Returns true if the connection has been removed from the pool and should be closed.
>
> 提示连接池当前连接已经处于空闲状态，返回true表示connection已从连接池中移除，应该关闭当前连接

```kotlin
fun connectionBecameIdle(connection: RealConnection): Boolean {
  connection.assertThreadHoldsLock()
	// 如果显式声明连接关闭或者连接池的最大空闲数目为0 -> 移除连接
  return if (connection.noNewExchanges || maxIdleConnections == 0) {
    connection.noNewExchanges = true
    connections.remove(connection)
    if (connections.isEmpty()) cleanupQueue.cancelAll()
    true
  } else {
      // 告知连接池进行清理等操作。
    cleanupQueue.schedule(cleanupTask)
    false
  }
}
```

> 清理连接

```kotlin
fun cleanup(now: Long): Long {
  var inUseConnectionCount = 0
  var idleConnectionCount = 0
  var longestIdleConnection: RealConnection? = null
  var longestIdleDurationNs = Long.MIN_VALUE

  // Find either a connection to evict, or the time that the next eviction is due.
  for (connection in connections) {
    synchronized(connection) {
      // If the connection is in use, keep searching.
        // 获取此连接的call个数，以及空闲时间
      if (pruneAndGetAllocationCount(connection, now) > 0) {
        inUseConnectionCount++ // 累计正在使用的连接数
      } else {
        idleConnectionCount++ // 累计空闲连接数

        // If the connection is ready to be evicted, we're done.
        // 计算空闲时间段
        val idleDurationNs = now - connection.idleAtNs
        // 计算空闲的最大时间段
        if (idleDurationNs > longestIdleDurationNs) {
          longestIdleDurationNs = idleDurationNs
          longestIdleConnection = connection
        } else {
          Unit
        }
      }
    }
  }

  when {
      // 如果空闲的最大时间段大于目前设置的连接存活时间，或者空闲连接数超出最大值
      // 立即开启调度
    longestIdleDurationNs >= this.keepAliveDurationNs
        || idleConnectionCount > this.maxIdleConnections -> {
      // We've chosen a connection to evict. Confirm it's still okay to be evict, then close it.
      val connection = longestIdleConnection!!
      synchronized(connection) {
          // 如果当前连接不再空闲亦或不再是空闲最久的连接
        if (connection.calls.isNotEmpty()) return 0L // No longer idle.
        if (connection.idleAtNs + longestIdleDurationNs != now) return 0L // No longer oldest.
          // 如果是空闲连接，并且还是空闲最久的连接，从连接池中释放该连接
        connection.noNewExchanges = true
        connections.remove(longestIdleConnection)
      }
		// 关闭socket
      connection.socket().closeQuietly()
      if (connections.isEmpty()) cleanupQueue.cancelAll()

      // Clean up again immediately.
      return 0L
    }
		// 如果有空闲连接
    idleConnectionCount > 0 -> {
      // A connection will be ready to evict soon.
        // 计算下一次调度时间=存活时间-最长空闲时间段
      return keepAliveDurationNs - longestIdleDurationNs
    }
	 // 如果有存活连接
    inUseConnectionCount > 0 -> {
      // All connections are in use. It'll be at least the keep alive duration 'til we run
      // again.
        // 没有空闲连接，下次调度只能等最大生存时间
      return keepAliveDurationNs
    }
	// 没有连接，不需要调度
    else -> {
      // No connections, idle or in use.
      return -1
    }
  }
}
```





##### 连接创建

`ExchangeFinder.kt`

```kotlin
// Connect. Tell the call about the connecting call so async cancels work.
val newConnection = RealConnection(connectionPool, route)
call.connectionToCancel = newConnection
try {
  newConnection.connect(
      connectTimeout,
      readTimeout,
      writeTimeout,
      pingIntervalMillis,
      connectionRetryEnabled,
      call,
      eventListener
  )
} finally {
  call.connectionToCancel = null
}
```

> 连接

```kotlin
fun connect(
  connectTimeout: Int,
  readTimeout: Int,
  writeTimeout: Int,
  pingIntervalMillis: Int,
  connectionRetryEnabled: Boolean,
  call: Call,
  eventListener: EventListener
) {
 	// ......

  while (true) {
    try {
        // tunnel即使用http对https进行代理
      if (route.requiresTunnel()) {
        connectTunnel(connectTimeout, readTimeout, writeTimeout, call, eventListener)
        if (rawSocket == null) {
          // We were unable to connect the tunnel but properly closed down our resources.
          break
        }
      } else {
          // 连接到socket
          // new Socket() -> connect
        connectSocket(connectTimeout, readTimeout, call, eventListener)
      }
        // 根据协议类型建立连接
        // ssl握手，http/1.1 http/2
      establishProtocol(connectionSpecSelector, pingIntervalMillis, call, eventListener)
      eventListener.connectEnd(call, route.socketAddress, route.proxy, protocol)
      break
    } catch (e: IOException) {
      // ......
    }
  }

//......

  idleAtNs = System.nanoTime()
}
```



> Socket建立

```kotlin
private fun connectSocket(
  connectTimeout: Int,
  readTimeout: Int,
  call: Call,
  eventListener: EventListener
) {
  val proxy = route.proxy
  val address = route.address
	// 创建Socket
  val rawSocket = when (proxy.type()) {
    Proxy.Type.DIRECT, Proxy.Type.HTTP -> address.socketFactory.createSocket()!!
    else -> Socket(proxy)
  }
  this.rawSocket = rawSocket

  eventListener.connectStart(call, route.socketAddress, proxy)
  rawSocket.soTimeout = readTimeout
  try {
      // 连接
      // 即 socket.connect(address, connectTimeout)
    Platform.get().connectSocket(rawSocket, route.socketAddress, connectTimeout)
  } catch (e: ConnectException) {
 		//......
  }

    // ......
}
```



> tls，http协议连接创建

```kotlin
private fun establishProtocol(
  connectionSpecSelector: ConnectionSpecSelector,
  pingIntervalMillis: Int,
  call: Call,
  eventListener: EventListener
) {
    // 如果没有声明tls连接
  if (route.address.sslSocketFactory == null) {
      // Cleartext HTTP/2 with no "upgrade" round trip.
    if (Protocol.H2_PRIOR_KNOWLEDGE in route.address.protocols) { // 非tls的http2连接
      socket = rawSocket
      protocol = Protocol.H2_PRIOR_KNOWLEDGE
      startHttp2(pingIntervalMillis)
      return
    }
	// http 1.1连接
    socket = rawSocket
    protocol = Protocol.HTTP_1_1
    return
  }
	// tls + http2连接
  eventListener.secureConnectStart(call)
  connectTls(connectionSpecSelector)
  eventListener.secureConnectEnd(call, handshake)

  if (protocol === Protocol.HTTP_2) {
    startHttp2(pingIntervalMillis)
  }
}
```



> http2连接创建

[http2连接](https://halfrost.com/http2_begin/)

```kotlin
private fun startHttp2(pingIntervalMillis: Int) {
  val socket = this.socket!!
  val source = this.source!!
  val sink = this.sink!!
  socket.soTimeout = 0 // HTTP/2 connection timeouts are set per-stream.
    // 参数配置
  val http2Connection = Http2Connection.Builder(client = true, taskRunner = TaskRunner.INSTANCE)
      .socket(socket, route.address.url.host, source, sink)
      .listener(this)
      .pingIntervalMillis(pingIntervalMillis)
      .build()
  this.http2Connection = http2Connection
  this.allocationLimit = Http2Connection.DEFAULT_SETTINGS.getMaxConcurrentStreams()
    // 开启连接
  http2Connection.start()
}
```

> 写入连接序言和settings帧可能会写入window update帧

```kotlin
fun start(sendConnectionPreface: Boolean = true, taskRunner: TaskRunner = TaskRunner.INSTANCE) {
  if (sendConnectionPreface) {
      // 写入连接序言 
      // "PRI * HTTP/2.0\r\n\r\nSM\r\n\r\n"
    writer.connectionPreface()
      // 写入setting帧
    writer.settings(okHttpSettings)
    val windowSize = okHttpSettings.initialWindowSize
    if (windowSize != DEFAULT_INITIAL_WINDOW_SIZE) {
        // 写入update帧
      writer.windowUpdate(0, (windowSize - DEFAULT_INITIAL_WINDOW_SIZE).toLong())
    }
  }
  // Thread doesn't use client Dispatcher, since it is scoped potentially across clients via
  // ConnectionPool.
  taskRunner.newQueue().execute(name = connectionName, block = readerRunnable)
}
```

> ssl握手

```kotlin
private fun connectTls(connectionSpecSelector: ConnectionSpecSelector) {
  val address = route.address
  val sslSocketFactory = address.sslSocketFactory
  var success = false
  var sslSocket: SSLSocket? = null
  try {
    // Create the wrapper over the connected socket.
      // 创建ssl socket
    sslSocket = sslSocketFactory!!.createSocket(
        rawSocket, address.url.host, address.url.port, true /* autoClose */) as SSLSocket

  
      // 配置socket加密套件，tls版本，扩展（alpn）
    val connectionSpec = connectionSpecSelector.configureSecureSocket(sslSocket)
    if (connectionSpec.supportsTlsExtensions) {
      Platform.get().configureTlsExtensions(sslSocket, address.url.host, address.protocols)
    }

      // 握手
    sslSocket.startHandshake()
    // block for session establishment
    val sslSocketSession = sslSocket.session
    val unverifiedHandshake = sslSocketSession.handshake()

    // Verify that the socket's certificates are acceptable for the target host.
    if (!address.hostnameVerifier!!.verify(address.url.host, sslSocketSession)) {
      val peerCertificates = unverifiedHandshake.peerCertificates
      if (peerCertificates.isNotEmpty()) {
        val cert = peerCertificates[0] as X509Certificate
        throw SSLPeerUnverifiedException("""
            |Hostname ${address.url.host} not verified:
            |    certificate: ${CertificatePinner.pin(cert)}
            |    DN: ${cert.subjectDN.name}
            |    subjectAltNames: ${OkHostnameVerifier.allSubjectAltNames(cert)}
            """.trimMargin())
      } else {
        throw SSLPeerUnverifiedException(
            "Hostname ${address.url.host} not verified (no certificates)")
      }
    }

    val certificatePinner = address.certificatePinner!!
      
    handshake = Handshake(unverifiedHandshake.tlsVersion, unverifiedHandshake.cipherSuite,
        unverifiedHandshake.localCertificates) {
      certificatePinner.certificateChainCleaner!!.clean(unverifiedHandshake.peerCertificates,
          address.url.host)
    }

    // Check that the certificate pinner is satisfied by the certificates presented.
    certificatePinner.check(address.url.host) {
      handshake!!.peerCertificates.map { it as X509Certificate }
    }

    // Success! Save the handshake and the ALPN protocol.
    val maybeProtocol = if (connectionSpec.supportsTlsExtensions) {
      Platform.get().getSelectedProtocol(sslSocket)
    } else {
      null
    }
    socket = sslSocket
    source = sslSocket.source().buffer()
    sink = sslSocket.sink().buffer()
    protocol = if (maybeProtocol != null) Protocol.get(maybeProtocol) else Protocol.HTTP_1_1
    success = true
  } finally {
    if (sslSocket != null) {
      Platform.get().afterHandshake(sslSocket)
    }
    if (!success) {
      sslSocket?.closeQuietly()
    }
  }
}
```





##### 连接复用



`ExchangeFinder.kt`

```kotlin
private fun findConnection(
  connectTimeout: Int,
  readTimeout: Int,
  writeTimeout: Int,
  pingIntervalMillis: Int,
  connectionRetryEnabled: Boolean
): RealConnection {

  // Attempt to reuse the connection from the call.
    // 
  val callConnection = call.connection // This may be mutated by releaseConnectionNoEvents()!
  if (callConnection != null) {
    
  	// .....
      // 如果call 本身含有connection，并且非空直接返回
    if (call.connection != null) {
      check(toClose == null)
      return callConnection
    }
 
      // .....
  }

  // We need a new connection. Give it fresh stats.
  refusedStreamCount = 0
  connectionShutdownCount = 0
  otherFailureCount = 0

    // 尝试从连接池中获取一个连接
  if (connectionPool.callAcquirePooledConnection(address, call, null, false)) {
    val result = call.connection!!
    eventListener.connectionAcquired(call, result)
    return result
  }

    // 加入proxy，selector获取连接
  val routes: List<Route>?
  val route: Route
  if (nextRouteToTry != null) {
    // Use a route from a preceding coalesced connection.
    routes = null
    route = nextRouteToTry!!
    nextRouteToTry = null
  } else if (routeSelection != null && routeSelection!!.hasNext()) {
    // Use a route from an existing route selection.
    routes = null
    route = routeSelection!!.next()
  } else {
    // Compute a new route selection. This is a blocking operation!
    var localRouteSelector = routeSelector
    if (localRouteSelector == null) {
      localRouteSelector = RouteSelector(address, call.client.routeDatabase, call, eventListener)
      this.routeSelector = localRouteSelector
    }
    val localRouteSelection = localRouteSelector.next()
    routeSelection = localRouteSelection
    routes = localRouteSelection.routes

    if (call.isCanceled()) throw IOException("Canceled")

    // 加入proxy等获取连接
    if (connectionPool.callAcquirePooledConnection(address, call, routes, false)) {
      val result = call.connection!!
      eventListener.connectionAcquired(call, result)
      return result
    }

    route = localRouteSelection.next()
  }

  // 创建新的连接
  val newConnection = RealConnection(connectionPool, route)
  call.connectionToCancel = newConnection
  try {
    newConnection.connect(
        connectTimeout,
        readTimeout,
        writeTimeout,
        pingIntervalMillis,
        connectionRetryEnabled,
        call,
        eventListener
    )
  } finally {
    call.connectionToCancel = null
  }
  call.client.routeDatabase.connected(newConnection.route())

  
    // 获取多路复用连接
  if (connectionPool.callAcquirePooledConnection(address, call, routes, true)) {
    val result = call.connection!!
    nextRouteToTry = route
    newConnection.socket().closeQuietly()
    eventListener.connectionAcquired(call, result)
    return result
  }

    // 如果多路复用的连接没有获取到
    // 将创建的连接存放如连接池
  synchronized(newConnection) {
    connectionPool.put(newConnection)
    call.acquireConnectionNoEvents(newConnection)
  }

  eventListener.connectionAcquired(call, newConnection)
  return newConnection
}
```



> 连接池的复用分为3次
>
> 1. 第一次复用：如果该call之前已经获取连接（call.connection非null）
> 2. 第二次复用：从连接池中获取（无Route）
> 3. 第三次复用：从连接池中获取（有Route）
> 4. 第四次复用：从连接池中获取（有Route，但要求多路复用）







##### 连接清理

`ExchangeFinder.kt`

> 每一次使用新创建的连接的时候都会想connectionPool中存入连接

```kotlin
synchronized(newConnection) {
      connectionPool.put(newConnection)
      call.acquireConnectionNoEvents(newConnection)
}
```



`RealConnectionPool.kt`

> 每一次存入连接过程都会想clean队列存入一个clean task

```kotlin
fun put(connection: RealConnection) {
  connection.assertThreadHoldsLock()

  connections.add(connection)
  cleanupQueue.schedule(cleanupTask)
}
```



> 调度

```kotlin
fun schedule(task: Task, delayNanos: Long = 0L) {
    // 确保线程安全
  synchronized(taskRunner) {
    if (shutdown) {
      if (task.cancelable) {
        taskLog(task, this) { "schedule canceled (queue is shutdown)" }
        return
      }
      taskLog(task, this) { "schedule failed (queue is shutdown)" }
      throw RejectedExecutionException()
    }
	// schedule任务
    if (scheduleAndDecide(task, delayNanos, recurrence = false)) {
      taskRunner.kickCoordinator(this)
    }
  }
}
```





```kotlin
internal fun scheduleAndDecide(task: Task, delayNanos: Long, recurrence: Boolean): Boolean {
    // 初始化cleanQueue
  task.initQueue(this)
	// 获取当前时间
  val now = taskRunner.backend.nanoTime()
    // 计算执行任务的时间
  val executeNanoTime = now + delayNanos

  // If the task is already scheduled, take the earlier of the two times.
  val existingIndex = futureTasks.indexOf(task)
  if (existingIndex != -1) {
    if (task.nextExecuteNanoTime <= executeNanoTime) {
      taskLog(task, this) { "already scheduled" }
      return false
    }
    futureTasks.removeAt(existingIndex) // Already scheduled later: reschedule below!
  }
  task.nextExecuteNanoTime = executeNanoTime
  taskLog(task, this) {
    if (recurrence) "run again after ${formatDuration(executeNanoTime - now)}"
    else "scheduled after ${formatDuration(executeNanoTime - now)}"
  }
	// 插入任务
  var insertAt = futureTasks.indexOfFirst { it.nextExecuteNanoTime - now > delayNanos }
  if (insertAt == -1) insertAt = futureTasks.size
  futureTasks.add(insertAt, task)
	// 如果task被插入队首
  return insertAt == 0
}
```





```kotlin
private val runnable: Runnable = object : Runnable {
    override fun run() {
      while (true) {
        val task = synchronized(this@TaskRunner) {
            // 等待下一次运行
          awaitTaskToRun()
        } ?: return

        logElapsed(task, task.queue!!) {
          var completedNormally = false
          try {
              // 执行
            runTask(task)
            completedNormally = true
          } finally {
            // If the task is crashing start another thread to service the queues.
            if (!completedNormally) {
              backend.execute(this)
            }
          }
        }
      }
    }
  }

internal fun kickCoordinator(taskQueue: TaskQueue) {
  this.assertThreadHoldsLock()
	// 如果当前队列没有活跃的task
  if (taskQueue.activeTask == null) {
      // 加入task
    if (taskQueue.futureTasks.isNotEmpty()) {
      readyQueues.addIfAbsent(taskQueue)
    } else {
      readyQueues.remove(taskQueue)
    }
  }

  if (coordinatorWaiting) {
    backend.coordinatorNotify(this@TaskRunner)
  } else {
      // 执行
    backend.execute(runnable)
  }
}
```

`RealBackend.kt`

```kotlin
private val executor = ThreadPoolExecutor(
        0, // corePoolSize.
        Int.MAX_VALUE, // maximumPoolSize.
        60L, TimeUnit.SECONDS, // keepAliveTime.
        SynchronousQueue(),
        threadFactory
    )

override fun execute(runnable: Runnable) {
      executor.execute(runnable)
    }
```



> 连接清理是通过新建连接的同时向线程池中加入一个清理任务，这样就清理的任务就交给了线程池执行。
>
> 清理策略是遍历所有空闲的连接，把超过空闲最大时长的清除，如果都没有超过最大空闲时长，但是空闲连接数超出最大值，则将空闲最久的连接清除。



#### ConnectSpec

> 用于配置http连接的配置

```kotlin
class ConnectionSpec internal constructor(
    // 是否支持tls
  @get:JvmName("isTls") val isTls: Boolean,
    // 是否支持tlsExtension
    // SNI ECC ALPN ......
  @get:JvmName("supportsTlsExtensions") val supportsTlsExtensions: Boolean,
    // 支持的加密套件版本号
  private val cipherSuitesAsString: Array<String>?,
    // 支持的tls版本号
  private val tlsVersionsAsString: Array<String>?
) 
```



> 内置的实例化对象

```kotlin
// 最新的加密配置
@JvmField
val RESTRICTED_TLS = Builder(true)
// 最新的加密套件
    .cipherSuites(*RESTRICTED_CIPHER_SUITES)
    .tlsVersions(TlsVersion.TLS_1_3, TlsVersion.TLS_1_2)
    .supportsTlsExtensions(true)
    .build()

// 较为现代化的配置
@JvmField
    val MODERN_TLS = Builder(true)
        .cipherSuites(*APPROVED_CIPHER_SUITES)
        .tlsVersions(TlsVersion.TLS_1_3, TlsVersion.TLS_1_2)
        .supportsTlsExtensions(true)
        .build()

// 兼容性的加密配置
// 支持版本比较多
@JvmField
    val COMPATIBLE_TLS = Builder(true)
        .cipherSuites(*APPROVED_CIPHER_SUITES)
        .tlsVersions(TlsVersion.TLS_1_3, TlsVersion.TLS_1_2, TlsVersion.TLS_1_1, TlsVersion.TLS_1_0)
        .supportsTlsExtensions(true)
        .build()

// 明文
@JvmField
 val CLEARTEXT = Builder(false).build()
```



> 默认的配置

```kotlin
internal var connectionSpecs: List<ConnectionSpec> = DEFAULT_CONNECTION_SPECS
```

```kotlin
// 即现代化的加密套件和明文
internal val DEFAULT_CONNECTION_SPECS = immutableListOf(
    ConnectionSpec.MODERN_TLS, ConnectionSpec.CLEARTEXT)
```



> 自定义配置

```kotlin
val okhttpClient = OkHttpClient.Builder()
//......
    .connectionSpecs(listOf(ConnectionSpec.Builder(ConnectionSpec.COMPATIBLE_TLS)
        .cipherSuites(CipherSuite.TLS_AES_128_CCM_8_SHA256)
        .build()))
//......
    .build()
```







#### CookieJar

> 用于管理Cookie的实体类



> 关于Cookie

> Cookie是一个key=value的键值对，多个cookie之间通过分号和空格即（"; "）分隔。如`Set-Cookie: username=Alice; Expires=Wed, 01-Jan-2023 00:00:00 GMT; Path=/`

> cookie参数含义如下
>
> - `username=Alice`自定义的key=value，可以根据实际情况
> - `Expires=Wed, 01-Jan-2023 00:00:00 GMT`: Cookie 的过期时间，以 GMT 时间格式表示。
> - `Path=/`: Cookie 的可访问路径，设置为 "/" 表示该 Cookie 可以在整个网站的任意页面中访问。

> 桥接拦截器中有使用到这个类

> 逻辑很简单即在拦截器处理前后获取cookie，更新cookie

```kotlin
override fun intercept(chain: Interceptor.Chain): Response {
  

 
// 依据url加载cookie
  val cookies = cookieJar.loadForRequest(userRequest.url)
    // 将加载的cookie加入header
  if (cookies.isNotEmpty()) {
    requestBuilder.header("Cookie", cookieHeader(cookies))
  }

	// 交由后续拦截器处理，获得response
  val networkResponse = chain.proceed(requestBuilder.build())
	// 更新cookie
  cookieJar.receiveHeaders(userRequest.url, networkResponse.headers)

  //.......
}
```



> 接口

```kotlin
interface CookieJar {
  fun saveFromResponse(url: HttpUrl, cookies: List<Cookie>)

 
  fun loadForRequest(url: HttpUrl): List<Cookie>

  companion object {
  	// 一个不包含任何cookie的cookieJar实例
    @JvmField
    val NO_COOKIES: CookieJar = NoCookies()
    private class NoCookies : CookieJar {
      override fun saveFromResponse(url: HttpUrl, cookies: List<Cookie>) {
      }

      override fun loadForRequest(url: HttpUrl): List<Cookie> {
        return emptyList()
      }
    }
  }
}
```



> 向Client配置CookieJar

```kotlin
val okhttpClient = okhttpBuilder
    .cookieJar(youCookieJar())
    .build()
```



#### Dispatcher

> 用以调度okhttp call的执行

![image-20230305190959843](https://typora-blog-picture.oss-cn-chengdu.aliyuncs.com/blog/image-20230305190959843.png)



> 构造函数
>
> - `constructor()`
>
> - `constructor(executorService: ExecutorService) : this()`
>
>   设置线程池



> 成员属性
>
> ```kotlin
> // 最大并发请求数量，即同时运行的request最大数量。
> @get:Synchronized var maxRequests = 64
>   set(maxRequests) {
>     require(maxRequests >= 1) { "max < 1: $maxRequests" }
>     synchronized(this) {
>       field = maxRequests
>     }
>       // 尝试执行
>     promoteAndExecute()
>   }
> ```
>
> ```kotlin
> // 每个主机最大请求数目
> @get:Synchronized var maxRequestsPerHost = 5
>   set(maxRequestsPerHost) {
>     require(maxRequestsPerHost >= 1) { "max < 1: $maxRequestsPerHost" }
>     synchronized(this) {
>       field = maxRequestsPerHost
>     }
>     promoteAndExecute()
>   }
> ```
>
> ```kotlin
> // 空闲回调 
> @set:Synchronized
>   @get:Synchronized
>   var idleCallback: Runnable? = null
> 
> // 异步运行的线程池
> private var executorServiceOrNull: ExecutorService? = null
> 
>   @get:Synchronized
>   @get:JvmName("executorService") val executorService: ExecutorService
>     get() {
>         // 设置默认线程池
>       if (executorServiceOrNull == null) {
>         executorServiceOrNull = ThreadPoolExecutor(0, Int.MAX_VALUE, 60, TimeUnit.SECONDS,
>             SynchronousQueue(), threadFactory("$okHttpName Dispatcher", false))
>       }
>       return executorServiceOrNull!!
>     }
> ```
>
> ```kotlin
> // 异步准备call队列
> private val readyAsyncCalls = ArrayDeque<AsyncCall>()
> 
> // 异步运行队列
> private val runningAsyncCalls = ArrayDeque<AsyncCall>()
> 
> // 同步运行队列
> private val runningSyncCalls = ArrayDeque<RealCall>()
> ```



> 成员方法
>
> ```kotlin
> // 执行异步的call
> internal fun enqueue(call: AsyncCall) {
>   synchronized(this) {
>       // 放入准备队列
>     readyAsyncCalls.add(call)
> 
>     // 相同的call公用一个原子integer（用于计数每个host的runningCall数量）
>     if (!call.call.forWebSocket) {
>       val existingCall = findExistingCallWithHost(call.host)
>       if (existingCall != null) call.reuseCallsPerHostFrom(existingCall)
>     }
>   }
>   promoteAndExecute()
> }
> 
> // 寻找与host相同的ayncCall返回满足的第一个call
>   private fun findExistingCallWithHost(host: String): AsyncCall? {
>     for (existingCall in runningAsyncCalls) {
>       if (existingCall.host == host) return existingCall
>     }
>     for (existingCall in readyAsyncCalls) {
>       if (existingCall.host == host) return existingCall
>     }
>     return null
>   }
> 
> 
> // 尝试执行。
> private fun promoteAndExecute(): Boolean {
>     this.assertThreadDoesntHoldLock()
> 
>     val executableCalls = mutableListOf<AsyncCall>()
>     val isRunning: Boolean
>     // 确保安全
>     synchronized(this) {
>       val i = readyAsyncCalls.iterator()
>       while (i.hasNext()) {
>         val asyncCall = i.next()
> 		// 如果运行的请求数上限，或者host的请求数超过上限
>         if (runningAsyncCalls.size >= this.maxRequests) break // Max capacity.
>         if (asyncCall.callsPerHost.get() >= this.maxRequestsPerHost) continue // Host max capacity.
> 		// 满足运行条件
>         // 从ready队列移除
>         i.remove()
>           // 累计host的请求数
>         asyncCall.callsPerHost.incrementAndGet()
> 			// 放入执行队列
>         executableCalls.add(asyncCall)
>         runningAsyncCalls.add(asyncCall)
>       }
>       isRunning = runningCallsCount() > 0
>     }
> 	// 执行所有的之前筛选出来的asyncCall
>     for (i in 0 until executableCalls.size) {
>       val asyncCall = executableCalls[i]
>       asyncCall.executeOn(executorService)
>     }
> 
>     return isRunning
>   }
> ```
>
> ```kotlin
> // 取消所有call
> @Synchronized fun cancelAll() {
>   for (call in readyAsyncCalls) {
>     call.call.cancel()
>   }
>   for (call in runningAsyncCalls) {
>     call.call.cancel()
>   }
>   for (call in runningSyncCalls) {
>     call.cancel()
>   }
> }
> ```
>
> ```kotlin
> // 执行同步请求
> @Synchronized internal fun executed(call: RealCall) {
>     // 加入runningCall
>   runningSyncCalls.add(call)
> }
> ```
>
> ```kotlin
> // call执行完成
> internal fun finished(call: AsyncCall) {
>   call.callsPerHost.decrementAndGet()
>   finished(runningAsyncCalls, call)
> }
> 
> internal fun finished(call: RealCall) {
>     finished(runningSyncCalls, call)
>   }
> 
> private fun <T> finished(calls: Deque<T>, call: T) {
>     val idleCallback: Runnable?
>     synchronized(this) {
>         // 移除
>       if (!calls.remove(call)) throw AssertionError("Call wasn't in-flight!")
>       idleCallback = this.idleCallback
>     }
> 	// 重新入队
>     val isRunning = promoteAndExecute()
> 	// 如果没有可运行的call
>     if (!isRunning && idleCallback != null) {
>         // 执行idle
>       idleCallback.run()
>     }
>   }
> ```
>
> ```kotlin
> // 队列获取
> @Synchronized fun queuedCalls(): List<Call> {
>   return Collections.unmodifiableList(readyAsyncCalls.map { it.call })
> }
> 
> @Synchronized fun runningCalls(): List<Call> {
>   return Collections.unmodifiableList(runningSyncCalls + runningAsyncCalls.map { it.call })
> }
> 
> @Synchronized fun queuedCallsCount(): Int = readyAsyncCalls.size
> 
> @Synchronized fun runningCallsCount(): Int = runningAsyncCalls.size + runningSyncCalls.size
> ```



> 可见，dispatcher就是调度器，控制call的调度方式，同步异步调度。

`RealCall`

```kotlin
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

override fun enqueue(responseCallback: Callback) {
  check(executed.compareAndSet(false, true)) { "Already Executed" }

  callStart()
  client.dispatcher.enqueue(AsyncCall(responseCallback))
}
```



#### Dns

> DNS即Domain Name System

> 可以用于获取域名到IP地址的映射的分布式k-v数据库

```kotlin
interface Dns {
    
  @Throws(UnknownHostException::class)
  fun lookup(hostname: String): List<InetAddress>

}
```



> 内置实例

```kotlin
companion object {

  @JvmField
  val SYSTEM: Dns = DnsSystem()
  private class DnsSystem : Dns {
    override fun lookup(hostname: String): List<InetAddress> {
      try {
        return InetAddress.getAllByName(hostname).toList()
      } catch (e: NullPointerException) {
        throw UnknownHostException("Broken system behaviour for dns lookup of $hostname").apply {
          initCause(e)
        }
      }
    }
  }
}
```



> 过程

`ConnectInterceptor` ->

​	`realChain.call.initExchange(chain)`->

​		`exchangeFinder.find(client, chain)`->

​			......

​				`findConnection`->

​					`localRouteSelector.next()`



`RouteSelector.kt`

```kotlin
@Throws(IOException::class)
private fun resetNextInetSocketAddress(proxy: Proxy) {
  // Clear the addresses. Necessary if getAllByName() below throws!
  val mutableInetSocketAddresses = mutableListOf<InetSocketAddress>()
  inetSocketAddresses = mutableInetSocketAddresses

  val socketHost: String
  val socketPort: Int
    // 获取代理的地址
  if (proxy.type() == Proxy.Type.DIRECT || proxy.type() == Proxy.Type.SOCKS) {
    socketHost = address.url.host
    socketPort = address.url.port
  } else {
    val proxyAddress = proxy.address()
    require(proxyAddress is InetSocketAddress) {
      "Proxy.address() is not an InetSocketAddress: ${proxyAddress.javaClass}"
    }
    socketHost = proxyAddress.socketHost
    socketPort = proxyAddress.port
  }

    // socks代理
  if (proxy.type() == Proxy.Type.SOCKS) {
    mutableInetSocketAddresses += InetSocketAddress.createUnresolved(socketHost, socketPort)
  } else {
      // 查看dns
    eventListener.dnsStart(call, socketHost)

    // Try each address for best behavior in mixed IPv4/IPv6 environments.
    val addresses = address.dns.lookup(socketHost)
    if (addresses.isEmpty()) {
      throw UnknownHostException("${address.dns} returned no addresses for $socketHost")
    }

    eventListener.dnsEnd(call, socketHost, addresses)

    for (inetAddress in addresses) {
      mutableInetSocketAddresses += InetSocketAddress(inetAddress, socketPort)
    }
  }
}
```



> DNS即在RouteSelector解析Proxy时解析hostName



#### Proxy/ProxySelector

> 类结构信息

> Proxy是Java net的一个bean包含type和address
>
> - type
>
>   即代理类型DIRECT（直连），HTTP（HTTP代理），SOCKS（SOCKS代理）

![image-20230305175434624](https://typora-blog-picture.oss-cn-chengdu.aliyuncs.com/blog/image-20230305175434624.png)



> ProxySelector 

> proxy的factory，一次可以生产一个proxy集合

![image-20230305181104847](https://typora-blog-picture.oss-cn-chengdu.aliyuncs.com/blog/image-20230305181104847.png)



`RouteSelector.kt`

```kotlin

  init {
    resetNextProxy(address.url, address.proxy)
  }

private fun resetNextProxy(url: HttpUrl, proxy: Proxy?) {
    fun selectProxies(): List<Proxy> {
        // 如果有proxy，使用proxy
        // 没proxy使用proxySelector
      if (proxy != null) return listOf(proxy)

   
      val uri = url.toUri()
      if (uri.host == null) return immutableListOf(Proxy.NO_PROXY)

      val proxiesOrNull = address.proxySelector.select(uri)
      if (proxiesOrNull.isNullOrEmpty()) return immutableListOf(Proxy.NO_PROXY)

      return proxiesOrNull.toImmutableList()
    }

    eventListener.proxySelectStart(call, url)
    proxies = selectProxies()
    nextProxyIndex = 0
    eventListener.proxySelectEnd(call, url, proxies)
  }
```



> Proxy的使用会经由如下调用
>
> （获取的proxy会根据dns获取ip）

`RouteSelector.next`->

​	`nextProxy`->

​		`proxies[nextProxyIndex++]`（依据放入次序顺序拿去proxy）

> 配置

```kotlin
val okhttpClient = OkHttpClient.Builder()
    .proxy(Proxy.NO_PROXY)
    .proxySelector(ProxySelector.getDefault())
    .build()
```



#### EventListener

> 事件监听器

```kotlin
abstract class EventListener {

  open fun callStart(
    call: Call
  ) {
  }

  open fun proxySelectStart(
    call: Call,
    url: HttpUrl
  ) {
  }

  open fun proxySelectEnd(
    call: Call,
    url: HttpUrl,
    proxies: List<@JvmSuppressWildcards Proxy>
  ) {
  }

  open fun dnsStart(
    call: Call,
    domainName: String
  ) {
  }

  open fun dnsEnd(
    call: Call,
    domainName: String,
    inetAddressList: List<@JvmSuppressWildcards InetAddress>
  ) {
  }

  open fun connectStart(
    call: Call,
    inetSocketAddress: InetSocketAddress,
    proxy: Proxy
  ) {
  }

  open fun secureConnectStart(
    call: Call
  ) {
  }

  open fun secureConnectEnd(
    call: Call,
    handshake: Handshake?
  ) {
  }

  open fun connectEnd(
    call: Call,
    inetSocketAddress: InetSocketAddress,
    proxy: Proxy,
    protocol: Protocol?
  ) {
  }

  open fun connectFailed(
    call: Call,
    inetSocketAddress: InetSocketAddress,
    proxy: Proxy,
    protocol: Protocol?,
    ioe: IOException
  ) {
  }

  open fun connectionAcquired(
    call: Call,
    connection: Connection
  ) {
  }

  open fun connectionReleased(
    call: Call,
    connection: Connection
  ) {
  }

  open fun requestHeadersStart(
    call: Call
  ) {
  }

  open fun requestHeadersEnd(call: Call, request: Request) {
  }

  open fun requestBodyStart(
    call: Call
  ) {
  }

  open fun requestBodyEnd(
    call: Call,
    byteCount: Long
  ) {
  }

  open fun requestFailed(
    call: Call,
    ioe: IOException
  ) {
  }

  open fun responseHeadersStart(
    call: Call
  ) {
  }

  open fun responseHeadersEnd(
    call: Call,
    response: Response
  ) {
  }

  open fun responseBodyStart(
    call: Call
  ) {
  }

  open fun responseBodyEnd(
    call: Call,
    byteCount: Long
  ) {
  }

  open fun responseFailed(
    call: Call,
    ioe: IOException
  ) {
  }

  open fun callEnd(
    call: Call
  ) {
  }

  
  open fun callFailed(
    call: Call,
    ioe: IOException
  ) {
  }

  
  open fun canceled(
    call: Call
  ) {
  }

 
  open fun satisfactionFailure(call: Call, response: Response) {
  }

  
  open fun cacheHit(call: Call, response: Response) {
  }

  
  open fun cacheMiss(call: Call) {
  }

 
  open fun cacheConditionalHit(call: Call, cachedResponse: Response) {
  }

}
```



#### ssl连接配置

> 此过程包含3部分
>
> - `sslSocketFactory`
>
>   - `sslSocketFactory`
>
>     管理ssl socket创建
>
>   - `trustManager`
>
>     用于管理客户端可信证书
>
> - `hostnameVerifier`
>
>   依据host和ssl会话判断是否接入
>
> - `certificatePinner`
>
>   绑定服务端的证书证书
>



> 配置逻辑

```kotlin
init {
    // 如果不是是tls连接
  if (connectionSpecs.none { it.isTls }) {
      // 无需sslSocket，certificate当然也不需要证书绑定
    this.sslSocketFactoryOrNull = null
    this.certificateChainCleaner = null
    this.x509TrustManager = null
    this.certificatePinner = CertificatePinner.DEFAULT
  } else if (builder.sslSocketFactoryOrNull != null) {
      // 如果设置了sslSocketFactory，采用设置的参数
    this.sslSocketFactoryOrNull = builder.sslSocketFactoryOrNull
    this.certificateChainCleaner = builder.certificateChainCleaner!!
    this.x509TrustManager = builder.x509TrustManagerOrNull!!
    this.certificatePinner = builder.certificatePinner
        .withCertificateChainCleaner(certificateChainCleaner!!)
  } else {
      // 如果是tls连接，并且没有设置配置
      // 获取平台的默认实现
    this.x509TrustManager = Platform.get().platformTrustManager()
    this.sslSocketFactoryOrNull = Platform.get().newSslSocketFactory(x509TrustManager!!)
    this.certificateChainCleaner = CertificateChainCleaner.get(x509TrustManager!!)
    this.certificatePinner = builder.certificatePinner
        .withCertificateChainCleaner(certificateChainCleaner!!)
  }
```

> 配置sslSocketFactory

```kotlin
fun sslSocketFactory(
  sslSocketFactory: SSLSocketFactory,
  trustManager: X509TrustManager
) = apply {
  if (sslSocketFactory != this.sslSocketFactoryOrNull || trustManager != this.x509TrustManagerOrNull) {
    this.routeDatabase = null
  }
	// 用于创建sslSocket即tls连接
  this.sslSocketFactoryOrNull = sslSocketFactory
    // 用于对无关证书进行清理，每次tls握手发送的ca证书都会经过cleaner清理
  this.certificateChainCleaner = CertificateChainCleaner.get(trustManager)
    // 证书的管理类
  this.x509TrustManagerOrNull = trustManager
}
```

> 证书绑定

```kotlin
fun certificatePinner(certificatePinner: CertificatePinner) = apply {
  if (certificatePinner != this.certificatePinner) {
    this.routeDatabase = null
  }

  this.certificatePinner = certificatePinner
}
```

> hostname校验

```kotlin
fun hostnameVerifier(hostnameVerifier: HostnameVerifier) = apply {
  if (hostnameVerifier != this.hostnameVerifier) {
    this.routeDatabase = null
  }

  this.hostnameVerifier = hostnameVerifier
}
```



`RealConnection.kt`

> tls连接建立过程

```kotlin
private fun connectTls(connectionSpecSelector: ConnectionSpecSelector) {
    val address = route.address
    val sslSocketFactory = address.sslSocketFactory
    var success = false
    var sslSocket: SSLSocket? = null
    try {
      // 创建用于tls的socket
      sslSocket = sslSocketFactory!!.createSocket(
          rawSocket, address.url.host, address.url.port, true /* autoClose */) as SSLSocket

      // 配置socket连接
      val connectionSpec = connectionSpecSelector.configureSecureSocket(sslSocket)
      if (connectionSpec.supportsTlsExtensions) {
        Platform.get().configureTlsExtensions(sslSocket, address.url.host, address.protocols)
      }

      // tls握手
      sslSocket.startHandshake()
  
      val sslSocketSession = sslSocket.session
      val unverifiedHandshake = sslSocketSession.handshake()

      // 校验hostname
      if (!address.hostnameVerifier!!.verify(address.url.host, sslSocketSession)) {
        val peerCertificates = unverifiedHandshake.peerCertificates
        if (peerCertificates.isNotEmpty()) {
          val cert = peerCertificates[0] as X509Certificate
          throw SSLPeerUnverifiedException(......)
        } else {
          throw SSLPeerUnverifiedException("Hostname ${address.url.host} not verified (no certificates)")
        }
      }

      val certificatePinner = address.certificatePinner!!

      handshake = Handshake(unverifiedHandshake.tlsVersion, unverifiedHandshake.cipherSuite,
          unverifiedHandshake.localCertificates) {
        certificatePinner.certificateChainCleaner!!.clean(unverifiedHandshake.peerCertificates,
            address.url.host)
      }

      // 服务端证书绑定
      certificatePinner.check(address.url.host) {
        handshake!!.peerCertificates.map { it as X509Certificate }
      }

      // 校验成功
      val maybeProtocol = if (connectionSpec.supportsTlsExtensions) {
        Platform.get().getSelectedProtocol(sslSocket)
      } else {
        null
      }
      // ......
    } finally {
       // ......
    }
  }
  }
```



> tls连接过程

![okhttp-tls.drawio](https://typora-blog-picture.oss-cn-chengdu.aliyuncs.com/blog/okhttp-tls.drawio.png)



















#### socket

> `SocketFactory`可以用来创建socket

`RealConnection.kt`

> 当需要连接socket时会通过socketFactory创建socket

```kotlin
private fun connectSocket(
  connectTimeout: Int,
  readTimeout: Int,
  call: Call,
  eventListener: EventListener
) {
 	
    val proxy = route.proxy
    val address = route.address

    val rawSocket = when (proxy.type()) {
      Proxy.Type.DIRECT, Proxy.Type.HTTP -> address.socketFactory.createSocket()!!
      else -> Socket(proxy)
    }
    //......
    
}    
```



> `SocketFactory`类结构

![image-20230307165012458](https://typora-blog-picture.oss-cn-chengdu.aliyuncs.com/blog/image-20230307165012458.png)



`OkHttpClient.kt`

> 设置socketFactory

```kotlin
fun socketFactory(socketFactory: SocketFactory) = apply {
  require(socketFactory !is SSLSocketFactory) { "socketFactory instanceof SSLSocketFactory" }

  if (socketFactory != this.socketFactory) {
    this.routeDatabase = null
  }

  this.socketFactory = socketFactory
}
```









#### 其他

- 重定向

  - `followRedirects`

    相同协议是否进行重定向http -> http,https -> https

  - `followSslRedirects`

    不同协议是否进行重定向，http-> https ,https -> http

- 协议配置

  - `protocols`

- 超时配置

  - `callTimeoutMillis`

  - `connectTimeMillis`
  - `readTimeoutMillis`

  - `writeTimeoutMillis`

  - `pingIntervalMillis`

- 其他

  - 



##### 重定向

> 调用栈

`RetryAndFollowUpInterceptor.intercept` ->

​	`followUpRequest`->

​		`buildRedirectRequest`->

```kotlin
private fun buildRedirectRequest(userResponse: Response, method: String): Request? {
 	
    if (!client.followRedirects) return null
    
    // ......
    val sameScheme = url.scheme == userResponse.request.url.scheme
    if (!sameScheme && !client.followSslRedirects) return null
    
}    
```



##### 协议配置



```kotlin
enum class Protocol(private val protocol: String) {
    
  HTTP_1_0("http/1.0"),

  HTTP_1_1("http/1.1"),

  @Deprecated("OkHttp has dropped support for SPDY. Prefer {@link #HTTP_2}.")
  SPDY_3("spdy/3.1"),

  HTTP_2("h2"),

  H2_PRIOR_KNOWLEDGE("h2_prior_knowledge"),

  QUIC("quic");
    
  override fun toString() = protocol

}

```



> 默认协议配置

```kotlin
internal val DEFAULT_PROTOCOLS = immutableListOf(HTTP_2, HTTP_1_1)
```



> 配置连接

> 主要适配了http2协议

```kotlin
private fun establishProtocol(
  connectionSpecSelector: ConnectionSpecSelector,
  pingIntervalMillis: Int,
  call: Call,
  eventListener: EventListener
) {
    // 没有设置tls连接
  if (route.address.sslSocketFactory == null) {
      // h2w
    if (Protocol.H2_PRIOR_KNOWLEDGE in route.address.protocols) {
      socket = rawSocket
      protocol = Protocol.H2_PRIOR_KNOWLEDGE
      startHttp2(pingIntervalMillis)
      return
    }

    socket = rawSocket
    protocol = Protocol.HTTP_1_1
    return
  }

  eventListener.secureConnectStart(call)
  connectTls(connectionSpecSelector)
  eventListener.secureConnectEnd(call, handshake)
    // h2
  if (protocol === Protocol.HTTP_2) {
    startHttp2(pingIntervalMillis)
  }
}
```



##### 超时配置



> callTimeout

`Builder.kt`

```kotlin
// 默认没有打开超时 
internal var callTimeout = 0
```

`OkHttpClient.kt`

```kotlin
val callTimeoutMillis: Int = builder.callTimeout
```

`RealCall.kt`

```kotlin
private val timeout = object : AsyncTimeout() {
  override fun timedOut() {
    cancel()
  }
}.apply {
  timeout(client.callTimeoutMillis.toLong(), MILLISECONDS)
}
```



> 如下设置会直接映射到socket的超时设置（10秒）

```kotlin
internal var connectTimeout = 10_000
internal var readTimeout = 10_000
internal var writeTimeout = 10_000
```



> Web socket and HTTP/2 ping interval (in milliseconds). By default pings are not sent.
>
> web socket或者http/2的心跳包间隔时间，默认没有设置心跳包

```kotlin
internal var pingInterval = 0
```

`RealConnection.kt`

```kotlin
private fun startHttp2(pingIntervalMillis: Int) {
  val socket = this.socket!!
  val source = this.source!!
  val sink = this.sink!!
    // 0心跳包
  socket.soTimeout = 0 // HTTP/2 connection timeouts are set per-stream.
  val http2Connection = Http2Connection.Builder(client = true, taskRunner = TaskRunner.INSTANCE)
      .socket(socket, route.address.url.host, source, sink)
      .listener(this)
      .pingIntervalMillis(pingIntervalMillis)
      .build()
  this.http2Connection = http2Connection
  this.allocationLimit = Http2Connection.DEFAULT_SETTINGS.getMaxConcurrentStreams()
  http2Connection.start()
}
```

`Http2Connection`

```kotlin
if (builder.pingIntervalMillis != 0) { // 如果pingIntervalMillis则表明没有心跳包
    // 入队执行
  val pingIntervalNanos = TimeUnit.MILLISECONDS.toNanos(builder.pingIntervalMillis.toLong())
  writerQueue.schedule("$connectionName ping", pingIntervalNanos) {
    val failDueToMissingPong = synchronized(this@Http2Connection) {
      if (intervalPongsReceived < intervalPingsSent) {
        return@synchronized true
      } else {
        intervalPingsSent++
        return@synchronized false
      }
    }
    if (failDueToMissingPong) {
      failConnection(null)
      return@schedule -1L
    } else {
      writePing(false, INTERVAL_PING, 0)
      return@schedule pingIntervalNanos
    }
  }
}
```





## Websocket



### 使用



> 可以发现Websocket的使用和http请求过程是类似的，只是在request设置上有少许不同。
>
> 另外http是通过newCall获取call实例对象，而websocket是通过newWebsocket创建call对象

```kotlin
fun main() {

    val request = Request.Builder()
        .get()
        .url("ws://ip:port")
        .build()

    val webSocket = okhttpClient.newWebSocket(request, object : WebSocketListener() {
        override fun onMessage(webSocket: WebSocket, text: String) {
            println(text)
            super.onMessage(webSocket, text)
        }

        override fun onOpen(webSocket: WebSocket, response: Response) {
            super.onOpen(webSocket, response)
            println(response)
        }

        override fun onFailure(webSocket: WebSocket, t: Throwable, response: Response?) {
            super.onFailure(webSocket, t, response)
            println(t)
        }
    })

  	// write
    // ......

}
```



### Websocket创建

> 创建websocket连接

```kotlin
override fun newWebSocket(request: Request, listener: WebSocketListener): WebSocket {
    // websocket实例类似于RealCall
  val webSocket = RealWebSocket(
      taskRunner = TaskRunner.INSTANCE,
      originalRequest = request,
      listener = listener,
      random = Random(),
      pingIntervalMillis = pingIntervalMillis.toLong(),
      extensions = null, // Always null for clients.
      minimumDeflateSize = minWebSocketMessageToCompress
  )
    // 连接
  webSocket.connect(this)
  return webSocket
}
```

> 与服务端建立websocket连接

```kotlin
fun connect(client: OkHttpClient) {
  if (originalRequest.header("Sec-WebSocket-Extensions") != null) {
    failWebSocket(ProtocolException(
        "Request header not permitted: 'Sec-WebSocket-Extensions'"), null)
    return
  }
// 配置专用的client
  val webSocketClient = client.newBuilder()
      .eventListener(EventListener.NONE)
      .protocols(ONLY_HTTP1)
      .build()
    // 使用http/1.1发送协议升级请求
  val request = originalRequest.newBuilder()
      .header("Upgrade", "websocket")
      .header("Connection", "Upgrade")
      .header("Sec-WebSocket-Key", key)
      .header("Sec-WebSocket-Version", "13")
      .header("Sec-WebSocket-Extensions", "permessage-deflate")
      .build()
    // 创建call对象
  call = RealCall(webSocketClient, request, forWebSocket = true)
    // 异步请求
  call!!.enqueue(object : Callback {
    override fun onResponse(call: Call, response: Response) {
      val exchange = response.exchange
      val streams: Streams
      try {
          // 查看协议升级结果
        checkUpgradeSuccess(response, exchange)
        streams = exchange!!.newWebSocketStreams()
      } catch (e: IOException) { // 异常处理，关闭请求
        exchange?.webSocketUpgradeFailed()
        failWebSocket(e, response)
        response.closeQuietly()
        return
      }

      // Apply the extensions. If they're unacceptable initiate a graceful shut down.
      // TODO(jwilson): Listeners should get onFailure() instead of onClosing() + onClosed(1010).
        // 解析配置参数
      val extensions = WebSocketExtensions.parse(response.headers)
      this@RealWebSocket.extensions = extensions
      if (!extensions.isValid()) {
        synchronized(this@RealWebSocket) {
          messageAndCloseQueue.clear() // Don't transmit any messages.
          close(1010, "unexpected Sec-WebSocket-Extensions in response header")
        }
      }

      // Process all web socket messages.
      try {
        val name = "$okHttpName WebSocket ${request.url.redact()}"
          // 初始化必要参数信息
        initReaderAndWriter(name, streams)
          // 通知listener
        listener.onOpen(this@RealWebSocket, response)
          // 死循环读取消息
        loopReader()
      } catch (e: Exception) {
        failWebSocket(e, null)
      }
    }

    override fun onFailure(call: Call, e: IOException) {
        // 关闭请求
      failWebSocket(e, null)
    }
  })
}
```



> 初始化websocket成员变量和连接必要参数

> 特别重要

```kotlin
fun initReaderAndWriter(name: String, streams: Streams) {
  val extensions = this.extensions!!
  synchronized(this) {
      // 参数
    this.name = name
    this.streams = streams
		// 连接包装类，writer
    this.writer = WebSocketWriter(
        isClient = streams.client,
        sink = streams.sink,
        random = random,
        perMessageDeflate = extensions.perMessageDeflate,
        noContextTakeover = extensions.noContextTakeover(streams.client),
        minimumDeflateSize = minimumDeflateSize
    )
      // 写task
    this.writerTask = WriterTask()
      // 用于发送心跳包的task
    if (pingIntervalMillis != 0L) {
      val pingIntervalNanos = MILLISECONDS.toNanos(pingIntervalMillis)
      taskQueue.schedule("$name ping", pingIntervalNanos) {
        writePingFrame()
        return@schedule pingIntervalNanos
      }
    }
    if (messageAndCloseQueue.isNotEmpty()) {
      runWriter() // Send messages that were enqueued before we were connected.
    }
  }
	// 连接包装类，reader
  reader = WebSocketReader(
      isClient = streams.client,
      source = streams.source,
      frameCallback = this,
      perMessageDeflate = extensions.perMessageDeflate,
      noContextTakeover = extensions.noContextTakeover(!streams.client)
  )
}
```



> 读取数据

```kotlin
fun loopReader() {
    // 只要没有关闭，死循环读取
  while (receivedCloseCode == -1) {
    // This method call results in one or more onRead* methods being called on this thread.
    reader!!.processNextFrame()
  }
}
```



> 读取

```kotlin
@Throws(IOException::class)
fun processNextFrame() {
    // 读头部
  readHeader()
    // 读control或者message帧
  if (isControlFrame) {
    readControlFrame()
  } else {
    readMessageFrame()
  }
}
```



> 监听

```kotlin
private fun readControlFrame() {
   
    when (opcode) {
      OPCODE_CONTROL_PING -> {
          // ping
        frameCallback.onReadPing(controlFrameBuffer.readByteString())
      }
      OPCODE_CONTROL_PONG -> {
          // pong
        frameCallback.onReadPong(controlFrameBuffer.readByteString())
      }
      OPCODE_CONTROL_CLOSE -> {
        / ......
          // close
        frameCallback.onReadClose(code, reason)
        closed = true
      }
    }
  }
```



```kotlin
private fun readMessageFrame() {
  val opcode = this.opcode
  if (opcode != OPCODE_TEXT && opcode != OPCODE_BINARY) {
    throw ProtocolException("Unknown opcode: ${opcode.toHexString()}")
  }

  readMessage()

  // message

  if (opcode == OPCODE_TEXT) {
    frameCallback.onReadMessage(messageFrameBuffer.readUtf8())
  } else {
    frameCallback.onReadMessage(messageFrameBuffer.readByteString())
  }
}
```

> framecallback会调用到RealWebSocket的方法

![image-20230308142207355](https://typora-blog-picture.oss-cn-chengdu.aliyuncs.com/blog/image-20230308142207355.png)



> 方法如下

```kotlin
@Throws(IOException::class)
override fun onReadMessage(text: String) {
  listener.onMessage(this, text)
}

@Throws(IOException::class)
override fun onReadMessage(bytes: ByteString) {
  listener.onMessage(this, bytes)
}

@Synchronized override fun onReadPing(payload: ByteString) {
    // Don't respond to pings after we've failed or sent the close frame.
    if (failed || enqueuedClose && messageAndCloseQueue.isEmpty()) return

    pongQueue.add(payload)
    runWriter()
    receivedPingCount++
  }

  @Synchronized override fun onReadPong(payload: ByteString) {
    // This API doesn't expose pings.
    receivedPongCount++
    awaitingPong = false
  }

  override fun onReadClose(code: Int, reason: String) {
    require(code != -1)

    var toClose: Streams? = null
    var readerToClose: WebSocketReader? = null
    var writerToClose: WebSocketWriter? = null
    synchronized(this) {
      check(receivedCloseCode == -1) { "already closed" }
      receivedCloseCode = code
      receivedCloseReason = reason
      if (enqueuedClose && messageAndCloseQueue.isEmpty()) {
        toClose = this.streams
        this.streams = null
        readerToClose = this.reader
        this.reader = null
        writerToClose = this.writer
        this.writer = null
        this.taskQueue.shutdown()
      }
    }

    try {
      listener.onClosing(this, code, reason)

      if (toClose != null) {
        listener.onClosed(this, code, reason)
      }
    } finally {
      toClose?.closeQuietly()
      readerToClose?.closeQuietly()
      writerToClose?.closeQuietly()
    }
  }
```





### Websocket消息发送

```kotlin
override fun send(text: String): Boolean {
    // 发送text
  return send(text.encodeUtf8(), OPCODE_TEXT)
}

override fun send(bytes: ByteString): Boolean {
    // 发送二进制消息
  return send(bytes, OPCODE_BINARY)
}

@Synchronized private fun send(data: ByteString, formatOpcode: Int): Boolean {
  // Don't send new frames after we've failed or enqueued a close frame.
    // 确保没有异常
  if (failed || enqueuedClose) return false

  // If this frame overflows the buffer, reject it and close the web socket.
    // 如果待发送数据长度大于指定大小（15m）
    // 由于write操作是先写入Buffer然后在发送给服务端，为了避免oom限制大小。
  if (queueSize + data.size > MAX_QUEUE_SIZE) {
    close(CLOSE_CLIENT_GOING_AWAY, null)
    return false
  }

  // Enqueue the message frame.
    // 累计大小
  queueSize += data.size.toLong()
  messageAndCloseQueue.add(Message(formatOpcode, data))
    // 执行task
  runWriter()
  return true
}
```

> 线程池入队

```kotlin
private fun runWriter() {
  this.assertThreadHoldsLock()

  val writerTask = writerTask
  if (writerTask != null) {
    taskQueue.schedule(writerTask)
  }
}
```

> task

```kotlin
private inner class WriterTask : Task("$name writer") {
  override fun runOnce(): Long {
    try {
      if (writeOneFrame()) return 0L
    } catch (e: IOException) {
      failWebSocket(e, null)
    }
    return -1L
  }
}
```



> 写入一个数据帧

```kotlin
internal fun writeOneFrame(): Boolean {
  val writer: WebSocketWriter?
  val pong: ByteString?
  var messageOrClose: Any? = null
  var receivedCloseCode = -1
  var receivedCloseReason: String? = null
  var streamsToClose: Streams? = null
  var readerToClose: WebSocketReader? = null
  var writerToClose: WebSocketWriter? = null

  synchronized(this@RealWebSocket) {
    if (failed) {
      return false // Failed web socket.
    }

    writer = this.writer
    pong = pongQueue.poll()
    if (pong == null) {
      //......
    }
  }

  try {
    if (pong != null) {
      writer!!.writePong(pong)
    } else if (messageOrClose is Message) { // 发送请求
      val message = messageOrClose as Message
        // 写入message帧
      writer!!.writeMessageFrame(message.formatOpcode, message.data)
      synchronized(this) {
        queueSize -= message.data.size.toLong()
      }
    } else if (messageOrClose is Close) {
      val close = messageOrClose as Close
      writer!!.writeClose(close.code, close.reason)

      // We closed the writer: now both reader and writer are closed.
      if (streamsToClose != null) {
        listener.onClosed(this, receivedCloseCode, receivedCloseReason!!)
      }
    } else {
      throw AssertionError()
    }

    return true
  } finally {
      // 关闭
    streamsToClose?.closeQuietly()
    readerToClose?.closeQuietly()
    writerToClose?.closeQuietly()
  }
}
```
