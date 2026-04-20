---
created: 2026-04-17
tags:
  - "#type/library"
  - "#status/draft"
  - "#lang/spring"
  - "#topic/http"
related:
  - "[[RestTemplate in Spring Boot]]"
  - "[[HTTP Clients in Spring]]"
---

# RestClient in Spring Boot

## 1. What

`RestClient` là HTTP client đồng bộ (synchronous, blocking) hiện đại được giới thiệu trong Spring Framework 6.1 và Spring Boot 3.2. Nó cung cấp fluent builder API lấy cảm hứng từ `WebClient`, nhưng hoạt động trên mô hình blocking truyền thống — không cần Project Reactor hay `Mono`/`Flux`. `RestClient` là lựa chọn chính thức được Spring khuyến nghị để thay thế `RestTemplate` trong các ứng dụng Spring MVC mới.

---

## 2. Why

Spring từng có hai HTTP client với API hoàn toàn khác nhau: `RestTemplate` (synchronous, nhiều overloaded method) và `WebClient` (reactive, fluent builder). Developer không reactive phải dùng `RestTemplate` với API cồng kềnh và verbose. Developer reactive dùng `WebClient` nhưng có thể bị overkill với app không cần reactive.

`RestClient` ra đời để giải quyết khoảng trống này: mang trải nghiệm fluent API của `WebClient` vào thế giới synchronous mà không yêu cầu thêm `spring-boot-starter-webflux`. Nó cũng được thiết kế tốt hơn `RestTemplate`:

- `RestTemplate` có method `getForObject`, `getForEntity`, `postForObject`, `postForEntity`, `postForLocation`, `exchange`, `execute` — nhiều overload cho cùng chức năng, khó nhớ cái nào dùng khi nào.
- `RestClient` có một luồng duy nhất: `.get()/.post()/.put()/.delete()` → `.uri()` → `.headers()` → `.body()` → `.retrieve()` → `.body()/.toEntity()/.toBodilessEntity()`. Tất cả trong một cú pháp nhất quán.

Thêm vào đó, với Java 21 Virtual Threads, blocking I/O trở nên hiệu quả gần tương đương non-blocking (virtual thread tự park khi block thay vì giữ platform thread) — `RestClient` với Virtual Threads là stack đơn giản nhất mà vẫn cho throughput cao.

---

## 3. Mental Model

Hãy tưởng tượng bạn đang order đồ ăn tại quán:

- `RestTemplate` (cũ): Bạn phải lật qua 20 trang thực đơn dày cộp, mỗi món có nhiều cách order khác nhau tùy trường hợp. Bạn phải nhớ "dùng form A cho trường hợp này, form B cho trường hợp kia". Nhân viên cũ kỹ nhưng đáng tin.

- `RestClient` (mới): Thực đơn được thiết kế lại gọn gàng — chỉ có một flow: "Chọn món" → "Chỉ định bàn" → "Thêm yêu cầu đặc biệt" → "Xác nhận order" → "Nhận đồ ăn". Bất kể bạn order món gì, cú pháp đều như nhau. Và nhân viên vẫn đứng chờ mang đồ về cho bạn (blocking) — nhưng trông chuyên nghiệp hơn nhiều.

---

## 4. Where it fits

```
[Spring MVC Application — Spring Boot 3.2+]
        |
[Service Layer]
        |
[RestClient (injected Bean)]
        |
        |-- [ClientHttpRequestInterceptor]  (logging, auth, request ID)
        |
        |-- [HttpMessageConverter]          (JSON <-> POJO via Jackson)
        |
        |-- [ClientHttpRequestFactory]      (underlying HTTP connection)
        |        |
        |        |-- JdkClientHttpRequestFactory (default, JDK 11+ HttpClient)
        |        |-- HttpComponentsClientHttpRequestFactory (Apache HttpClient 5)
        |        |-- OkHttp3ClientHttpRequestFactory (OkHttp)
        |
        v
[External REST API]
```

Cùng infrastructure với `RestTemplate`:
- `RestClient` và `RestTemplate` dùng chung `ClientHttpRequestFactory` và `HttpMessageConverter`.
- `RestClient` có thể được tạo từ existing `RestTemplate`: `RestClient.create(restTemplate)` — hữu ích khi migrate dần dần.

---

## 5. When to use

- Dự án mới với Spring Boot 3.2+ dùng Spring MVC (không phải WebFlux) — đây là lựa chọn mặc định.
- Migrate từ `RestTemplate` trong dự án hiện tại sang Spring Boot 3.2 — API khá tương tự, migration dễ.
- Khi muốn synchronous HTTP client với error handling tường minh per status code mà không cần callback/reactive.
- Khi dùng Java 21 Virtual Threads (`spring.threads.virtual.enabled=true`) — kết hợp này cho performance tốt với code đơn giản.
- Khi cần fluent API dễ đọc, dễ maintain trong code review.

---

## 6. When NOT to use

Không dùng trong Spring Boot < 3.2 / Spring Framework < 6.1 — class `RestClient` không tồn tại.

Không dùng trong Spring WebFlux reactive application — dùng `WebClient`. `RestClient` là blocking, gọi từ Netty event loop thread sẽ block event loop.

Không dùng khi cần fan-out nhiều request song song một cách tự nhiên — `RestClient` không có operator tương đương `Mono.zip`. Để fan-out với `RestClient`, phải tự quản lý thread với `CompletableFuture` hoặc dùng `WebClient`.

Không dùng khi cần streaming response (SSE, chunked) — `RestClient` load toàn bộ response vào memory. Dùng `WebClient` với `bodyToFlux()` cho streaming.

---

## 7. Trade-offs

| Pros | Cons |
|---|---|
| Fluent API nhất quán, dễ đọc và maintain | Blocking — mỗi request chiếm thread (giảm nhẹ với Virtual Threads) |
| Không cần Reactor/WebFlux dependency | Không hỗ trợ streaming response (Flux) |
| Cùng infrastructure với RestTemplate — dễ migrate | Chỉ từ Spring Boot 3.2+ (Spring Framework 6.1+) |
| Error handling tường minh với `.onStatus()` | Ecosystem và community chưa nhiều bằng RestTemplate |
| Tích hợp Micrometer Observation tự động | Không có fan-out operator built-in như `Mono.zip` |
| Dùng tốt với Java 21 Virtual Threads | Cần Java 17+ |

---

## 8. Alternatives

| Option | Khi nào chọn thay RestClient |
|---|---|
| `RestTemplate` | Legacy codebase chưa thể migrate, Spring Boot < 3.2 |
| `WebClient` | Cần reactive, streaming, fan-out, hoặc đang dùng Spring WebFlux |
| `Feign Client` | Microservices với Spring Cloud, muốn declarative interface |
| `java.net.http.HttpClient` | Không dùng Spring, hoặc cần standalone HTTP client không có Spring dependency |

---

## 9. How

```java
// =============================================
// 1. Cấu hình RestClient Bean (Production-ready)
// =============================================
@Configuration
public class RestClientConfig {

    @Bean
    public RestClient restClient(RestClient.Builder builder) {
        // Cấu hình Apache HttpClient với connection pool
        HttpComponentsClientHttpRequestFactory factory =
            HttpComponentsClientHttpRequestFactory.builder()
                .httpClient(
                    HttpClients.custom()
                        .setConnectionManager(
                            PoolingHttpClientConnectionManagerBuilder.create()
                                .setMaxConnPerRoute(20)
                                .setMaxConnTotal(100)
                                .setConnectionTimeToLive(TimeValue.ofSeconds(30))
                                .build())
                        .build())
                .setConnectTimeout(Duration.ofSeconds(3))
                .setConnectionRequestTimeout(Duration.ofSeconds(3))
                .build();

        return builder
            .baseUrl("https://api.example.com")
            .defaultHeader(HttpHeaders.ACCEPT, MediaType.APPLICATION_JSON_VALUE)
            .requestFactory(factory)
            .requestInterceptor(new RequestIdInterceptor())   // custom interceptor
            .build();
    }

    // Interceptor tự động thêm request ID vào mọi request
    @Component
    public class RequestIdInterceptor implements ClientHttpRequestInterceptor {
        @Override
        public ClientHttpResponse intercept(HttpRequest request, byte[] body,
                ClientHttpRequestExecution execution) throws IOException {
            request.getHeaders().set("X-Request-ID", UUID.randomUUID().toString());
            return execution.execute(request, body);
        }
    }
}


// =============================================
// 2. GET requests — các pattern phổ biến
// =============================================
@Service
public class UserService {

    @Autowired
    private RestClient restClient;

    // GET basic — trả về object trực tiếp
    public User getUser(Long id) {
        return restClient.get()
            .uri("/users/{id}", id)
            .retrieve()
            .onStatus(status -> status.value() == 404, (request, response) -> {
                throw new UserNotFoundException("User not found: " + id);
            })
            .onStatus(HttpStatusCode::is5xxServerError, (request, response) -> {
                throw new ExternalServiceException("User service unavailable");
            })
            .body(User.class);
    }

    // GET với ResponseEntity (cần status code và headers)
    public ResponseEntity<User> getUserWithMetadata(Long id) {
        return restClient.get()
            .uri("/users/{id}", id)
            .retrieve()
            .toEntity(User.class);
    }

    // GET với generic type — List<User>
    public List<User> getAllUsers() {
        return restClient.get()
            .uri("/users")
            .retrieve()
            .body(new ParameterizedTypeReference<List<User>>() {});
    }

    // GET với query parameters
    public PagedResponse<User> getUsersPaged(int page, int size, String sortBy) {
        return restClient.get()
            .uri(uriBuilder -> uriBuilder
                .path("/users")
                .queryParam("page", page)
                .queryParam("size", size)
                .queryParam("sort", sortBy)
                .build())
            .retrieve()
            .body(new ParameterizedTypeReference<PagedResponse<User>>() {});
    }

    // GET với custom headers (e.g. Authorization)
    public User getUserWithAuth(Long id, String token) {
        return restClient.get()
            .uri("/users/{id}", id)
            .header(HttpHeaders.AUTHORIZATION, "Bearer " + token)
            .retrieve()
            .body(User.class);
    }
}


// =============================================
// 3. POST, PUT, DELETE
// =============================================
@Service
public class UserWriteService {

    @Autowired
    private RestClient restClient;

    // POST — tạo resource mới
    public User createUser(CreateUserRequest request) {
        return restClient.post()
            .uri("/users")
            .contentType(MediaType.APPLICATION_JSON)
            .body(request)
            .retrieve()
            .onStatus(status -> status.value() == 409, (req, resp) -> {
                throw new DuplicateEmailException("Email already exists");
            })
            .body(User.class);
    }

    // POST — chỉ cần biết response có success không, không cần body
    public void createUserNoBody(CreateUserRequest request) {
        restClient.post()
            .uri("/users")
            .contentType(MediaType.APPLICATION_JSON)
            .body(request)
            .retrieve()
            .toBodilessEntity();  // consume response mà không parse body
    }

    // PUT — update toàn bộ resource
    public User updateUser(Long id, UpdateUserRequest request) {
        return restClient.put()
            .uri("/users/{id}", id)
            .contentType(MediaType.APPLICATION_JSON)
            .body(request)
            .retrieve()
            .body(User.class);
    }

    // PATCH — update một phần
    public User patchUser(Long id, Map<String, Object> updates) {
        return restClient.patch()
            .uri("/users/{id}", id)
            .contentType(MediaType.APPLICATION_JSON)
            .body(updates)
            .retrieve()
            .body(User.class);
    }

    // DELETE — không có response body
    public void deleteUser(Long id) {
        restClient.delete()
            .uri("/users/{id}", id)
            .retrieve()
            .toBodilessEntity();
    }
}


// =============================================
// 4. Advanced: tạo RestClient từ RestTemplate cũ (migration path)
// =============================================
@Bean
public RestClient migratedRestClient(RestTemplate existingRestTemplate) {
    // Tái sử dụng configuration của RestTemplate cũ (interceptors, converters, factory)
    return RestClient.create(existingRestTemplate);
}


// =============================================
// 5. Multiple RestClient Beans cho nhiều external services
// =============================================
@Configuration
public class MultipleRestClientsConfig {

    // Primary HTTP client factory được share
    @Bean
    public ClientHttpRequestFactory httpRequestFactory() {
        return HttpComponentsClientHttpRequestFactory.builder()
            .setConnectTimeout(Duration.ofSeconds(3))
            .build();
    }

    @Bean("paymentRestClient")
    public RestClient paymentRestClient(RestClient.Builder builder,
            ClientHttpRequestFactory factory) {
        return builder
            .baseUrl("${services.payment.base-url}")
            .requestFactory(factory)
            .defaultHeader("X-Service-Name", "order-service")
            .build();
    }

    @Bean("inventoryRestClient")
    public RestClient inventoryRestClient(RestClient.Builder builder,
            ClientHttpRequestFactory factory) {
        return builder
            .baseUrl("${services.inventory.base-url}")
            .requestFactory(factory)
            .build();
    }
}

// Inject bằng @Qualifier
@Service
public class OrderService {
    @Autowired @Qualifier("paymentRestClient")
    private RestClient paymentRestClient;

    @Autowired @Qualifier("inventoryRestClient")
    private RestClient inventoryRestClient;
}


// =============================================
// 6. Error handling nâng cao
// =============================================
@Service
public class RobustUserService {

    @Autowired
    private RestClient restClient;

    public Optional<User> findUser(Long id) {
        try {
            User user = restClient.get()
                .uri("/users/{id}", id)
                .retrieve()
                .onStatus(status -> status.value() == 404,
                    (req, resp) -> {
                        throw new UserNotFoundException(id);
                    })
                .body(User.class);
            return Optional.ofNullable(user);
        } catch (UserNotFoundException e) {
            return Optional.empty();
        } catch (RestClientResponseException e) {
            log.error("HTTP error {} calling user API: {}", e.getStatusCode(), e.getMessage());
            throw new ExternalServiceException("User service error", e);
        } catch (ResourceAccessException e) {
            log.error("Network error calling user API", e);
            throw new ExternalServiceException("User service unreachable", e);
        }
    }
}


// =============================================
// 7. Testing với MockRestServiceServer
// =============================================
@SpringBootTest
class UserServiceTest {

    @Autowired
    private UserService userService;

    @Autowired
    private RestClient.Builder restClientBuilder;

    private MockRestServiceServer mockServer;

    @BeforeEach
    void setUp() {
        // RestClient.Builder có thể được mock bằng MockRestServiceServer
        RestTemplate restTemplate = new RestTemplate();
        mockServer = MockRestServiceServer.createServer(restTemplate);
        // Inject RestClient từ mocked RestTemplate
    }

    @Test
    void getUser_shouldReturnUser_whenExists() {
        mockServer.expect(requestTo("https://api.example.com/users/1"))
            .andExpect(method(HttpMethod.GET))
            .andRespond(withSuccess(
                """
                {"id": 1, "name": "Alice", "email": "alice@example.com"}
                """,
                MediaType.APPLICATION_JSON));

        User user = userService.getUser(1L);

        assertThat(user.getName()).isEqualTo("Alice");
        mockServer.verify();
    }

    @Test
    void getUser_shouldThrowUserNotFoundException_when404() {
        mockServer.expect(requestTo("https://api.example.com/users/999"))
            .andExpect(method(HttpMethod.GET))
            .andRespond(withStatus(HttpStatus.NOT_FOUND));

        assertThatThrownBy(() -> userService.getUser(999L))
            .isInstanceOf(UserNotFoundException.class);
    }
}
```

---

## 10. Production concerns

**Connection Pool:**
`RestClient` mặc định dùng `JdkClientHttpRequestFactory` (JDK 11+ `HttpClient`) với connection pool cơ bản. Cho production traffic cao, upgrade sang Apache HttpClient 5 với pooling được cấu hình chi tiết:

```yaml
# application.yml — không có direct config cho RestClient pool
# Phải cấu hình qua Bean như ví dụ trong section 9
```

**Virtual Threads (Java 21+):**
`RestClient` đặc biệt tốt khi kết hợp với Virtual Threads. Kích hoạt trong Spring Boot 3.2+:

```yaml
spring:
  threads:
    virtual:
      enabled: true
```

Virtual Thread tự park khi blocking (I/O wait) và không tiêu tốn platform thread, nên `RestClient` đạt throughput gần tương đương `WebClient` với code đơn giản hơn nhiều.

**Failure modes:**
- Không set timeout: `RestClient` không có default timeout. Một API chậm sẽ block thread vĩnh viễn. Luôn set `connectTimeout` và `readTimeout` ở request factory level.
- Không handle `RestClientResponseException` và `ResourceAccessException`: hai loại exception này có ngữ nghĩa khác nhau. `RestClientResponseException` nghĩa là server phản hồi nhưng với status lỗi; `ResourceAccessException` nghĩa là không thể kết nối (network error, timeout).
- Không dùng `toBodilessEntity()` khi không cần body: nếu không consume response body, connection có thể không được trả về pool.

**Monitoring:**
- Spring Boot 3.2+ tích hợp Micrometer Observation cho `RestClient` tự động nếu `ObservationRegistry` được expose. Metrics `http.client.requests` tự động được record.
- Cấu hình trong `application.yml`:
  ```yaml
  management:
    observations:
      http:
        client:
          requests:
            name: http.client.requests
  ```

---

## 11. Common mistakes

- Mistake: Nghĩ rằng `RestClient` là non-blocking vì API trông giống `WebClient`. `restClient.get().uri("...").retrieve().body(User.class)` là synchronous hoàn toàn.
  Fix: Nhớ rằng `RestClient` trả về `User` trực tiếp (blocking), không phải `Mono<User>`. Nó sẽ block thread hiện tại cho đến khi nhận response.

- Mistake: Không xử lý `RestClientResponseException` — exception tổng quát cho 4xx và 5xx.
  Fix: Dùng `.onStatus()` cho xử lý tường minh, hoặc bắt `RestClientResponseException` (và các subclass của nó như `HttpClientErrorException.NotFound`) trong try-catch ở service layer.

- Mistake: Tạo `RestClient` mới với `RestClient.create()` trong mỗi method call.
  Fix: Khai báo `@Bean` và inject. `RestClient` instance là immutable và thread-safe — nên tạo một lần và tái sử dụng.

- Mistake: Quên gọi `toBodilessEntity()` hoặc `.body()` sau `.retrieve()`. `retrieve()` không tự execute request ngay — bạn phải terminal operation (`.body()`, `.toEntity()`, `.toBodilessEntity()`) để trigger execution.
  Fix: Luôn chain terminal operation sau `retrieve()`. Không có terminal operation, request không được gửi đi và không có exception nào được throw.

- Mistake: Dùng `RestClient` với Spring Boot 3.1 hoặc Spring Framework 6.0 — `RestClient` không tồn tại ở các phiên bản này.
  Fix: Kiểm tra phiên bản. Spring Boot 3.2 = Spring Framework 6.1 — đây là mức tối thiểu. Nếu không thể nâng cấp, dùng `WebClient` (có fluent API tương tự) hoặc `RestTemplate`.

---

## 12. Sample project

**Tên project: API Key Manager — Multi-Provider Rate-Aware HTTP Client**

Xây dựng một Spring Boot 3.2 service hoạt động như một proxy wrapper cho 3 external AI APIs (OpenAI, Anthropic, Google Gemini). Expose endpoint `POST /ai/complete` nhận prompt và trả về completions từ tất cả 3 providers.

**Constraint bắt buộc:**
- Phải dùng 3 `RestClient` Bean riêng biệt, mỗi cái cấu hình `baseUrl` và `defaultHeaders` riêng cho từng provider.
- Phải implement `ClientHttpRequestInterceptor` tự động rotate API key (dùng key tiếp theo trong pool nếu nhận 429 Too Many Requests) — không được hardcode logic trong service method.
- Phải implement custom `ResponseErrorHandler` (hoặc `.onStatus()`) phân biệt: 401 Unauthorized (key invalid, remove khỏi pool), 429 Rate Limited (rotate key và retry), 5xx (throw exception với provider name).
- Phải dùng Apache HttpClient 5 connection pool với max 10 connections per provider.
- Phải viết unit test với `MockRestServiceServer` test đủ 3 scenario: success, key rotation khi 429, và pool exhausted khi tất cả keys đều 401.

---

## 13. Interview

**Core Q&A:**

Q: `RestClient` và `RestTemplate` dùng chung gì bên dưới?
A: Cả hai dùng chung: `ClientHttpRequestFactory` (underlying HTTP connection/pool), `HttpMessageConverter` (JSON/XML serialization), và `ClientHttpRequestInterceptor` (interceptor chain). `RestClient` thực chất là một fluent facade bao bọc lấy cùng infrastructure này. Bạn có thể tạo `RestClient` từ existing `RestTemplate` bằng `RestClient.create(restTemplate)` để tái sử dụng cấu hình.

Q: Sự khác biệt giữa `.body(User.class)` và `.toEntity(User.class)` là gì?
A: `.body(User.class)` trả về `User` trực tiếp (chỉ body). `.toEntity(User.class)` trả về `ResponseEntity<User>` (body + HTTP status code + response headers). Dùng `.toEntity()` khi cần kiểm tra status code hoặc đọc response headers (ví dụ: `Location` header sau POST tạo resource).

Q: Làm thế nào để xử lý response có generic type như `List<User>` với `RestClient`?
A: Dùng `ParameterizedTypeReference` để preserve generic type information ở runtime: `.body(new ParameterizedTypeReference<List<User>>() {})`. Anonymous subclass này giữ được generic parameter `User` thông qua Java reflection (`getGenericSuperclass()`), cho phép Jackson deserialize đúng kiểu.

Q: Làm thế nào để tự động thêm header `Authorization` vào mọi request của `RestClient`?
A: Hai cách: (1) `defaultHeader(HttpHeaders.AUTHORIZATION, "Bearer " + token)` tại thời điểm tạo Bean — phù hợp khi token không đổi (service account). (2) `requestInterceptor(interceptor)` với custom `ClientHttpRequestInterceptor` đọc token từ context mỗi request — phù hợp khi token là JWT của user hiện tại (lấy từ `SecurityContextHolder`).

Q: `RestClient` có hỗ trợ retry không?
A: `RestClient` không có built-in retry operator như `WebClient`. Để retry, dùng Spring Retry với `@Retryable` annotation trên service method hoặc `RetryTemplate` wrapper. Resilience4j cũng là option tốt cho retry + circuit breaker + timeout trong một chỗ.

**Scenario:**

Scenario: Bạn đang migrate từ `RestTemplate` sang `RestClient` trong một service lớn có 20+ method. Bạn tiếp cận thế nào để minimize risk?
Answer: Migration incremental: (1) Tạo `RestClient` Bean từ existing `RestTemplate`: `RestClient.create(existingRestTemplate)` — giữ nguyên connection pool và interceptors. (2) Migrate từng method một, bắt đầu từ method đơn giản nhất. (3) So sánh: `restTemplate.getForObject(url, User.class)` → `restClient.get().uri(url).retrieve().body(User.class)`. (4) Viết test cho mỗi method trước và sau migration. (5) Giữ cả hai Bean trong transition period nếu cần rollback từng phần.

Scenario: Bạn cần gọi một API cần authenticate bằng OAuth2 client credentials flow trước mỗi request (token expire sau 1 giờ). Bạn implement thế nào với `RestClient`?
Answer: Implement `ClientHttpRequestInterceptor` với token cache: interceptor giữ token và expiry timestamp. Trước mỗi request, kiểm tra xem token có còn hiệu lực không. Nếu hết hạn, gọi `RestClient` khác (hoặc `OAuth2AuthorizedClientManager` của Spring Security) để lấy token mới, cache lại, rồi thêm vào header. Dùng `synchronized` hoặc `ReentrantLock` để thread-safe khi nhiều thread cùng refresh token.

Scenario: Bạn cần gọi 3 APIs tuần tự: lấy user, sau đó lấy orders của user đó, sau đó tính tổng. Và cũng cần gọi một API độc lập (địa chỉ giao hàng) song song với flow trên. Bạn làm thế nào với `RestClient`?
Answer: Với `RestClient` (blocking), flow tuần tự là tự nhiên: `user = getUser(id)` → `orders = getOrders(user.getId())` → `total = calculateTotal(orders)`. Cho call độc lập song song, dùng `CompletableFuture`: `CompletableFuture<Address> addressFuture = CompletableFuture.supplyAsync(() -> getAddress(id), executor)`. Sau khi sequential flow xong, gọi `addressFuture.get(3, TimeUnit.SECONDS)` để lấy kết quả. Đây là pattern tốt cho mixed sequential + parallel trong blocking context. Nếu cần nhiều hơn, xem xét migrate call đó sang `WebClient` và dùng `Mono.zip`.

---

## 14. References

- Spring Framework Docs - RestClient: https://docs.spring.io/spring-framework/reference/integration/rest-clients.html#rest-restclient
- RestClient JavaDoc: https://docs.spring.io/spring-framework/docs/current/javadoc-api/org/springframework/web/client/RestClient.html
- Spring Blog - Introducing RestClient in Spring Framework 6.1: https://spring.io/blog/2023/07/13/new-in-spring-6-1-restclient
- Spring Boot Reference - Calling REST Services: https://docs.spring.io/spring-boot/docs/current/reference/html/io.html#io.rest-client.restclient
- Spring Framework 6.1 Release Notes: https://github.com/spring-projects/spring-framework/wiki/What%27s-New-in-Spring-Framework-6.x#61
- Java 21 Virtual Threads with Spring Boot: https://spring.io/blog/2023/09/20/virtual-threads-made-easy-with-spring-boot-3-2

---

## 15. Real-world Code

- Spring Framework RestClient tests — comprehensive usage patterns: https://github.com/spring-projects/spring-framework/tree/main/spring-webmvc/src/test/java/org/springframework/web/client
- Baeldung RestClient tutorial: https://github.com/eugenp/tutorials/tree/master/spring-web-modules/spring-resttemplate-2
- Spring Boot integration tests with RestClient: https://github.com/spring-projects/spring-boot/tree/main/spring-boot-tests/spring-boot-integration-tests
- Spring initializr source — uses RestClient internally: https://github.com/spring-io/start.spring.io/tree/main/start-client

---

## 16. Community

- Stack Overflow — "Spring RestClient vs RestTemplate — when to use": https://stackoverflow.com/questions/78029674/spring-restclient-vs-resttemplate
- Baeldung — Guide to Spring 6 RestClient: https://www.baeldung.com/spring-6-restclient
- Spring GitHub Discussions — RestClient feature discussion: https://github.com/spring-projects/spring-framework/issues/29552
- Reddit r/SpringBoot — "RestClient with Virtual Threads replaces need for WebClient": https://www.reddit.com/r/SpringBoot/comments/restclient_virtual_threads/
- Sébastien Deleuze (Spring committer) blog on RestClient: https://spring.io/blog/2023/07/13/new-in-spring-6-1-restclient
- DZone — "Spring RestClient: The Modern Way to Make HTTP Calls": https://dzone.com/articles/spring-restclient-modern-http-calls
