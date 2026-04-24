---
created: 2026-04-17
tags:
  - "#type/library"
  - "#status/draft"
  - "#lang/spring"
  - "#topic/http"
  - "#topic/async"
related:
  - "[[HTTP Clients in Spring]]"
  - "[[RestTemplate in Spring Boot]]"
  - "[[RestClient in Spring Boot]]"
  - "[[ExchangeStrategies in WebClient]]"
---

# WebClient in Spring Boot

## 1. What

`WebClient` là HTTP client hiện đại, non-blocking và reactive của Spring Framework, thuộc module `spring-webflux`. Nó trả về `Mono<T>` hoặc `Flux<T>` từ Project Reactor thay vì block thread để chờ response. `WebClient` có thể được dùng trong cả ứng dụng Spring WebFlux (reactive) lẫn Spring MVC (servlet-based) — trong MVC context, bạn có thể gọi `.block()` để lấy kết quả synchronously hoặc dùng nó để fan-out nhiều request song song.

---

## 2. Why

`RestTemplate` xử lý HTTP theo mô hình one-thread-per-request: mỗi request chiếm một thread trong suốt thời gian chờ response từ server. Ở latency thấp điều này ổn, nhưng ở latency cao (ví dụ gọi external API với average 200ms) và concurrency cao (ví dụ 1000 concurrent users), thread pool bị cạn kiệt:

```
1000 concurrent requests x 200ms wait time
= 200 threads bị blocked tại một thời điểm
= Thread pool exhaustion nếu max pool = 200
= Request tiếp theo bị reject hoặc phải queue
```

`WebClient` dùng event loop (Netty NIO): một lượng nhỏ thread (mặc định = số CPU cores * 2) xử lý hàng nghìn concurrent connections. Thread không bị block trong khi chờ — nó đăng ký callback và tiếp tục xử lý connection khác. Khi response về, callback được gọi, data được xử lý. Đây là mô hình của Node.js, Nginx — cho phép throughput cực cao với ít resource.

---

## 3. Mental Model

Hãy tưởng tượng bạn quản lý một tổng đài cuộc gọi:

- `RestTemplate` (blocking): Mỗi nhân viên chỉ xử lý một cuộc gọi tại một thời điểm. Khi khách hàng đang chờ tra thông tin (external API latency), nhân viên đó ngồi yên chờ — không làm gì cả. Muốn xử lý 100 cuộc gọi đồng thời, bạn cần 100 nhân viên (thread).

- `WebClient` (non-blocking): Mỗi nhân viên có thể xử lý nhiều cuộc gọi đồng thời. Khi một khách hàng chờ tra thông tin, nhân viên đặt khách hàng đó vào chế độ "hold" (registered callback), và chuyển sang phục vụ khách khác. Khi thông tin tra xong, tổng đài tự động báo lại và nhân viên hoàn tất cuộc gọi đó. Chỉ cần 4-8 nhân viên (event loop threads) xử lý được hàng nghìn cuộc gọi.

Thẻ rung khi đặt đồ ăn tại nhà hàng cũng là một hình ảnh tốt: bạn nhận thẻ (`Mono`), tự do làm việc khác, thẻ rung khi món sẵn sàng (`subscribe` được gọi). Không ai trong hàng phải đứng chờ tại quầy.

---

## 4. Where it fits

```
[Spring WebFlux App]                    [Spring MVC App]
        |                                       |
[WebClient]                            [WebClient (with .block())]
        |                                       |
[Project Reactor Pipeline]             [Mono.zip for fan-out]
        |                                       |
[Netty Event Loop]                     [Servlet Thread]
        |                                       |
[Non-blocking NIO]                     [Blocks only at .block()]
        |                                       |
[External API]                         [External API]
```

Trong reactive pipeline:

```
[Controller: returns Mono<ResponseBody>]
        |
[Service: returns Mono<User>]
        |
[WebClient: returns Mono<User>]
        |
[Netty NIO: async HTTP, no thread block]
        |
[External API Response]
        |
[Operators: map, flatMap, filter, onErrorResume]
        |
[Subscriber: framework auto-subscribes when request comes in]
```

---

## 5. When to use

- Trong ứng dụng Spring WebFlux (reactive): `WebClient` là lựa chọn duy nhất đúng đắn — dùng `RestTemplate` hoặc `RestClient` sẽ block event loop thread.
- Khi cần fan-out nhiều request song song: `Mono.zip()` cho phép gọi 3-5 API cùng lúc và combine kết quả, hiệu quả hơn nhiều so với sequential blocking calls.
- Khi cần streaming response: Server-Sent Events (SSE), chunked transfer, long-polling — `Flux<T>` xử lý data stream tự nhiên.
- Khi cần non-blocking timeout: `.timeout(Duration.ofSeconds(3))` không block thread mà dùng Reactor scheduler.
- Trong Spring MVC app khi một endpoint cụ thể cần call nhiều downstream APIs song song để giảm latency (fan-out pattern).

---

## 6. When NOT to use

Không dùng `WebClient` nếu team chưa hiểu Project Reactor. Reactive programming có learning curve dốc: operators như `flatMap`, `switchIfEmpty`, `zipWith`, error propagation, backpressure — tất cả đều có behavior khác với imperative code. Code sai dễ gây data không arrive, lỗi im lặng, hay memory leak.

Không block `WebClient` bằng `.block()` bên trong reactive context (khi thread đang chạy trên Netty event loop). Điều này throws `IllegalStateException: block()/blockFirst()/blockLast() are blocking, which is not supported in thread nio-*`. Đây là lỗi runtime, không phải compile-time.

Không dùng `WebClient` chỉ vì "reactive is better" nếu app là Spring MVC đơn giản và không có bottleneck. Overhead của reactive programming (context switching, operator chains, debugging) không đáng nếu chỉ cần gọi 1-2 synchronous API. Dùng `RestClient` thay thế.

Không dùng trong ứng dụng cần đơn giản về error handling và debugging: reactive stack traces rất khó đọc so với imperative. Một lỗi trong `flatMap` của `WebClient` có thể có stack trace dài 50 dòng với các Reactor internals không liên quan.

---

## 7. Trade-offs

| Pros | Cons |
|---|---|
| Non-blocking — high throughput với ít thread | Learning curve dốc: cần hiểu Reactor (Mono, Flux, operators) |
| Fan-out nhiều request song song tự nhiên | Stack trace khó đọc, khó debug khi có lỗi |
| Hỗ trợ streaming response (Flux) | Dùng `.block()` sai chỗ gây deadlock hoặc exception |
| Fluent API dễ đọc và compose | Cần thêm dependency `spring-boot-starter-webflux` |
| Tích hợp tốt với Micrometer Observation | Operator chaining phức tạp khi logic business logic phức tạp |
| Có thể dùng trong cả MVC và WebFlux app | Memory model và subscription lifecycle cần hiểu rõ để tránh leak |

---

## 8. Alternatives

| Option | So sánh |
|---|---|
| `RestClient` | Cùng fluent API nhưng synchronous — không cần Reactor, đơn giản hơn nhiều cho non-reactive app |
| `RestTemplate` | Blocking, legacy, nhưng đơn giản và quen thuộc |
| `java.net.http.HttpClient` (JDK 11+) | Built-in async support, nhưng không tích hợp Reactor và Spring ecosystem |
| `Retrofit` với `RxJava` | Reactive cho non-Spring context |
| `Vert.x WebClient` | Alternative reactive framework — không tích hợp Spring |

---

## 9. How

```java
// =============================================
// 1. Cấu hình WebClient Bean (Production-ready)
// =============================================
@Configuration
public class WebClientConfig {

    @Bean
    public WebClient webClient() {
        // Cấu hình Netty HttpClient với timeout và connection pool
        HttpClient httpClient = HttpClient.create()
            .option(ChannelOption.CONNECT_TIMEOUT_MILLIS, 3000)  // connect timeout
            .responseTimeout(Duration.ofSeconds(10))             // response timeout
            .doOnConnected(conn -> conn
                .addHandlerLast(new ReadTimeoutHandler(10, TimeUnit.SECONDS))
                .addHandlerLast(new WriteTimeoutHandler(3, TimeUnit.SECONDS)));

        return WebClient.builder()
            .baseUrl("https://api.example.com")
            .clientConnector(new ReactorClientHttpConnector(httpClient))
            .defaultHeader(HttpHeaders.CONTENT_TYPE, MediaType.APPLICATION_JSON_VALUE)
            .defaultHeader(HttpHeaders.ACCEPT, MediaType.APPLICATION_JSON_VALUE)
            .filter(logRequest())   // ExchangeFilterFunction để log
            .build();
    }

    // ExchangeFilterFunction — tương đương với Interceptor trong RestTemplate
    private ExchangeFilterFunction logRequest() {
        return ExchangeFilterFunction.ofRequestProcessor(clientRequest -> {
            log.info("HTTP {} {}", clientRequest.method(), clientRequest.url());
            return Mono.just(clientRequest);
        });
    }
}


// =============================================
// 2. GET request — cơ bản
// =============================================
@Service
public class UserService {

    @Autowired
    private WebClient webClient;

    // Trả về Mono — non-blocking, gọi subscribe hoặc return lên controller
    public Mono<User> getUser(Long id) {
        return webClient.get()
            .uri("/users/{id}", id)
            .retrieve()
            .onStatus(HttpStatusCode::is4xxClientError,
                response -> response.bodyToMono(ErrorResponse.class)
                    .flatMap(err -> Mono.error(new UserNotFoundException(id, err.getMessage()))))
            .onStatus(HttpStatusCode::is5xxServerError,
                response -> Mono.error(new ExternalServiceException("User service error")))
            .bodyToMono(User.class);
    }

    // GET list — Flux cho streaming, hoặc collectList() nếu cần List
    public Flux<User> getAllUsers() {
        return webClient.get()
            .uri("/users")
            .retrieve()
            .bodyToFlux(User.class);
    }

    // GET với query parameters
    public Mono<PagedResponse<User>> getUsersPaged(int page, int size) {
        return webClient.get()
            .uri(uriBuilder -> uriBuilder
                .path("/users")
                .queryParam("page", page)
                .queryParam("size", size)
                .build())
            .retrieve()
            .bodyToMono(new ParameterizedTypeReference<PagedResponse<User>>() {});
    }
}


// =============================================
// 3. POST request
// =============================================
public Mono<User> createUser(User user) {
    return webClient.post()
        .uri("/users")
        .contentType(MediaType.APPLICATION_JSON)
        .bodyValue(user)
        .retrieve()
        .onStatus(status -> status.value() == 409,
            resp -> Mono.error(new DuplicateUserException("User already exists")))
        .bodyToMono(User.class);
}


// =============================================
// 4. Fan-out — gọi nhiều API song song (quan trọng!)
// =============================================
public Mono<DashboardData> getDashboard(Long userId) {
    Mono<User> userMono = getUser(userId);
    Mono<List<Order>> ordersMono = getOrders(userId);
    Mono<List<Notification>> notificationsMono = getNotifications(userId);

    // Mono.zip — chạy 3 call SONG SONG, kết hợp khi tất cả hoàn thành
    return Mono.zip(userMono, ordersMono, notificationsMono)
        .map(tuple -> new DashboardData(
            tuple.getT1(),  // User
            tuple.getT2(),  // List<Order>
            tuple.getT3()   // List<Notification>
        ));
    // Nếu không có Mono.zip, 3 calls này sẽ là sequential:
    // Total latency = latency1 + latency2 + latency3
    // Với Mono.zip: Total latency = max(latency1, latency2, latency3)
}


// =============================================
// 5. Error handling và fallback
// =============================================
public Mono<User> getUserWithFallback(Long id) {
    return webClient.get()
        .uri("/users/{id}", id)
        .retrieve()
        .bodyToMono(User.class)
        .timeout(Duration.ofSeconds(3))        // timeout operator
        .onErrorReturn(TimeoutException.class, User.getDefaultUser())  // fallback khi timeout
        .onErrorResume(WebClientResponseException.NotFound.class,
            ex -> Mono.just(User.getGuestUser()))  // fallback khi 404
        .onErrorResume(ex -> {                     // fallback tổng quát
            log.error("Failed to get user {}: {}", id, ex.getMessage());
            return Mono.error(new ServiceException("Failed to fetch user"));
        });
}


// =============================================
// 6. Retry với exponential backoff
// =============================================
public Mono<User> getUserWithRetry(Long id) {
    return webClient.get()
        .uri("/users/{id}", id)
        .retrieve()
        .bodyToMono(User.class)
        .retryWhen(Retry.backoff(3, Duration.ofMillis(500))  // 3 lần retry, bắt đầu 500ms
            .maxBackoff(Duration.ofSeconds(5))               // tối đa 5 giây
            .filter(ex -> ex instanceof WebClientRequestException)  // chỉ retry network error
            .onRetryExhaustedThrow((spec, signal) ->
                new ExternalServiceException("Service unavailable after retries")));
}


// =============================================
// 7. Dùng WebClient trong Spring MVC (blocking mode)
// =============================================
@RestController
@RequestMapping("/mvc")
public class MvcController {

    @Autowired private WebClient webClient;

    // Scenario: Spring MVC app, cần fan-out 3 calls song song
    @GetMapping("/dashboard/{userId}")
    public DashboardData getDashboard(@PathVariable Long userId) {
        // Tạo 3 Mono TRƯỚC (không block ngay)
        Mono<User> userMono = webClient.get().uri("/users/{id}", userId)
            .retrieve().bodyToMono(User.class);
        Mono<List<Order>> ordersMono = webClient.get().uri("/orders?userId={id}", userId)
            .retrieve().bodyToFlux(Order.class).collectList();
        Mono<Inventory> inventoryMono = webClient.get().uri("/inventory/{id}", userId)
            .retrieve().bodyToMono(Inventory.class);

        // Zip rồi mới block một lần — 3 calls chạy song song
        return Mono.zip(userMono, ordersMono, inventoryMono)
            .map(t -> new DashboardData(t.getT1(), t.getT2(), t.getT3()))
            .block(Duration.ofSeconds(10));  // block ở đây là OK vì đây là servlet thread
    }
}


// =============================================
// 8. Server-Sent Events streaming với Flux
// =============================================
@GetMapping(value = "/events", produces = MediaType.TEXT_EVENT_STREAM_VALUE)
public Flux<StockPrice> streamPrices() {
    return webClient.get()
        .uri("https://stream.example.com/prices")
        .accept(MediaType.TEXT_EVENT_STREAM)
        .retrieve()
        .bodyToFlux(StockPrice.class)
        .doOnNext(price -> log.debug("Received: {}", price))
        .onErrorResume(ex -> {
            log.error("Stream error", ex);
            return Flux.empty();
        });
}
```

---

## 10. Production concerns

**Netty Connection Pool:**
`WebClient` dùng Netty connection pool mặc định: max connections tổng = 500, max connections per route = 1000. Trong production với nhiều downstream service, cần cấu hình riêng để tránh một service chiếm hết pool:

```java
ConnectionProvider provider = ConnectionProvider.builder("custom-pool")
    .maxConnections(100)
    .maxIdleTime(Duration.ofSeconds(20))
    .maxLifeTime(Duration.ofSeconds(60))
    .pendingAcquireTimeout(Duration.ofSeconds(5))
    .evictInBackground(Duration.ofSeconds(30))
    .build();

HttpClient httpClient = HttpClient.create(provider)
    .responseTimeout(Duration.ofSeconds(10));
```

**Failure modes:**
- Blocking event loop thread: gọi `.block()` từ Netty event loop thread (thread có tên `reactor-http-nio-*`) sẽ deadlock toàn bộ server. Cần nhận diện pattern này và dùng `subscribeOn(Schedulers.boundedElastic())` để offload sang dedicated thread nếu cần block.
- Subscription không bao giờ xảy ra: `Mono`/`Flux` là lazy — nếu không có ai subscribe, HTTP request không bao giờ được gửi. Đây là lỗi phổ biến khi bắt đầu học Reactor.
- Memory leak với `exchangeToMono()`: nếu dùng `exchangeToMono()` và không consume response body, connection bị giữ không được trả về pool.

**Monitoring:**
- Bật Micrometer Observation integration: `WebClient.builder().observationRegistry(observationRegistry)`. Tự động record `http.client.requests` metrics với labels: uri, method, status, outcome.
- Dùng `.log()` operator để debug reactive pipeline trong development: `bodyToMono(User.class).log("webclient.user")`.
- Bật Reactor debug agent trong development để cải thiện stack trace: `Hooks.onOperatorDebug()` (tốn performance, chỉ dùng khi debug).

---

## 11. Common mistakes

- Mistake: Gọi `.block()` bên trong method có return type là `Mono` hoặc `Flux`, hoặc trong code đang chạy trên Netty thread.
  Fix: Trả về `Mono`/`Flux` xuyên suốt pipeline. Framework sẽ subscribe khi cần. Nếu thực sự cần blocking, offload bằng `.subscribeOn(Schedulers.boundedElastic())` trước khi `.block()`.

- Mistake: Không xử lý lỗi (onError) trong reactive chain. Lỗi từ downstream service làm unhandled exception trôi vào Reactor và có thể bị log mà không propagate đúng lên caller.
  Fix: Luôn thêm `onErrorResume()` hoặc `onErrorReturn()` ở cuối chain để handle gracefully. Ít nhất log lỗi với context (userId, requestId) và re-throw business exception.

- Mistake: Tạo `WebClient` mới với `WebClient.create()` trong mỗi method call.
  Fix: Khai báo `WebClient` là `@Bean` (singleton), tái sử dụng connection pool. `WebClient` instance là immutable và thread-safe sau khi tạo.

- Mistake: Không set timeout — `WebClient` mặc định không có global response timeout.
  Fix: Set `responseTimeout` ở Netty `HttpClient` level (global), và/hoặc dùng `.timeout(Duration)` operator per-request nếu cần fine-grained control.

- Mistake: Dùng `flatMap` khi chỉ cần `map` — `flatMap` dành cho operations trả về `Mono`/`Flux`, còn `map` dành cho synchronous transformation.
  Fix: Hiểu rõ: `map(user -> user.getName())` — sync transform. `flatMap(user -> repo.findById(user.getId()))` — async transform returning Mono. Nhầm có thể gây compilation error hoặc logic sai.

---

## 12. Sample project

**Tên project: Reactive Crypto Price Aggregator**

Xây dựng một Spring WebFlux service expose endpoint `GET /crypto/summary/{symbol}` (ví dụ: BTC, ETH) trả về tóm tắt thông tin từ nhiều nguồn.

**Constraint bắt buộc:**
- Phải gọi đúng 4 exchange APIs (CoinGecko, CoinMarketCap, Binance, Kraken) SONG SONG dùng `Mono.zip` — không được sequential.
- Nếu bất kỳ exchange nào không phản hồi trong 2 giây, phải timeout riêng exchange đó và thay bằng giá trị `null` trong kết quả — không được làm fail toàn bộ request.
- Phải implement rate limiting: không gọi quá 10 request/giây tổng cộng đến CoinGecko (dùng Resilience4j RateLimiter wrapped trong reactive operator).
- Endpoint phải hỗ trợ Server-Sent Events: `GET /crypto/stream/{symbol}` push giá realtime mỗi 5 giây dùng `Flux` kết hợp `Flux.interval`.
- Phải viết test với `MockWebServer` (OkHttp) để mock external APIs, không được gọi API thật trong test.

---

## 13. Interview

**Core Q&A:**

Q: Tại sao `WebClient` scalable hơn `RestTemplate`?
A: `RestTemplate` dùng blocking I/O: mỗi request giữ một thread trong toàn bộ thời gian chờ response. Với 1000 concurrent requests, cần 1000 threads. `WebClient` dùng non-blocking NIO qua Netty: thread không bị block mà đăng ký callback, được giải phóng để xử lý request khác trong khi chờ. Với 1000 concurrent requests, chỉ cần 4-8 event loop threads. Điều này reduce memory footprint (mỗi thread tốn ~1MB stack) và context switching overhead đáng kể.

Q: Sự khác biệt giữa `retrieve()` và `exchangeToMono()`?
A: `retrieve()` là high-level API: tự động throw `WebClientResponseException` cho 4xx/5xx, tự động consume và release response body. `exchangeToMono()` là low-level: bạn nhận `ClientResponse` và tự xử lý status, headers, body — kể cả release resource. Dùng `retrieve()` trong hầu hết cases; dùng `exchangeToMono()` khi cần đọc custom headers hoặc xử lý response body conditional dựa trên status code.

Q: Làm thế nào để truyền context (như request ID) qua reactive chain?
A: Dùng `Context` của Project Reactor — tương tự `ThreadLocal` nhưng cho reactive. Gắn context vào chain bằng `.contextWrite(Context.of("requestId", requestId))`, đọc bằng `Mono.deferContextual(ctx -> ...)`. Trong WebClient, dùng `ExchangeFilterFunction` để đọc context và gắn vào header.

Q: `Mono.zip` vs `Mono.flatMap` — khi nào dùng cái nào?
A: `Mono.zip`: combine kết quả của nhiều Mono chạy SONG SONG, tất cả phải thành công. Dùng cho fan-out. `flatMap`: chuyển đổi kết quả của một Mono thành Mono khác TUẦN TỰ (kết quả của cái trước làm input cho cái sau). Dùng khi gọi hai API mà call thứ hai phụ thuộc vào kết quả của call đầu tiên.

Q: Làm thế nào để retry với backoff trong WebClient?
A: Dùng `.retryWhen(Retry.backoff(maxAttempts, firstBackoff))`. Tham số `filter` để chỉ retry một số loại exception (ví dụ: chỉ retry network error, không retry 4xx). `maxBackoff` để giới hạn thời gian chờ tối đa. `jitter(0.5)` để thêm random delay tránh thundering herd problem.

**Scenario:**

Scenario: Bạn có endpoint Spring WebFlux cần gọi UserService (200ms avg) và OrderService (300ms avg). Nếu gọi tuần tự, total latency là 500ms. Làm sao giảm xuống còn ~300ms?
Answer: Dùng `Mono.zip` để call song song: `Mono.zip(userServiceClient.getUser(id), orderServiceClient.getOrders(id)).map(t -> new Response(t.getT1(), t.getT2()))`. Hai Mono được subscribe đồng thời, total latency = max(200ms, 300ms) = 300ms.

Scenario: Bạn gặp lỗi "block()/blockFirst()/blockLast() are blocking, which is not supported in thread reactor-http-nio-4" trong production. Nguyên nhân và cách fix?
Answer: Đang gọi `.block()` từ Netty event loop thread — thường xảy ra khi inject service A vào service B và service A gọi WebClient, service B gọi `.block()` trên Mono trả về. Fix: (1) Thay `.block()` bằng cách return `Mono` lên controller và để framework subscribe. (2) Nếu bắt buộc phải block, dùng `.subscribeOn(Schedulers.boundedElastic())` để offload sang thread pool cho phép blocking. (3) Review lại architecture để tránh mixing blocking và reactive code.

Scenario: Bạn cần gọi 5 external APIs song song và muốn lấy kết quả từ 3 cái nhanh nhất, hủy 2 cái chậm. Bạn dùng operator nào?
Answer: `Flux.merge()` kết hợp với `.take(3)`: `Flux.merge(call1, call2, call3, call4, call5).take(3).collectList()`. `Flux.merge()` subscribe tất cả 5 Mono/Flux cùng lúc và emit kết quả ngay khi arrive. `.take(3)` lấy 3 cái đầu tiên và cancel các subscription còn lại. Đây là "first N wins" pattern.

---

## 14. References

- Spring Framework Docs - WebClient: https://docs.spring.io/spring-framework/reference/web/webflux-webclient.html
- Project Reactor Reference Guide: https://projectreactor.io/docs/core/release/reference/
- Spring Framework Docs - Testing WebClient with MockWebServer: https://docs.spring.io/spring-framework/reference/testing/spring-mvc-test-client.html
- Netty Connection Provider Configuration: https://projectreactor.io/docs/netty/release/reference/index.html#_connection_pool
- Spring Blog - WebClient in Spring MVC context: https://spring.io/blog/2023/03/28/reactive-webclient-in-spring-mvc
- Micrometer Observation with WebClient: https://micrometer.io/docs/observation

---

## 15. Real-world Code

- Spring Framework WebClient tests — nhiều pattern thực tế: https://github.com/spring-projects/spring-framework/tree/main/spring-webflux/src/test/java/org/springframework/web/reactive/function/client
- Baeldung WebClient tutorials: https://github.com/eugenp/tutorials/tree/master/spring-5-reactive-client
- Spring Boot WebFlux sample — sử dụng WebClient: https://github.com/spring-projects/spring-boot/tree/main/spring-boot-tests/spring-boot-integration-tests/spring-boot-server-tests
- RSocket với WebClient — streaming use case: https://github.com/spring-attic/spring-rsocket-demo
- Spring Petclinic reactive version: https://github.com/spring-petclinic/spring-petclinic-reactive

---

## 16. Community

- Stack Overflow — "Block() in non-reactive context": https://stackoverflow.com/questions/57373011/block-ing-in-spring-webflux
- Stack Overflow — "WebClient vs RestTemplate performance": https://stackoverflow.com/questions/47974757/webclient-vs-resttemplate
- Baeldung — Guide to Spring 5 WebClient: https://www.baeldung.com/spring-5-webclient
- Baeldung — WebClient Filters: https://www.baeldung.com/spring-webflux-filters
- Reddit r/SpringBoot — "When to use WebClient in a Spring MVC app": https://www.reddit.com/r/SpringBoot/comments/webclient_in_mvc/
- Stephane Maldini (Reactor author) on reactive principles: https://medium.com/@smaldini/reactor-core-3-0-go-ga-reactor-net-3-0-m2-1b08b8b43acd
- Simon Baslé's blog on Reactor patterns: https://simonbasle.github.io/
