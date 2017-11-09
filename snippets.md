# JSON

```kotlin
json {
  obj {
    "foo" to 1
  }
}
```

# Bidding service

```kotlin
class Service : AbstractVerticle() {

  val logger = LoggerFactory.getLogger(Service::class.java)

  override fun start() {
    val random = Random()
    val myId = UUID.randomUUID().toString()

    vertx.createHttpServer()
      .requestHandler { req ->
        val myBid = 10 + random.nextInt(20)
        req.response()
          .putHeader("Content-Type", "application/json")
          .end(JsonObject()
            .put("id", myId)
            .put("bid", myBid)
            .encode())
        logger.info("$myId bids $myBid")
      }
      .listen(config().getInteger("port", 3000))
  }
}
```

# RxJava 2 MainVerticle

```kotlin
class MainVerticle : AbstractVerticle() {

  val logger = LoggerFactory.getLogger(MainVerticle::class.java)

  override fun start(startFuture: Future<Void>) {

    val s1 = vertx.rxDeployVerticle("sample.Service",
      DeploymentOptions().setConfig(JsonObject().put("port", 3000))).toObservable()

    val s2 = vertx.rxDeployVerticle("sample.Service",
      DeploymentOptions().setConfig(JsonObject().put("port", 3001))).toObservable()

    val s3 = vertx.rxDeployVerticle("sample.Service",
      DeploymentOptions().setConfig(JsonObject().put("port", 3002))).toObservable()

    Observables
      .zip(s1, s2, s3) { a, b, c -> "$a:$b:$c" }
      .subscribeBy(
        onError = { logger.error("Woops", it) },
        onComplete = { logger.info("All 3 verticles are up") })

    vertx.createHttpServer()
      .requestHandler(this::findBestBid)
      .rxListen(8080)
      .subscribeBy(
        onSuccess = { startFuture.complete() },
        onError = { startFuture.fail(it) }
      )
  }

  val httpClient: WebClient by lazy {
    WebClient.create(vertx)
  }

  fun findBestBid(request: HttpServerRequest) {

    val q1 = httpClient.get(3000, "localhost", "/")
      .`as`(BodyCodec.jsonObject())
      .rxSend()
      .toObservable()

    val q2 = httpClient.get(3001, "localhost", "/")
      .`as`(BodyCodec.jsonObject())
      .rxSend()
      .toObservable()

    val q3 = httpClient.get(3002, "localhost", "/")
      .`as`(BodyCodec.jsonObject())
      .rxSend()
      .toObservable()

    Observable.merge(q1, q2, q3)
      .timeout(3, TimeUnit.SECONDS)
      .map { it.body() }
      .reduce { acc, next ->
        if (acc.getInteger("bid") > next.getInteger("bid")) {
          next
        } else {
          acc
        }
      }.subscribeBy(
      onSuccess = {
        request.response()
          .putHeader("Content-Type", "application/json")
          .end(it.encode())
      }, onError = {
      logger.error("Could not find a best match", it)
      request.response()
        .setStatusCode(500)
        .end()
    })
  }
}
```

# Coroutine and Vert.x playground

```kotlin
class MainVerticle : CoroutineVerticle() {

  val logger = LoggerFactory.getLogger(MainVerticle::class.java)

  override suspend fun start() {

    awaitEvent<Long> { vertx.setTimer(1000, it) }
    logger.info("1s elapsted")

    launch(vertx.dispatcher()) {
      var i = 0
      while (true) {
        awaitEvent<Long> { vertx.setTimer(1000, it) }
        vertx.eventBus().send("a.b.c", i++)
        logger.info("a.b.c <<< $i")
      }
    }

    val incoming = vertx.eventBus().consumer<Int>("a.b.c").toChannel(vertx)
    for (msg in incoming) {
      logger.info("a.b.c >>> ${msg.body()}")
    }

    awaitResult<HttpServer> {
      vertx.createHttpServer()
        .requestHandler { req ->
          launch(vertx.dispatcher()) {
            for (buffer in req.toChannel(vertx)) {
              println(buffer.toString(Charsets.UTF_8))
            }
            req.response().end("Bye")
          }
        }
        .listen(8080, it)
    }
  }
}
```

# Coroutine playground

## Launch

```kotlin
fun main(args: Array<String>) = runBlocking {

  val jobs = arrayListOf<Job>()

  for (i in 0..10) {
    jobs.add(launch {
      delay(1, TimeUnit.SECONDS)
      println("$i -- ${Thread.currentThread().name}")
      delay(1, TimeUnit.SECONDS)
      println("$i -- ${Thread.currentThread().name}")
    })
  }

  jobs.forEach { it.join() }
}
```

## Channels

```kotlin
fun main(args: Array<String>) = runBlocking {

  val chan = Channel<Int>(10)

  launch {
    val rand = Random()
    while (true) {
      val next = rand.nextInt(50)
      chan.send(next)
      if (next == 42) {
        chan.close()
        return@launch
      }
    }
  }

  for (i in chan) {
    println(i)
  }

  println("Done")
}
```

## Select

```kotlin
fun main(args: Array<String>) = runBlocking {

  val ints = Channel<Int>()
  val stop = Channel<Boolean>()

  val producer = launch {
    var i = 0;
    while (true) {
      ints.send(i++)
      delay(300)
    }
  }

  launch {
    delay(3, TimeUnit.SECONDS)
    stop.send(true)
  }

  while (producer.isActive) {
    select<Unit> {
      ints.onReceive {
        println(">>> $it")
      }
      stop.onReceive {
        println("[stopping]")
        producer.cancelAndJoin()
      }
    }
  }

  println("[done]")
}
```

# Coroutine edge service

```kotlin
class MainVerticle : CoroutineVerticle() {

  val logger = LoggerFactory.getLogger(MainVerticle::class.java)

  override suspend fun start() {

    val s1 = awaitResult<String> {
      vertx.deployVerticle("sample.Service",
        DeploymentOptions().setConfig(JsonObject().put("port", 3000)), it)
    }

    val s2 = awaitResult<String> {
      vertx.deployVerticle("sample.Service",
        DeploymentOptions().setConfig(JsonObject().put("port", 3001)), it)
    }

    val s3 = awaitResult<String> {
      vertx.deployVerticle("sample.Service",
        DeploymentOptions().setConfig(JsonObject().put("port", 3002)), it)
    }

    logger.info("All 3 verticles have been deployed")

    awaitResult<HttpServer> {
      vertx.createHttpServer()
        .requestHandler {
          launch(vertx.dispatcher()) {
            findBestBid(it)
          }
        }
        .listen(8080, it)
    }

    logger.info("Edge service listening on port 8080")
  }

  val httpClient: WebClient by lazy {
    WebClient.create(vertx)
  }

  private suspend fun findBestBid(request: HttpServerRequest) {

    val bidChannel = Channel<JsonObject>()

    for (port in 3000..3002) {
      launch(vertx.dispatcher()) {
        try {
          val bid = awaitResult<HttpResponse<JsonObject>> {
            httpClient.get(port, "localhost", "/")
              .`as`(BodyCodec.jsonObject())
              .send(it)
          }
          bidChannel.send(bid.body())
        } catch (t: Throwable) {
          logger.error("Web client error", t)
          bidChannel.send(JsonObject().put("error", true))
        }
      }
    }

    val offers = listOf<JsonObject>(bidChannel.receive(), bidChannel.receive(), bidChannel.receive())
    val best = offers.minBy { it.getInteger("bid", Integer.MAX_VALUE) }
    request.response()
      .putHeader("Content-Type", "application/json")
      .end(best?.encode())
  }
}
```
