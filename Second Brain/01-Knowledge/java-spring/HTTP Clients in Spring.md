---
created: 2026-04-17
tags:
  - "#type/concept"
  - "#status/draft"
  - "#lang/spring"
  - "#topic/http"
related: "[[RestTemplate in Spring Boot]], [[WebClient in Spring Boot]], [[RestClient in Spring Boot]], [[Feign Client in Spring Boot]]"
---

# HTTP Clients in Spring

## 1. What

Spring cung cấp nhiều abstraction khác nhau để thực hiện HTTP request từ phía server, mỗi cái phù hợp với một mô hình lập trình và ngữ cảnh kiến trúc riêng. Bốn lựa chọn chính là `RestTemplate` (synchronous, legacy), `WebClient` (reactive, non-blocking), `RestClient` (synchronous, modern — Spring 6.1+), và `Feign Client` (declarative, microservices). Hiểu rõ lịch sử và sự khác biệt giữa chúng là nền tảng để chọn đúng công cụ cho đúng bài toán.

---

## 2. Why

Trước khi có các abstraction này, developer phải tự quản lý `HttpURLConnection` hoặc dùng Apache HttpClient thuần túy — vô cùng verbose và dễ mắc lỗi: quên đóng connection, không handle timeout, không tái sử dụng connection pool, phải tự parse JSON bằng tay. Spring dần đưa ra các lớp abstraction để giảm boilerplate, hỗ trợ type-safe deserialization qua `HttpMessageConverter`, và tích hợp với hệ sinh thái Spring (interceptors, error handling, tracing). Theo thời gian, các nhu cầu mới nảy sinh — reactive programming, microservices, declarative style, Virtual Threads — dẫn đến sự ra đời lần lượt của `WebClient` (Spring 5), `RestClient` (Spring 6.1), và sự phổ biến của OpenFeign trong Spring Cloud.

---

## 3. Mental Model

Hãy tưởng tượng bạn là một thợ mộc và cần đóng đinh vào tường:

- `RestTemplate`: Cái búa cũ kỹ từ hộp dụng cụ của cha bạn để lại. Nặng, cán gỗ hơi cùn, nhưng đáng tin cậy và mọi người trong xưởng đều biết dùng. Không ai cập nhật thiết kế nó nữa, nhưng nó vẫn làm được việc cho những codebase đang chạy.

- `WebClient`: Cái máy bắn đinh khí nén. Cực kỳ mạnh — bắn hàng trăm đinh một lúc không cần nghỉ (non-blocking, high concurrency). Nhưng cần học cách vận hành, cần nguồn khí, và nếu bạn chỉ cần đóng một cây đinh thì lắp máy lên còn mất công hơn dùng búa.

- `RestClient`: Cái búa mới được thiết kế lại từ đầu cho năm 2024 — cùng nguyên lý cơ học với búa cũ (blocking), nhưng cán ergonomic, trọng lượng cân bằng, grip tốt hơn. Đây là lựa chọn mặc định cho dự án mới.

- `Feign Client`: Cái robot CNC được lập trình sẵn. Bạn chỉ cần vẽ bản thiết kế (interface + annotations), robot tự đóng đinh theo đúng bản vẽ. Lý tưởng cho môi trường factory (microservices) nơi bạn cần đóng hàng nghìn loại đinh khác nhau theo pattern nhất quán.

---

## 4. Where it fits

Vị trí trong luồng xử lý của ứng dụng:

```
[Incoming Request]
        |
[Controller Layer]
        |
[Service Layer]
        |
        |-- [RestClient / RestTemplate]  -->  External REST API (sync, blocking)
        |
        |-- [WebClient]                  -->  External REST API (async, reactive pipeline)
        |                                -->  Streaming / SSE endpoints
        |
        |-- [Feign Client]               -->  Other Microservices (via Service Discovery)
                                         -->  Spring Cloud LoadBalancer + Circuit Breaker
```

Trong kiến trúc microservices điển hình:

```
[Order Service]
     |-- Feign --> [Inventory Service]  (internal, service discovery)
     |-- Feign --> [User Service]       (internal, service discovery)
     |-- RestClient --> [Payment Gateway API]  (external, hardcoded URL)
     |-- WebClient --> [Notification Stream]   (external, SSE/streaming)
```

---

## 5. When to use

| Client | Dùng khi |
|---|---|
| `RestTemplate` | Maintain codebase cũ (Spring Boot < 3.x); team quen thuộc; không có ngân sách/thời gian migrate |
| `WebClient` | App dùng Spring WebFlux; cần non-blocking trong reactive pipeline; cần streaming response (SSE, chunked); cần fan-out nhiều request song song với `Mono.zip` |
| `RestClient` | Dự án mới với Spring Boot 3.2+; muốn fluent API, synchronous, không cần reactive; kết hợp với Virtual Threads (Java 21+) để đạt throughput cao |
| `Feign Client` | Môi trường microservices với Spring Cloud; cần load balancing, circuit breaker, service discovery tích hợp; muốn contract rõ ràng giữa các team service |

---

## 6. When NOT to use

Không dùng `RestTemplate` cho project mới — nó đã deprecated và sẽ không nhận feature mới. API của nó cũng quá verbose với nhiều overloaded method khó nhớ.

Không dùng `WebClient` nếu app là Spring MVC thuần túy và team chưa quen với reactive programming. Chi phí học Project Reactor (Mono/Flux, operators, backpressure) rất cao. Code dễ trở thành callback hell và khó debug vì stack trace reactive rất khó đọc.

Không dùng `Feign Client` cho external API bên ngoài tổ chức khi không có service discovery. Feign được thiết kế cho internal microservice communication — dùng nó để gọi Stripe API hay Google Maps API sẽ phức tạp hơn cần thiết, không cần đến load balancing hay service registry.

Không dùng `RestClient` trong Spring Boot < 3.2 — class này không tồn tại trong các phiên bản cũ hơn.

---

## 7. Trade-offs

| Client | Pros | Cons |
|---|---|---|
| RestTemplate | Đơn giản, tài liệu phong phú, mọi developer Spring đều biết | Deprecated, blocking, verbose với complex request, không nhận feature mới |
| WebClient | Non-blocking, high throughput, streaming, functional API | Learning curve cao (Reactor), stack trace khó debug, overhead nếu dùng trong MVC app |
| RestClient | Fluent API hiện đại, synchronous, không cần Reactor, tích hợp Virtual Threads tốt | Chỉ từ Spring 6.1+, ecosystem và tài liệu cộng đồng chưa nhiều bằng RestTemplate |
| Feign Client | Declarative, ít code nhất, tích hợp Spring Cloud hoàn hảo, dễ mock trong unit test | Thêm dependency, khó debug proxy, blocking mặc định, phụ thuộc Spring Cloud ecosystem |

---

## 8. Alternatives

| Tool | Mô tả | Khi nào dùng |
|---|---|---|
| `java.net.http.HttpClient` (JDK 11+) | Built-in, không cần dependency | Khi viết thư viện standalone không muốn phụ thuộc Spring |
| `OkHttp` (Square) | Lightweight, phổ biến, HTTP/2 support | Khi cần control thấp hơn mức Spring abstraction |
| `Retrofit` (Square) | Declarative như Feign, phổ biến trong Android | Java backend mà không dùng Spring Cloud |
| `Apache HttpClient 5` | Low-level, full control | Làm underlying HTTP engine cho các abstraction trên |
| `gRPC` | Protocol Buffers, binary, bidirectional streaming | Khi cần latency cực thấp, strong typing, streaming hai chiều giữa services |

---

## 9. How

So sánh cùng một tác vụ (GET một user, POST tạo user) với 4 client:

```java
// =============================================
// RestTemplate (legacy, synchronous)
// =============================================
@Configuration
public class RestTemplateConfig {
    @Bean
    public RestTemplate restTemplate(RestTemplateBuilder builder) {
        return builder
            .rootUri("https://api.example.com")
            .setConnectTimeout(Duration.ofSeconds(3))
            .setReadTimeout(Duration.ofSeconds(5))
            .build();
    }
}

@Service
public class UserServiceRT {
    @Autowired private RestTemplate restTemplate;

    public User getUser(Long id) {
        return restTemplate.getForObject("/users/{id}", User.class, id);
    }

    public User createUser(User user) {
        ResponseEntity<User> response = restTemplate.postForEntity("/users", user, User.class);
        return response.getBody();
    }

    // Generic list — phải dùng exchange + ParameterizedTypeReference
    public List<User> getAllUsers() {
        return restTemplate.exchange(
            "/users",
            HttpMethod.GET,
            null,
            new ParameterizedTypeReference<List<User>>() {}
        ).getBody();
    }
}


// =============================================
// WebClient (reactive, non-blocking)
// =============================================
@Configuration
public class WebClientConfig {
    @Bean
    public WebClient webClient() {
        return WebClient.builder()
            .baseUrl("https://api.example.com")
            .defaultHeader(HttpHeaders.CONTENT_TYPE, MediaType.APPLICATION_JSON_VALUE)
            .build();
    }
}

@Service
public class UserServiceWC {
    @Autowired private WebClient webClient;

    // Trả về Mono — non-blocking
    public Mono<User> getUser(Long id) {
        return webClient.get()
            .uri("/users/{id}", id)
            .retrieve()
            .onStatus(HttpStatusCode::is4xxClientError,
                resp -> Mono.error(new UserNotFoundException(id)))
            .bodyToMono(User.class);
    }

    // Fan-out: gọi 3 user song song
    public Mono<List<User>> getUsersConcurrent(List<Long> ids) {
        List<Mono<User>> monos = ids.stream()
            .map(this::getUser)
            .toList();
        return Mono.zip(monos, results ->
            Arrays.stream(results).map(o -> (User) o).toList());
    }
}


// =============================================
// RestClient (Spring 6.1+, synchronous, modern)
// =============================================
@Configuration
public class RestClientConfig {
    @Bean
    public RestClient restClient(RestClient.Builder builder) {
        return builder
            .baseUrl("https://api.example.com")
            .defaultHeader(HttpHeaders.ACCEPT, MediaType.APPLICATION_JSON_VALUE)
            .build();
    }
}

@Service
public class UserServiceRC {
    @Autowired private RestClient restClient;

    public User getUser(Long id) {
        return restClient.get()
            .uri("/users/{id}", id)
            .retrieve()
            .onStatus(HttpStatusCode::is4xxClientError, (req, resp) -> {
                throw new UserNotFoundException(id);
            })
            .body(User.class);
    }

    public List<User> getAllUsers() {
        return restClient.get()
            .uri("/users")
            .retrieve()
            .body(new ParameterizedTypeReference<List<User>>() {});
    }
}


// =============================================
// Feign Client (declarative, microservices)
// =============================================
@SpringBootApplication
@EnableFeignClients
public class Application { ... }

@FeignClient(name = "user-service", url = "${services.user-service.url}")
public interface UserServiceClient {

    @GetMapping("/users/{id}")
    User getUserById(@PathVariable("id") Long id);

    @PostMapping("/users")
    User createUser(@RequestBody User user);

    @GetMapping("/users")
    List<User> getAllUsers();
}

@Service
public class OrderService {
    @Autowired private UserServiceClient userServiceClient;

    public void processOrder(Long userId) {
        User user = userServiceClient.getUserById(userId); // trông như gọi method nội bộ
        // business logic...
    }
}
```

---

## 10. Production concerns

**Scaling:**
- `RestTemplate` và `RestClient` là blocking — mỗi request chiếm một thread trong suốt thời gian chờ response. Với traffic cao, cần thread pool đủ lớn hoặc kết hợp với Virtual Threads (Java 21+). Với tải rất cao (>1000 concurrent), cân nhắc `WebClient`.
- `WebClient` cho phép xử lý hàng nghìn concurrent request với event loop nhỏ (mặc định số CPU cores * 2 threads Netty). Không bị bottleneck bởi thread pool.
- `Feign Client` là blocking theo mặc định. Với Spring Boot 3.2+ và Virtual Threads, overhead này giảm đáng kể.

**Failure modes:**
- Không cấu hình timeout: request bị treo vô thời hạn, thread bị giữ, dẫn đến thread pool exhaustion và cascading failure toàn hệ thống.
- Không dùng circuit breaker: khi downstream service chết hoặc chậm, upstream service tiếp tục gửi request, tiêu tốn tài nguyên vô ích.
- Không retry với backoff: lỗi thoáng qua (transient failure) có thể được giải quyết bằng retry, nhưng retry không có backoff sẽ làm quá tải service đang gặp vấn đề.

**Monitoring:**
- Tích hợp Micrometer để track `http.client.requests` metrics (latency, status codes, uri).
- Dùng Spring Boot Actuator với endpoint `/actuator/metrics/http.client.requests`.
- Bật distributed tracing với Micrometer Tracing (Brave/Zipkin hoặc OpenTelemetry) để trace request xuyên qua các service.
- Log request/response bằng custom interceptor — cẩn thận với sensitive data (PII, token), không nên log ở production ở level FULL.

---

## 11. Common mistakes

- Mistake: Tạo mới instance `RestTemplate` hoặc `WebClient` bằng `new` trong mỗi method thay vì inject Bean.
  Fix: Khai báo là `@Bean` và inject, để tận dụng connection pool được share và cấu hình chung (base URL, default headers, timeout).

- Mistake: Dùng `RestTemplate` cho project mới Spring Boot 3.2+ chỉ vì "quen tay" hoặc copy từ tutorial cũ.
  Fix: Dùng `RestClient` — API fluent tương tự, nhưng hiện đại, được support đầy đủ, và dễ migrate lên `WebClient` sau này nếu cần.

- Mistake: Block `Mono`/`Flux` của `WebClient` bằng `.block()` bên trong một reactive pipeline (trong method có return type là `Mono`/`Flux`).
  Fix: Trả về `Mono`/`Flux` xuyên suốt pipeline. Chỉ dùng `.block()` ở test, ở `main()`, hoặc ở non-reactive thread rõ ràng. Gọi `.block()` trên Netty event loop thread sẽ throw `IllegalStateException`.

- Mistake: Không cấu hình timeout cho Feign Client, đặc biệt là `readTimeout`. Mặc định của Feign là rất ngắn (1 giây read timeout) dẫn đến lỗi giả với service chậm, hoặc không set timeout dẫn đến treo.
  Fix: Cấu hình rõ ràng trong `application.yml`:
  ```yaml
  feign:
    client:
      config:
        default:
          connect-timeout: 3000
          read-timeout: 10000
  ```

---

## 12. Sample project

**Tên project: Multi-Client Weather Aggregator**

Xây dựng một Spring Boot 3.2+ REST service tổng hợp dữ liệu thời tiết từ 3 provider khác nhau và trả về bản so sánh.

**Constraint bắt buộc:**
Service phải triển khai đúng 3 client:
1. `RestClient` gọi OpenWeatherMap API (synchronous, vì đây là primary source).
2. `WebClient` gọi WeatherAPI.com (async, để fetch song song với client 1 bằng `Mono.zip`).
3. `Feign Client` gọi internal `weather-cache-service` (internal microservice cung cấp cached forecast).

Endpoint duy nhất `GET /weather/compare?city=Hanoi` phải:
- Trả về JSON tổng hợp với kết quả từ cả 3 provider.
- Ghi lại latency (ms) của từng call trong response.
- Nếu bất kỳ provider nào fail, vẫn trả về kết quả từ các provider còn lại với trường `error` mô tả lỗi của provider thất bại — không được trả về 500 cho toàn bộ request.

---

## 13. Interview

**Core Q&A:**

Q: Sự khác biệt chính giữa `RestTemplate` và `RestClient` là gì?
A: Cả hai đều synchronous và blocking, sử dụng chung infrastructure bên dưới (`ClientHttpRequestFactory`, `HttpMessageConverter`). Điểm khác biệt là API: `RestClient` có fluent builder API (giống `WebClient`) dễ đọc, compose và xử lý lỗi hơn. `RestTemplate` có nhiều overloaded method khó nhớ. `RestClient` là lựa chọn được Spring khuyến nghị từ 6.1, `RestTemplate` đã deprecated.

Q: Tại sao `WebClient` có thể hoạt động trong app Spring MVC không phải reactive?
A: `WebClient` chỉ cần `spring-webflux` dependency. Nó không yêu cầu toàn bộ app phải chạy trên Netty event loop. Trong MVC context, bạn có thể gọi `.block()` để lấy kết quả synchronously — thread đó vốn đã là dedicated servlet thread nên block là chấp nhận được. Hoặc dùng WebClient để fan-out nhiều request song song rồi `.block()` ở cuối chờ kết quả.

Q: Feign Client hoạt động như thế nào dưới nền (under the hood)?
A: Spring Cloud dùng JDK Dynamic Proxy để tạo implementation từ interface annotated lúc startup. Khi một method được gọi, proxy phân tích annotations (`@GetMapping`, `@PathVariable`, v.v.), xây dựng HTTP request, gửi đi qua underlying HTTP client (mặc định là JDK HttpURLConnection hoặc Apache HttpClient nếu có trên classpath), và deserialize response về kiểu dữ liệu khai báo.

Q: Khi nào nên dùng `WebClient` thay vì `RestClient` trong app Spring MVC thuần túy?
A: Khi cần: (1) gọi nhiều API song song và tổng hợp kết quả — dùng `Mono.zip` hiệu quả hơn tạo nhiều thread; (2) streaming response (SSE, chunked transfer); (3) timeout per request với reactive timeout operators. Nếu không có nhu cầu đặc biệt này, `RestClient` với Virtual Threads là đủ và đơn giản hơn nhiều.

Q: `RestTemplate` có bị xóa khỏi Spring không?
A: Không bị xóa, chỉ "deprecated for maintenance" — không nhận feature mới nhưng vẫn được giữ lại để backward compatibility. Spring team không có kế hoạch remove trong tương lai gần. Codebase hiện tại dùng `RestTemplate` không bị broken, nhưng project mới không nên dùng nó.

**Scenario:**

Scenario: Bạn đang build một Spring Boot 3.2 service cần gọi external payment gateway để xử lý thanh toán. Bạn chọn client nào và tại sao?
Answer: `RestClient`. Lý do: (1) Payment flow cần sequential steps — charge, verify, confirm — blocking là phù hợp với logic này; (2) `RestClient` API fluent dễ đọc và xử lý error per status code rõ ràng; (3) Không cần Spring Cloud, không cần reactive. Thêm retry với Spring Retry cho transient failure và Resilience4j circuit breaker để protect resource nếu gateway không ổn định.

Scenario: Team của bạn có 8 microservices, mỗi service gọi 3-4 service khác. Khó maintain URL hardcode và cấu hình riêng ở từng service. Bạn đề xuất gì?
Answer: `Feign Client` với Spring Cloud. Định nghĩa interface per service, đăng ký tên service với Eureka. Feign + Spring Cloud LoadBalancer tự động resolve và phân tải. Tạo shared library chứa Feign interfaces để các team service cùng dùng, đảm bảo contract nhất quán.

Scenario: App Spring MVC đang bị thread pool exhaustion. Bạn thấy nhiều thread đang blocked chờ response từ external API recommendation engine (avg 800ms latency). Bạn xử lý thế nào?
Answer: Có hai hướng: (1) Ngắn hạn: thêm circuit breaker (Resilience4j) để fail fast khi service chậm, tránh giữ thread vô ích; tăng thread pool với Virtual Threads. (2) Dài hạn: migrate call đó sang `WebClient` và trả về `Mono` từ controller — toàn bộ request handling sẽ non-blocking cho endpoint đó, giải phóng thread trong khi chờ recommendation engine. Hoặc migrate toàn app sang Spring WebFlux nếu đây là bottleneck phổ biến.

---

## 14. References

- Spring Framework Docs - REST Clients: https://docs.spring.io/spring-framework/reference/integration/rest-clients.html
- Spring Framework Docs - WebClient: https://docs.spring.io/spring-framework/reference/web/webflux-webclient.html
- Spring Cloud OpenFeign Docs: https://docs.spring.io/spring-cloud-openfeign/docs/current/reference/html/
- Spring Blog - Introducing RestClient in Spring 6.1: https://spring.io/blog/2023/07/13/new-in-spring-6-1-restclient
- RestTemplate JavaDoc (deprecated notice): https://docs.spring.io/spring-framework/docs/current/javadoc-api/org/springframework/web/client/RestTemplate.html
- Spring Boot Reference - Calling REST Services: https://docs.spring.io/spring-boot/docs/current/reference/html/io.html#io.rest-client

---

## 15. Real-world Code

- Spring PetClinic Microservices — Feign Client usage between services: https://github.com/spring-petclinic/spring-petclinic-microservices
- Spring Boot Official Samples — WebClient demo: https://github.com/spring-projects/spring-boot/tree/main/spring-boot-samples/spring-boot-sample-webflux
- Baeldung Spring tutorials — WebClient và RestClient examples: https://github.com/eugenp/tutorials/tree/master/spring-5-reactive-client
- Spring Cloud Samples — Feign with Eureka: https://github.com/spring-cloud-samples/feign-eureka
- Real-world RestClient usage in Spring initializr source: https://github.com/spring-io/start.spring.io

---

## 16. Community

- Stack Overflow — "Difference between RestTemplate, WebClient and RestClient": https://stackoverflow.com/questions/47974757/webclient-vs-resttemplate
- Stack Overflow — "Should I use OpenFeign or RestTemplate for microservices": https://stackoverflow.com/questions/43821515/feign-vs-resttemplate
- Baeldung — Spring WebClient vs RestTemplate: https://www.baeldung.com/spring-webclient-resttemplate
- Baeldung — Introduction to Spring 6 RestClient: https://www.baeldung.com/spring-6-restclient
- Reddit r/SpringBoot — Discussion "RestClient vs WebClient for new non-reactive projects": https://www.reddit.com/r/SpringBoot/comments/restclient_webclient_discussion/
- InfoQ — "Spring 6.1 RestClient: A Modern HTTP Client for Spring MVC": https://www.infoq.com/articles/spring-6-restclient/
- Philipp Hauer's Blog — Comparison of Spring HTTP clients: https://phauer.com/2019/modern-best-practices-spring-resttemplate/
