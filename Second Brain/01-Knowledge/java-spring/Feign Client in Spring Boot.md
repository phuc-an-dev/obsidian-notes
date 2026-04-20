---
created: 2026-04-17
tags:
  - "#type/library"
  - "#status/draft"
  - "#lang/spring"
  - "#topic/http"
related:
  - "[[HTTP Clients in Spring]]"
  - "[[WebClient in Spring Boot]]"
---

# Feign Client in Spring Boot

## 1. What

`Feign Client` (OpenFeign, tích hợp qua Spring Cloud OpenFeign) là một HTTP client dạng khai báo (declarative). Thay vì viết code để build request và parse response, bạn chỉ cần định nghĩa một Java `interface` và dùng annotations của Spring MVC (`@GetMapping`, `@PostMapping`, `@PathVariable`, v.v.) để mô tả HTTP contract. Spring Cloud tự động generate implementation lúc runtime thông qua JDK Dynamic Proxy. Feign đặc biệt phổ biến trong microservices với Spring Cloud vì tích hợp tự nhiên với Eureka, Spring Cloud LoadBalancer, và Resilience4j.

---

## 2. Why

Trong kiến trúc microservices với 10+ services, mỗi service cần gọi 3-4 service khác. Nếu dùng `RestTemplate` hay `RestClient`, mỗi service phải:
- Manage URL riêng cho từng service đích.
- Build request theo từng endpoint cụ thể.
- Parse response riêng.
- Handle error riêng.
- Cấu hình retry, timeout, circuit breaker riêng.

Đây là code boilerplate lặp đi lặp lại và dễ drift (service A và service B cùng gọi service C nhưng với code khác nhau, dễ inconsistency). Feign giải quyết bằng cách:
- Centralize HTTP contract vào interface — thường được đặt trong shared library.
- Integrate tự động với service discovery (Eureka/Consul) — không cần hardcode URL.
- Tích hợp load balancing, circuit breaker, và retry theo chuẩn Spring Cloud.
- Code gọi service khác trông như gọi method Java nội bộ.

---

## 3. Mental Model

Hãy tưởng tượng bạn là giám đốc kinh doanh và cần đặt hàng từ nhiều nhà cung cấp khác nhau:

Cách cũ (RestTemplate): Với mỗi nhà cung cấp, bạn phải tự gọi điện, báo địa chỉ kho, đọc danh sách sản phẩm, thương lượng giá, và điền form đặt hàng thủ công — lặp lại quy trình này cho 10 nhà cung cấp.

Cách với Feign: Bạn có một **thư ký thông minh (Feign Proxy)**. Bạn đưa cho thư ký một **danh bạ giao dịch (Interface)** — trong đó liệt kê rõ: "Nhà cung cấp A, muốn lấy tồn kho thì gọi số máy `/inventory/{productId}`, muốn đặt hàng thì gửi form `/orders`". Từ đó về sau, bạn chỉ cần nói với thư ký: "Lấy tồn kho sản phẩm 123 từ nhà cung cấp A" — thư ký tự biết gọi cho ai, nói gì, và mang kết quả về cho bạn đúng format.

---

## 4. Where it fits

```
[Order Service]
      |
      |-- InventoryClient (Feign Interface)
      |         |
      |         v
      |   [Feign Proxy (JDK Dynamic Proxy)]
      |         |
      |         v
      |   [Spring Cloud LoadBalancer]
      |         |
      |         v
      |   [Eureka Service Registry] -- resolves "inventory-service" to actual IP:port
      |         |
      |         v
      |   [Inventory Service Instance 1] (load balanced)
      |   [Inventory Service Instance 2]
      |
      |-- UserClient (Feign Interface) --> [User Service]
      |
      |-- NotificationClient (Feign Interface) --> [Notification Service]
```

Trong monorepo microservices điển hình:

```
common-lib/
  src/main/java/
    com/example/clients/
      InventoryClient.java    <-- Interface Feign, shared giữa consumer services
      UserClient.java
      OrderClient.java

order-service/
  pom.xml                     <-- depends on common-lib
  src/main/java/
    com/example/order/
      OrderService.java       <-- @Autowired InventoryClient inventoryClient
```

---

## 5. When to use

- Trong kiến trúc microservices dùng Spring Cloud — đây là use case chính của Feign.
- Khi nhiều service cùng gọi một service khác và muốn centralize HTTP contract trong shared library.
- Khi muốn service communication code trông giống method call Java nội bộ — improve readability.
- Khi cần tích hợp service discovery (Eureka/Consul) để không hardcode URL — Feign tự động resolve tên service.
- Khi cần load balancing tự động giữa nhiều instance của cùng service — Spring Cloud LoadBalancer integrate tự nhiên.
- Khi cần circuit breaker (Resilience4j) cho từng service client — Feign có built-in support.

---

## 6. When NOT to use

Không dùng Feign cho external API bên ngoài tổ chức (Stripe, Twilio, Google Maps) khi không có service discovery. Feign có thể gọi external URL bằng `url` attribute, nhưng lúc đó bạn mất hết lợi thế (load balancing, service discovery) và chỉ còn lại tính declarative — không đáng thêm dependency Spring Cloud.

Không dùng khi app là Spring WebFlux reactive và cần non-blocking. Feign mặc định là blocking. Có Reactive Feign (`feign-reactor`) nhưng ít được maintain và ít phổ biến hơn nhiều so với WebClient.

Không dùng khi cần low-level control của HTTP request (custom serialization, binary protocol, chunked upload) — Feign abstraction có thể là trở ngại.

Không dùng khi team nhỏ với 1-2 service, không có Spring Cloud. Overhead của Spring Cloud ecosystem (Eureka, Config Server, Gateway) không xứng đáng cho project nhỏ.

---

## 7. Trade-offs

| Pros | Cons |
|---|---|
| Code cực kỳ ngắn gọn, clean — giống Java method call | Khó debug: proxy-generated code, stack trace ít ý nghĩa |
| Contract service được type-safe tại compile time | Phụ thuộc Spring Cloud — heavy ecosystem nếu chỉ cần Feign |
| Tích hợp sẵn service discovery, load balancing | Blocking mặc định — không tốt cho high-concurrency reactive app |
| Dễ mock trong unit test (chỉ cần mock interface) | Cấu hình phức tạp: timeout, retry, circuit breaker, fallback ở nhiều nơi |
| Shared interface trong common library enforce contract | Startup time tăng: Spring phải scan và create proxy cho mọi Feign interface |
| Error handling tập trung qua `ErrorDecoder` | Versioning khó: khi service đích thay đổi API, tất cả consumers phải update |

---

## 8. Alternatives

| Option | Khi nào chọn thay Feign |
|---|---|
| `RestClient` | Internal service call không cần service discovery, Spring Boot 3.2+ |
| `WebClient` | Reactive app, cần non-blocking, streaming |
| `RestTemplate` | Legacy app không dùng Spring Cloud |
| gRPC với Protobuf | Cần performance cao, strong typing, bidirectional streaming giữa services |
| GraphQL (DGS Federation) | Service mesh với complex data graph, query flexibility |

---

## 9. How

```java
// =============================================
// 1. Dependencies (pom.xml)
// =============================================
// spring-boot-starter-parent 3.2.x
<dependency>
    <groupId>org.springframework.cloud</groupId>
    <artifactId>spring-cloud-starter-openfeign</artifactId>
</dependency>
// Nếu dùng với Eureka service discovery:
<dependency>
    <groupId>org.springframework.cloud</groupId>
    <artifactId>spring-cloud-starter-netflix-eureka-client</artifactId>
</dependency>


// =============================================
// 2. Kích hoạt Feign
// =============================================
@SpringBootApplication
@EnableFeignClients(basePackages = "com.example.clients")  // scan package chứa interfaces
public class OrderServiceApplication {
    public static void main(String[] args) {
        SpringApplication.run(OrderServiceApplication.class, args);
    }
}


// =============================================
// 3. Định nghĩa Feign Interface
// =============================================
// name: tên service trong Eureka (hoặc tên logic)
// url: override URL nếu không dùng service discovery (external API, local dev)
// path: prefix path cho tất cả endpoints
@FeignClient(
    name = "inventory-service",
    // url = "${services.inventory.url}",  // dùng nếu không có Eureka
    path = "/api/v1",
    configuration = InventoryFeignConfig.class,
    fallbackFactory = InventoryClientFallbackFactory.class
)
public interface InventoryClient {

    // GET với path variable
    @GetMapping("/products/{id}/stock")
    ProductStock getStock(@PathVariable("id") Long productId);

    // GET với query params
    @GetMapping("/products")
    Page<Product> searchProducts(
        @RequestParam("keyword") String keyword,
        @RequestParam("page") int page,
        @RequestParam("size") int size);

    // POST với request body
    @PostMapping("/reservations")
    ReservationResponse reserveStock(@RequestBody ReservationRequest request);

    // PUT
    @PutMapping("/products/{id}")
    Product updateProduct(@PathVariable("id") Long id, @RequestBody UpdateProductRequest request);

    // DELETE
    @DeleteMapping("/reservations/{id}")
    void cancelReservation(@PathVariable("id") Long reservationId);

    // Truyền header per-call (ví dụ: correlation ID)
    @GetMapping("/products/{id}")
    Product getProduct(
        @PathVariable("id") Long id,
        @RequestHeader("X-Correlation-ID") String correlationId);
}


// =============================================
// 4. Feign Configuration (per-client)
// =============================================
// QUAN TRỌNG: Không đánh @Configuration để tránh bị component scan global
// Nếu đánh @Configuration, config này sẽ áp dụng cho TẤT CẢ Feign clients
public class InventoryFeignConfig {

    @Bean
    public Logger.Level feignLoggerLevel() {
        return Logger.Level.BASIC;  // NONE, BASIC, HEADERS, FULL
    }

    @Bean
    public Retryer feignRetryer() {
        // Retry 3 lần, bắt đầu sau 100ms, tối đa 1 giây giữa các lần
        return new Retryer.Default(100, 1000, 3);
    }

    @Bean
    public ErrorDecoder errorDecoder() {
        return new InventoryErrorDecoder();
    }
}

// Custom ErrorDecoder — xử lý error response thành business exceptions
public class InventoryErrorDecoder implements ErrorDecoder {
    private final ErrorDecoder defaultDecoder = new Default();

    @Override
    public Exception decode(String methodKey, Response response) {
        return switch (response.status()) {
            case 404 -> new ProductNotFoundException("Product not found");
            case 409 -> new InsufficientStockException("Not enough stock");
            case 429 -> new RateLimitException("Inventory service rate limit exceeded");
            default -> defaultDecoder.decode(methodKey, response);
        };
    }
}


// =============================================
// 5. Timeout configuration (application.yml)
// =============================================
```

```yaml
# application.yml
feign:
  client:
    config:
      default:                    # áp dụng cho tất cả Feign clients
        connect-timeout: 3000     # milliseconds
        read-timeout: 10000       # milliseconds
        logger-level: BASIC
      inventory-service:          # override cho client cụ thể
        read-timeout: 5000        # inventory service phải nhanh hơn
        logger-level: FULL        # FULL logging cho inventory trong dev
  compression:
    request:
      enabled: true               # Gzip compress request body
    response:
      enabled: true               # Decompress Gzip response

spring:
  cloud:
    loadbalancer:
      ribbon:
        enabled: false            # Dùng Spring Cloud LoadBalancer (không phải Ribbon cũ)
```

```java
// =============================================
// 6. Fallback với FallbackFactory (Resilience4j)
// =============================================
// FallbackFactory > Fallback vì có thể log được exception gốc
@Component
public class InventoryClientFallbackFactory
        implements FallbackFactory<InventoryClient> {

    private static final Logger log =
        LoggerFactory.getLogger(InventoryClientFallbackFactory.class);

    @Override
    public InventoryClient create(Throwable cause) {
        log.error("Inventory service fallback triggered: {}", cause.getMessage());

        return new InventoryClient() {
            @Override
            public ProductStock getStock(Long productId) {
                // Trả về stock = 0 khi service không available
                return ProductStock.builder()
                    .productId(productId)
                    .quantity(0)
                    .available(false)
                    .build();
            }

            @Override
            public Page<Product> searchProducts(String keyword, int page, int size) {
                return Page.empty();
            }

            @Override
            public ReservationResponse reserveStock(ReservationRequest request) {
                throw new ServiceUnavailableException(
                    "Inventory service unavailable. Please try again later.");
            }

            // Implement các method khác với fallback logic...
            @Override
            public Product updateProduct(Long id, UpdateProductRequest request) {
                throw new ServiceUnavailableException("Inventory service unavailable");
            }

            @Override
            public void cancelReservation(Long reservationId) {
                log.warn("Cancellation may be lost for reservation {} — service unavailable",
                    reservationId);
            }

            @Override
            public Product getProduct(Long id, String correlationId) {
                throw new ServiceUnavailableException("Inventory service unavailable");
            }
        };
    }
}

// Bật Feign circuit breaker trong application.yml
```

```yaml
feign:
  circuitbreaker:
    enabled: true  # Bật Resilience4j integration
```

```java
// =============================================
// 7. RequestInterceptor — tự động inject header vào mọi request
// =============================================
@Component
public class JwtForwardingInterceptor implements RequestInterceptor {

    @Override
    public void apply(RequestTemplate template) {
        // Lấy JWT token từ SecurityContext của request hiện tại và forward sang service đích
        Authentication auth = SecurityContextHolder.getContext().getAuthentication();
        if (auth instanceof JwtAuthenticationToken jwtAuth) {
            template.header("Authorization", "Bearer " + jwtAuth.getToken().getTokenValue());
        }
        // Thêm correlation ID để trace request xuyên services
        String correlationId = MDC.get("correlationId");
        if (correlationId != null) {
            template.header("X-Correlation-ID", correlationId);
        }
    }
}

// Đăng ký interceptor cho TẤT CẢ Feign clients (global config):
@Configuration
public class GlobalFeignConfig {
    @Bean
    public RequestInterceptor jwtForwardingInterceptor() {
        return new JwtForwardingInterceptor();
    }
}

// Hoặc đăng ký cho client cụ thể qua configuration class (không @Configuration)


// =============================================
// 8. Usage trong Service Layer
// =============================================
@Service
@RequiredArgsConstructor
public class OrderService {

    private final InventoryClient inventoryClient;
    private final UserClient userClient;

    public OrderResponse createOrder(CreateOrderRequest request) {
        // Trông như gọi method Java nội bộ — nhưng thực ra là HTTP call
        ProductStock stock = inventoryClient.getStock(request.getProductId());

        if (!stock.isAvailable() || stock.getQuantity() < request.getQuantity()) {
            throw new InsufficientStockException("Not enough stock for product: "
                + request.getProductId());
        }

        User user = userClient.getUserById(request.getUserId());

        ReservationResponse reservation = inventoryClient.reserveStock(
            new ReservationRequest(request.getProductId(), request.getQuantity()));

        return buildOrderResponse(user, stock, reservation);
    }
}


// =============================================
// 9. Unit testing — mock interface dễ dàng
// =============================================
@ExtendWith(MockitoExtension.class)
class OrderServiceTest {

    @Mock
    private InventoryClient inventoryClient;

    @Mock
    private UserClient userClient;

    @InjectMocks
    private OrderService orderService;

    @Test
    void createOrder_shouldSucceed_whenStockAvailable() {
        // Arrange
        when(inventoryClient.getStock(1L))
            .thenReturn(ProductStock.builder().productId(1L).quantity(10).available(true).build());
        when(userClient.getUserById(100L))
            .thenReturn(User.builder().id(100L).name("Alice").build());
        when(inventoryClient.reserveStock(any()))
            .thenReturn(new ReservationResponse("RES-001", ReservationStatus.CONFIRMED));

        // Act
        OrderResponse result = orderService.createOrder(
            new CreateOrderRequest(100L, 1L, 2));

        // Assert
        assertThat(result.getStatus()).isEqualTo(OrderStatus.CREATED);
        verify(inventoryClient).reserveStock(argThat(r -> r.getQuantity() == 2));
    }

    @Test
    void createOrder_shouldThrow_whenInsufficientStock() {
        when(inventoryClient.getStock(1L))
            .thenReturn(ProductStock.builder().quantity(0).available(false).build());

        assertThatThrownBy(() -> orderService.createOrder(new CreateOrderRequest(100L, 1L, 5)))
            .isInstanceOf(InsufficientStockException.class);

        verify(inventoryClient, never()).reserveStock(any());
    }
}
```

---

## 10. Production concerns

**Load Balancing:**
Khi `name` trong `@FeignClient` trùng với service name đăng ký trong Eureka, Spring Cloud LoadBalancer tự động resolve. Feign không cần biết IP thật — chỉ cần tên logic. LoadBalancer mặc định là round-robin. Cấu hình custom LoadBalancer (ví dụ: zone-aware, weighted) qua `ReactorLoadBalancer` Bean.

**Circuit Breaker với Resilience4j:**
Khi `feign.circuitbreaker.enabled=true`, Feign wrap mỗi method call bằng Resilience4j circuit breaker. Cấu hình per-client trong `application.yml`:

```yaml
resilience4j:
  circuitbreaker:
    instances:
      inventory-service:
        sliding-window-size: 10
        failure-rate-threshold: 50        # 50% failure rate -> OPEN
        wait-duration-in-open-state: 30s
        permitted-number-of-calls-in-half-open-state: 5
  timelimiter:
    instances:
      inventory-service:
        timeout-duration: 3s
```

**Failure modes:**
- Default timeout quá ngắn: Feign default `connectTimeout = 10s`, `readTimeout = 60s` trong một số phiên bản — nhưng một số phiên bản khác có thể là 1 giây. Luôn set explicit trong config.
- `@EnableFeignClients` không tìm thấy interfaces: nếu không set `basePackages`, Spring scan từ package của main class. Nếu Feign interfaces ở package khác (ví dụ shared lib), phải khai báo `basePackageClasses` hoặc `clients` attribute.
- Fallback không được gọi dù service down: Fallback chỉ được gọi khi circuit breaker mở hoặc khi Feign throw exception. Nếu service trả về 200 với error body, Feign không biết đó là lỗi — cần `ErrorDecoder` để convert thành exception.

**Monitoring:**
- Feign tự động expose metrics qua Micrometer nếu `feign.micrometer.enabled=true` (default true từ Spring Cloud 2022.0+). Metrics: `feign.Client.requests` với tags: feign.client.name, feign.method, feign.uri, http.status.
- Bật distributed tracing: Feign propagate trace context (B3, W3C TraceContext) tự động khi Micrometer Tracing có trên classpath.
- Bật detailed logging cho debug: `logging.level.com.example.clients: DEBUG` kết hợp với `Logger.Level.FULL`.

---

## 11. Common mistakes

- Mistake: Đánh `@Configuration` trên Feign configuration class được khai báo trong `@FeignClient(configuration = ...)`.
  Fix: Không đánh `@Configuration` lên per-client config class. Nếu đánh, Spring component scan sẽ register nó như global config — áp dụng cho tất cả Feign clients thay vì chỉ client đó. Chỉ đánh `@Configuration` khi thực sự muốn global config (và đăng ký trong `@EnableFeignClients(defaultConfiguration = ...)`).

- Mistake: Quên `@EnableFeignClients` trên main class.
  Fix: Đây là annotation cần thiết để Spring tạo proxy cho Feign interfaces. Không có nó, inject `InventoryClient` sẽ throw `NoSuchBeanDefinitionException`. Kiểm tra `basePackages` nếu interfaces không cùng package với main class.

- Mistake: Không cấu hình timeout — dùng default Feign timeout có thể quá dài hoặc quá ngắn.
  Fix: Luôn set explicit `connect-timeout` và `read-timeout` trong `application.yml` cho `default` config, và override cho từng client theo SLA của service đó.

- Mistake: Dùng Fallback thay vì FallbackFactory — Fallback không có thông tin về exception gốc.
  Fix: Luôn dùng `FallbackFactory` thay vì `fallback` attribute trong `@FeignClient`. Factory nhận `Throwable cause`, cho phép log exception gốc và quyết định fallback behavior tùy theo loại lỗi (timeout vs connection refused vs 5xx).

- Mistake: Đặt Feign interfaces trong service module thay vì shared library.
  Fix: Trong microservices, Feign interfaces nên được đặt trong shared common library. Service B muốn gọi service A sẽ import common library, sử dụng `ServiceAClient` interface. Điều này tránh duplicate code và giữ contract nhất quán giữa producer và consumer.

---

## 12. Sample project

**Tên project: E-Commerce Order Orchestration Service**

Xây dựng một Spring Boot 3.2 service (`order-service`) là một order orchestrator trong hệ thống e-commerce microservices. Service này gọi 3 internal services để xử lý một đơn hàng.

**Constraint bắt buộc:**
- Phải dùng 3 Feign interfaces: `InventoryClient`, `UserClient`, `PaymentClient` — mỗi cái trong package `com.example.clients` (giả lập shared library).
- `PaymentClient` phải có `FallbackFactory`: nếu payment service không phản hồi, throw `PaymentServiceUnavailableException` (không được nuốt lỗi).
- `InventoryClient` phải có `FallbackFactory`: nếu inventory service không phản hồi, assume `stock = 0` và reject order với message rõ ràng.
- Phải implement `RequestInterceptor` global inject `X-Request-ID` (UUID) vào header của mọi Feign request để support distributed tracing.
- `InventoryClient` phải có `ErrorDecoder` riêng xử lý: 404 → `ProductNotFoundException`, 409 → `InsufficientStockException`, 429 → log warning và throw `RateLimitException`.
- Phải viết integration test với `@SpringBootTest` và WireMock (thay vì MockRestServiceServer) để mock downstream services, cover 5 scenarios: happy path, inventory unavailable, payment failed, product not found, rate limited.

---

## 13. Interview

**Core Q&A:**

Q: Feign Client hoạt động như thế nào dưới nền (under the hood)?
A: Spring Cloud dùng JDK Dynamic Proxy (hoặc CGLIB nếu cần) để tạo implementation từ interface lúc startup. Khi một method được gọi, InvocationHandler của proxy phân tích annotations (`@GetMapping`, `@PathVariable`, v.v.) để build URL, method, headers, body. Request được gửi qua underlying HTTP client (JDK HttpURLConnection, OkHttp, hoặc Apache HttpClient tùy classpath). Response được decode qua `Decoder` (mặc định dùng Jackson). Nếu status là error, `ErrorDecoder` được gọi để convert thành exception.

Q: Sự khác biệt giữa `fallback` và `fallbackFactory` trong `@FeignClient`?
A: `fallback` chỉ nhận tên class implement interface — không có thông tin về exception gốc. `fallbackFactory` implement `FallbackFactory<T>` — factory nhận `Throwable cause` và trả về implementation của interface. `fallbackFactory` luôn là lựa chọn tốt hơn vì: (1) có thể log exception gốc để biết lý do fallback, (2) có thể quyết định fallback behavior khác nhau tùy theo loại lỗi (timeout vs 4xx vs 5xx).

Q: Làm thế nào để truyền JWT token của current user qua Feign sang service đích?
A: Implement `RequestInterceptor` Bean và đọc JWT từ `SecurityContextHolder`:
```java
@Bean
public RequestInterceptor jwtInterceptor() {
    return template -> {
        String token = ((JwtAuthenticationToken)
            SecurityContextHolder.getContext().getAuthentication())
            .getToken().getTokenValue();
        template.header("Authorization", "Bearer " + token);
    };
}
```
Đây là global interceptor áp dụng cho tất cả Feign clients. Nếu chỉ muốn cho client cụ thể, đặt trong per-client config class (không `@Configuration`).

Q: Tại sao `@Configuration` trên Feign config class lại nguy hiểm?
A: Nếu đánh `@Configuration` lên class được khai báo trong `@FeignClient(configuration = InventoryFeignConfig.class)`, Spring component scan sẽ register nó như global Spring config — khi đó `Logger.Level`, `Retryer`, `ErrorDecoder` trong class đó áp dụng cho TẤT CẢ Feign clients, không chỉ InventoryClient. Điều này thường là bug không mong muốn: client khác bị ảnh hưởng, và error decoder cho inventory có thể throw wrong exception cho payment client.

Q: Feign có hỗ trợ Spring Cloud LoadBalancer không, và làm thế nào?
A: Có. Khi `name` trong `@FeignClient` trùng với service name đăng ký trong Eureka/Consul, Spring Cloud LoadBalancer intercept request và resolve tên service thành actual IP:port của một instance đang running. Feign không biết địa chỉ thật — nó chỉ gửi request với host là service name, LoadBalancer thay thế. Mặc định round-robin. Config Eureka: `spring.application.name=inventory-service` ở service đích.

Q: Làm thế nào để cấu hình Feign khác nhau cho mỗi environment (dev, staging, prod)?
A: Dùng Spring profiles trong `application.yml`. URL override, timeout, log level có thể khác nhau:
```yaml
feign:
  client:
    config:
      inventory-service:
        url: ${INVENTORY_SERVICE_URL:http://localhost:8081}
        read-timeout: ${FEIGN_READ_TIMEOUT:10000}
        logger-level: ${FEIGN_LOG_LEVEL:BASIC}
```
Trên môi trường dev local, không có Eureka, set `url` để Feign bypass LoadBalancer và gọi thẳng localhost.

**Scenario:**

Scenario: Bạn có 10 Feign clients. Bạn muốn tất cả đều: (1) inject `X-Correlation-ID` header, (2) dùng chung một `ErrorDecoder` cơ bản, nhưng một số client có thêm custom behavior riêng. Bạn cấu hình thế nào?
Answer: Tạo `GlobalFeignConfig` class có `@Configuration` (global), đăng ký `RequestInterceptor` inject correlation ID và base `ErrorDecoder`. Với client cần custom behavior, tạo per-client config class (không `@Configuration`) override chỉ bean cần thay đổi (ví dụ: custom `ErrorDecoder` extends base class). Spring sẽ merge: global config là base, per-client config override. Đăng ký global config trong `@EnableFeignClients(defaultConfiguration = GlobalFeignConfig.class)`.

Scenario: Inventory service đang gặp vấn đề và phản hồi rất chậm (avg 5 giây thay vì 200ms bình thường). Order service của bạn đang bị thread pool exhaustion. Bạn xử lý thế nào?
Answer: Multi-layer approach: (1) Set `read-timeout = 2000ms` cho inventory Feign client — requests không chờ quá 2 giây. (2) Bật Resilience4j circuit breaker: sau 50% failure rate trong 10 requests, circuit mở 30 giây — không gửi request đến service chậm, fail fast ngay lập tức. (3) `FallbackFactory` trả về `stock = 0` khi circuit mở, order flow gracefully degraded (reject order vì "not enough stock") thay vì hang. (4) Monitor circuit breaker state qua Actuator `/actuator/circuitbreakers`.

Scenario: Bạn cần Feign Client gọi một external API (Stripe) không có trong Eureka. Bạn setup thế nào?
Answer: Dùng `url` attribute trong `@FeignClient`: `@FeignClient(name = "stripe-api", url = "${stripe.api.base-url}")`. Bằng cách này Feign bypass service discovery và gọi thẳng URL được config. `name` vẫn cần để định danh client cho config và metrics. Không cần `@EnableDiscoveryClient` hay Eureka cho client này.

---

## 14. References

- Spring Cloud OpenFeign Official Docs: https://docs.spring.io/spring-cloud-openfeign/docs/current/reference/html/
- OpenFeign GitHub (underlying library): https://github.com/OpenFeign/feign
- Spring Cloud Reference Docs — LoadBalancer: https://docs.spring.io/spring-cloud-commons/docs/current/reference/html/#spring-cloud-loadbalancer
- Resilience4j Spring Boot Starter Docs: https://resilience4j.readme.io/docs/getting-started-3
- Spring Cloud Circuit Breaker with Feign: https://spring.io/projects/spring-cloud-circuitbreaker
- Spring Cloud OpenFeign Changelog: https://github.com/spring-cloud/spring-cloud-openfeign/releases

---

## 15. Real-world Code

- Spring PetClinic Microservices — extensive Feign Client usage: https://github.com/spring-petclinic/spring-petclinic-microservices
- Spring Cloud Samples — Feign with Eureka: https://github.com/spring-cloud-samples/feign-eureka
- Piggy Metrics — real microservices project với Feign: https://github.com/sqshq/piggymetrics
- E-Commerce microservices example với Feign + Resilience4j: https://github.com/mehmetozanguven/spring-cloud-microservices-example
- Spring Cloud OpenFeign integration tests: https://github.com/spring-cloud/spring-cloud-openfeign/tree/main/spring-cloud-openfeign-core/src/test

---

## 16. Community

- Stack Overflow — "Spring Cloud Feign FallbackFactory vs Fallback": https://stackoverflow.com/questions/37706215/feign-fallbackfactory-vs-fallback
- Stack Overflow — "Feign Client @Configuration annotation issue": https://stackoverflow.com/questions/44613973/spring-cloud-feign-configuration-class-with-configuration-annotation
- Baeldung — Introduction to Spring Cloud OpenFeign: https://www.baeldung.com/spring-cloud-openfeign
- Baeldung — Feign Retry: https://www.baeldung.com/feign-retry
- Reddit r/SpringBoot — "Feign vs RestClient for microservices in 2024": https://www.reddit.com/r/SpringBoot/comments/feign_vs_restclient/
- Medium — "Production Feign Client Configuration Mistakes": https://medium.com/swlh/spring-cloud-feign-client-common-mistakes
- DZone — "Advanced Spring Cloud OpenFeign Configuration": https://dzone.com/articles/advanced-spring-cloud-openfeign
