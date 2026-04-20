---
created: 2026-04-17
tags:
  - "#type/library"
  - "#status/draft"
  - "#lang/spring"
  - "#topic/http"
related:
  - "[[HTTP Clients in Spring]]"
  - "[[RestClient in Spring Boot]]"
---

# RestTemplate in Spring Boot

## 1. What

`RestTemplate` là class HTTP client đồng bộ (synchronous, blocking) trung tâm của Spring Framework, dùng để thực hiện HTTP request từ phía server đến các REST API bên ngoài. Nó tự động serialize/deserialize Java objects sang JSON/XML thông qua `HttpMessageConverter`, che giấu toàn bộ chi tiết low-level về socket và connection management. Từ Spring 5.0, `RestTemplate` đã bị đưa vào chế độ maintenance và Spring khuyến nghị chuyển sang `RestClient` (Spring 6.1+) hoặc `WebClient` cho reactive context.

---

## 2. Why

Trước `RestTemplate`, developer phải dùng `HttpURLConnection` của Java SE — vô cùng verbose, dễ lỗi, và không có bất kỳ tích hợp nào với Spring:

```java
// Cách cũ không có RestTemplate — chỉ để minh họa sự phức tạp
URL url = new URL("https://api.example.com/users/1");
HttpURLConnection conn = (HttpURLConnection) url.openConnection();
conn.setRequestMethod("GET");
conn.setRequestProperty("Accept", "application/json");
BufferedReader br = new BufferedReader(new InputStreamReader(conn.getInputStream()));
StringBuilder sb = new StringBuilder();
String line;
while ((line = br.readLine()) != null) sb.append(line);
br.close();
// Sau đó phải tự parse JSON bằng ObjectMapper...
```

`RestTemplate` ra đời để giải quyết: không cần quản lý connection thủ công, tự động convert JSON sang POJO, tích hợp với Spring's error handling, interceptors, và authentication. Nó trở thành standard HTTP client của Spring trong nhiều năm từ Spring 3.0 đến Spring 5.x.

---

## 3. Mental Model

Hãy tưởng tượng `RestTemplate` như một **nhân viên bưu điện có kinh nghiệm**. Bạn đưa cho anh ta một mẩu giấy ghi: "Lấy hộ tôi gói hàng `User` từ địa chỉ `https://api.example.com/users/1`." Anh ta biết cách đến đó (mở connection), biết phải gõ cửa ra sao (HTTP protocol), biết cách đọc và dịch nội dung gói hàng (JSON deserialization), và mang về đúng loại hộp bạn yêu cầu (`User.class`). Tuy nhiên, trong suốt quá trình đó, bạn phải ngồi đợi anh ta về — thread của bạn bị block. Đây là nhân viên đáng tin nhưng chỉ xử lý một việc một lần.

---

## 4. Where it fits

```
[Service Layer]
      |
      v
[RestTemplate]
      |
      |-- [ClientHttpRequestInterceptor]  (logging, auth header injection)
      |
      |-- [HttpMessageConverter]          (JSON/XML <-> Java POJO)
      |
      |-- [ClientHttpRequestFactory]      (connection pool, timeout config)
      |         |
      |         |-- SimpleClientHttpRequestFactory (default, no pool)
      |         |-- HttpComponentsClientHttpRequestFactory (Apache HttpClient, pooling)
      |         |-- OkHttp3ClientHttpRequestFactory (OkHttp)
      |
      v
[External REST API]
```

---

## 5. When to use

- Trong các dự án legacy đang chạy Spring Boot 1.x, 2.x mà chưa có kế hoạch migrate.
- Khi cần maintain codebase cũ: thay đổi sang `RestClient` là tốt nhưng không phải ưu tiên nếu code đang stable.
- Khi team cần onboard nhanh và đã quen với `RestTemplate` — tài liệu và ví dụ trên internet rất phong phú.
- Khi viết integration test với `MockRestServiceServer` để mock external API calls mà không cần thay đổi production code.

---

## 6. When NOT to use

Không nên dùng `RestTemplate` cho project mới từ Spring Boot 3.x trở đi — dùng `RestClient` thay thế. `RestTemplate` không nhận feature mới, và API của nó có nhiều overloaded method gây nhầm lẫn:

```
getForObject vs getForEntity vs exchange vs execute
postForObject vs postForEntity vs postForLocation
```

Không dùng trong môi trường reactive (Spring WebFlux) — `RestTemplate` là blocking, không tương thích với reactive pipeline, và gọi nó từ Netty event loop thread sẽ block event loop.

Không dùng khi cần high concurrency mà không có Virtual Threads — mỗi request giữ một thread trong toàn bộ thời gian chờ, dẫn đến thread pool exhaustion khi tải cao.

---

## 7. Trade-offs

| Pros | Cons |
|---|---|
| API đơn giản, ít abstraction layer | Deprecated, không nhận feature mới từ Spring 5.0 |
| Tài liệu và StackOverflow answers cực kỳ phong phú | Blocking per request — tốn thread tài nguyên khi concurrency cao |
| Có sẵn trong `spring-boot-starter-web`, không cần thêm dependency | Nhiều overloaded method dễ nhầm — khó nhớ cái nào dùng cho trường hợp nào |
| `MockRestServiceServer` cho integration test rất tiện | Verbose khi cần handle generic type (`ParameterizedTypeReference`) |
| Tích hợp tốt với Spring Security OAuth2 RestTemplate (legacy) | Cấu hình connection pool phức tạp, phải đổi `ClientHttpRequestFactory` |

---

## 8. Alternatives

| Option | So sánh với RestTemplate |
|---|---|
| `RestClient` (Spring 6.1+) | Thay thế trực tiếp — cùng blocking model, cùng infrastructure, API fluent đẹp hơn nhiều |
| `WebClient` | Non-blocking, reactive — cho high concurrency, nhưng cần học Reactor |
| `Feign Client` | Declarative interface — ít code hơn nhưng cần Spring Cloud |
| `OkHttp` thuần | Lightweight, không có Spring integration tự động |
| `java.net.http.HttpClient` (JDK 11+) | No-dependency option, hỗ trợ async, nhưng verbose |

---

## 9. How

```java
// =============================================
// 1. Cấu hình Bean với connection pool (Production-ready)
// =============================================
@Configuration
public class RestTemplateConfig {

    @Bean
    public RestTemplate restTemplate(RestTemplateBuilder builder) {
        // Dùng Apache HttpClient với connection pool
        HttpComponentsClientHttpRequestFactory factory =
            new HttpComponentsClientHttpRequestFactory();
        factory.setConnectTimeout(3000);         // 3 giây connect timeout
        factory.setReadTimeout(10000);           // 10 giây read timeout

        return builder
            .requestFactory(() -> factory)
            .additionalInterceptors(new LoggingInterceptor()) // custom interceptor
            .build();
    }
}

// Interceptor để log request/response
public class LoggingInterceptor implements ClientHttpRequestInterceptor {
    private static final Logger log = LoggerFactory.getLogger(LoggingInterceptor.class);

    @Override
    public ClientHttpResponse intercept(HttpRequest request, byte[] body,
            ClientHttpRequestExecution execution) throws IOException {
        log.info("HTTP {} {}", request.getMethod(), request.getURI());
        ClientHttpResponse response = execution.execute(request, body);
        log.info("Response status: {}", response.getStatusCode());
        return response;
    }
}


// =============================================
// 2. Các method phổ biến
// =============================================
@Service
public class UserService {

    @Autowired
    private RestTemplate restTemplate;

    private static final String BASE_URL = "https://jsonplaceholder.typicode.com";

    // GET - trả về object trực tiếp (không có status code, headers)
    public User getUser(Long id) {
        return restTemplate.getForObject(BASE_URL + "/users/{id}", User.class, id);
    }

    // GET - trả về ResponseEntity (có status code, headers, body)
    public ResponseEntity<User> getUserWithStatus(Long id) {
        return restTemplate.getForEntity(BASE_URL + "/users/{id}", User.class, id);
    }

    // GET - Generic type (List, Map, etc.) — PHẢI dùng exchange + ParameterizedTypeReference
    // Lý do: Type Erasure của Java — List.class không mang thông tin <User>
    public List<User> getAllUsers() {
        ResponseEntity<List<User>> response = restTemplate.exchange(
            BASE_URL + "/users",
            HttpMethod.GET,
            null,
            new ParameterizedTypeReference<List<User>>() {}
        );
        return response.getBody();
    }

    // POST - gửi object, nhận về location header
    public URI createUserGetLocation(User user) {
        return restTemplate.postForLocation(BASE_URL + "/users", user);
    }

    // POST - gửi object, nhận về entity mới tạo
    public User createUser(User user) {
        ResponseEntity<User> response = restTemplate.postForEntity(
            BASE_URL + "/users", user, User.class);
        return response.getBody();
    }

    // PUT - update (không có return body thường)
    public void updateUser(Long id, User user) {
        restTemplate.put(BASE_URL + "/users/{id}", user, id);
    }

    // DELETE
    public void deleteUser(Long id) {
        restTemplate.delete(BASE_URL + "/users/{id}", id);
    }

    // exchange() - method linh hoạt nhất: tùy chỉnh method, headers, body
    public User getUserWithCustomHeader(Long id, String authToken) {
        HttpHeaders headers = new HttpHeaders();
        headers.set("Authorization", "Bearer " + authToken);
        headers.setAccept(List.of(MediaType.APPLICATION_JSON));

        HttpEntity<Void> requestEntity = new HttpEntity<>(headers);

        ResponseEntity<User> response = restTemplate.exchange(
            BASE_URL + "/users/{id}",
            HttpMethod.GET,
            requestEntity,
            User.class,
            id
        );
        return response.getBody();
    }
}


// =============================================
// 3. Custom Error Handler
// =============================================
@Component
public class CustomResponseErrorHandler implements ResponseErrorHandler {

    @Override
    public boolean hasError(ClientHttpResponse response) throws IOException {
        return response.getStatusCode().is4xxClientError()
            || response.getStatusCode().is5xxServerError();
    }

    @Override
    public void handleError(ClientHttpResponse response) throws IOException {
        if (response.getStatusCode() == HttpStatus.NOT_FOUND) {
            throw new ResourceNotFoundException("Resource not found");
        }
        if (response.getStatusCode().is5xxServerError()) {
            throw new ExternalServiceException("External service error: "
                + response.getStatusCode());
        }
    }
}

// Đăng ký error handler vào Bean config
@Bean
public RestTemplate restTemplate(RestTemplateBuilder builder) {
    return builder
        .errorHandler(new CustomResponseErrorHandler())
        .build();
}


// =============================================
// 4. Integration test với MockRestServiceServer
// =============================================
@SpringBootTest
class UserServiceTest {

    @Autowired private UserService userService;
    @Autowired private RestTemplate restTemplate;

    private MockRestServiceServer mockServer;

    @BeforeEach
    void setUp() {
        mockServer = MockRestServiceServer.createServer(restTemplate);
    }

    @Test
    void getUser_shouldReturnUser() throws Exception {
        mockServer.expect(requestTo("https://jsonplaceholder.typicode.com/users/1"))
            .andExpect(method(HttpMethod.GET))
            .andRespond(withSuccess(
                """
                {"id": 1, "name": "John Doe", "email": "john@example.com"}
                """,
                MediaType.APPLICATION_JSON));

        User user = userService.getUser(1L);

        assertThat(user.getName()).isEqualTo("John Doe");
        mockServer.verify();
    }
}
```

---

## 10. Production concerns

**Connection Pooling:**
Mặc định `RestTemplate` dùng `SimpleClientHttpRequestFactory` không có connection pool — mỗi request mở và đóng một TCP connection mới. Điều này tốn kém về latency (TCP handshake) và tài nguyên. Trong production, luôn cấu hình `HttpComponentsClientHttpRequestFactory` với Apache HttpClient hoặc `OkHttp3ClientHttpRequestFactory`:

```java
// Apache HttpClient với connection pool
PoolingHttpClientConnectionManager connectionManager =
    new PoolingHttpClientConnectionManager();
connectionManager.setMaxTotal(100);         // max 100 connections tổng
connectionManager.setDefaultMaxPerRoute(20); // max 20 connections per host

CloseableHttpClient httpClient = HttpClients.custom()
    .setConnectionManager(connectionManager)
    .evictExpiredConnections()
    .evictIdleConnections(30, TimeUnit.SECONDS)
    .build();

HttpComponentsClientHttpRequestFactory factory =
    new HttpComponentsClientHttpRequestFactory(httpClient);
factory.setConnectTimeout(3000);
factory.setReadTimeout(10000);
```

**Failure modes:**
- Thread pool exhaustion: nếu external service chậm (latency cao), threads bị giữ lâu. Cần kết hợp với circuit breaker (Resilience4j) để fail fast và timeout ngắn để giải phóng thread sớm.
- Silent data loss: nếu không xử lý 4xx/5xx error đúng cách, request thất bại có thể bị bỏ qua mà không có exception. Cần `ResponseErrorHandler` tùy chỉnh.
- Memory leak: nếu không consume response body (ngay cả khi error), connection không được trả về pool. Đặc biệt quan trọng khi dùng `exchange()` — phải đọc hoặc close response.

**Monitoring:**
- Tích hợp `MetricsClientHttpRequestInterceptor` của Micrometer để tự động record `http.client.requests` với labels: URI, method, status.
- Bật Spring Boot Actuator `/actuator/metrics/http.client.requests`.
- Sử dụng `ClientHttpRequestInterceptor` để log request ID để correlate với distributed tracing.

---

## 11. Common mistakes

- Mistake: Dùng `getForObject(url, List.class)` để nhận danh sách objects.
  Fix: Jackson bị Type Erasure nên sẽ deserialize thành `List<LinkedHashMap>` thay vì `List<User>`. Phải dùng `exchange()` với `new ParameterizedTypeReference<List<User>>() {}`.

- Mistake: Không cấu hình timeout — dùng `RestTemplate` mặc định không có timeout.
  Fix: Luôn set `connectTimeout` và `readTimeout`. `SimpleClientHttpRequestFactory` mặc định timeout là 0 (vô hạn). Một external API bị treo có thể giữ thread mãi mãi.

- Mistake: Khởi tạo `new RestTemplate()` bên trong method service thay vì inject Bean.
  Fix: Khai báo `@Bean` trong config class và inject. `new RestTemplate()` tạo connection pool mới mỗi lần, không tái sử dụng, gây resource leak.

- Mistake: Dùng string concatenation để build URL thay vì dùng URI template hoặc `UriComponentsBuilder`.
  Fix: Dùng `restTemplate.getForObject("/users/{id}", User.class, id)` với URI template, hoặc `UriComponentsBuilder.fromHttpUrl(BASE_URL).path("/users/{id}").buildAndExpand(id).toUri()`. String concatenation dễ gây lỗi encoding và injection.

- Mistake: Bỏ qua việc set `Content-Type` khi POST/PUT với `postForObject()`.
  Fix: Dùng `HttpEntity` với `HttpHeaders` để set `Content-Type: application/json`, hoặc dùng `postForEntity()` vì Spring sẽ tự detect content type từ `HttpMessageConverter`.

---

## 12. Sample project

**Tên project: GitHub Repository Stats Dashboard**

Xây dựng một Spring Boot app expose endpoint `/dashboard/user/{username}` trả về thông tin tổng hợp về một GitHub user: profile, danh sách public repos, và tổng số stars.

**Constraint bắt buộc:**
- Phải dùng `RestTemplate` với Apache HttpClient connection pool (không dùng `new RestTemplate()` mặc định).
- Phải implement `ResponseErrorHandler` tùy chỉnh xử lý 3 trường hợp: 404 (user không tồn tại), 403 (rate limited), và 5xx (GitHub server error) — mỗi loại throw exception khác nhau.
- Phải dùng `ClientHttpRequestInterceptor` để automatically inject `Authorization: token {github_token}` vào mọi request từ `application.properties`.
- Phải viết integration test dùng `MockRestServiceServer` cho ít nhất 3 scenarios: success, user not found, rate limited.

---

## 13. Interview

**Core Q&A:**

Q: `exchange()` khác gì với `getForObject()`, `getForEntity()`, và `postForEntity()`?
A: `getForObject()` trả về body trực tiếp (no status code). `getForEntity()` và `postForEntity()` trả về `ResponseEntity` (body + status + headers). `exchange()` là method linh hoạt nhất: cho phép chỉ định bất kỳ HTTP method nào, custom `HttpEntity` (headers + body), và kiểu return type kể cả generic (`ParameterizedTypeReference`). Dùng `exchange()` khi cần kiểm soát full request/response hoặc khi làm việc với generic types.

Q: Tại sao `getForObject(url, List.class)` nguy hiểm?
A: Type Erasure của Java khiến thông tin generic type bị mất lúc runtime. Jackson nhận `List.class` mà không biết element type là `User`, nên nó deserialize thành `List<LinkedHashMap>`. Khi bạn cast element sang `User` sẽ throw `ClassCastException`. Giải pháp: dùng `exchange()` với `new ParameterizedTypeReference<List<User>>() {}` — anonymous subclass này giữ được generic type information ở runtime thông qua reflection.

Q: `RestTemplate` có thread-safe không?
A: Có, `RestTemplate` instance là thread-safe sau khi được cấu hình. Đây là lý do tại sao nên khai báo nó là singleton Bean và inject thay vì tạo mới mỗi lần. Các `HttpMessageConverter` và `ClientHttpRequestInterceptor` cũng phải thread-safe.

Q: Làm thế nào để thêm JWT Bearer token vào mọi request mà không lặp code?
A: Implement `ClientHttpRequestInterceptor` và đăng ký vào `RestTemplate`:
```java
restTemplate.getInterceptors().add((request, body, execution) -> {
    request.getHeaders().set("Authorization", "Bearer " + getToken());
    return execution.execute(request, body);
});
```
Hoặc cấu hình trong `RestTemplateBuilder.additionalInterceptors()`.

Q: Sự khác biệt giữa `connectTimeout` và `readTimeout` là gì?
A: `connectTimeout` là thời gian tối đa để thiết lập TCP connection đến server. `readTimeout` là thời gian tối đa để nhận response data sau khi connection đã được thiết lập. Một request có thể connect thành công (dưới connectTimeout) nhưng vẫn bị timeout ở read nếu server xử lý chậm.

**Scenario:**

Scenario: Bạn cần gọi một API trả về `List<Map<String, Object>>` và bạn muốn map từng element sang một custom DTO. Bạn làm thế nào với RestTemplate?
Answer: Dùng `exchange()` với `ParameterizedTypeReference<List<Map<String, Object>>>()`. Sau đó dùng `ObjectMapper` hoặc stream để convert từng Map sang DTO: `mapper.convertValue(map, MyDto.class)`. Hoặc tốt hơn, tạo `List<MyDto>` trực tiếp nếu server trả về đúng format: `new ParameterizedTypeReference<List<MyDto>>() {}`.

Scenario: External API có lúc trả về 200 nhưng body là `{"error": "not found"}` thay vì 404. RestTemplate không throw exception. Bạn xử lý thế nào?
Answer: Implement custom `ResponseErrorHandler.hasError()` để kiểm tra cả response body, không chỉ status code. Dùng `response.getBody()` để đọc body khi status là 200 và check field `error`. Hoặc sau khi nhận object, validate field `error` trong service layer và throw business exception tương ứng.

Scenario: `RestTemplate` đang bị OutOfMemoryError khi download file lớn. Nguyên nhân và giải pháp?
Answer: Mặc định `RestTemplate.getForObject()` load toàn bộ response body vào memory. Với file lớn, dùng `execute()` với `ResponseExtractor` để stream response trực tiếp vào file: `restTemplate.execute(url, HttpMethod.GET, null, response -> { Files.copy(response.getBody(), filePath); return null; })`. Điều này stream data mà không buffer vào heap.

---

## 14. References

- Spring Framework Docs - RestTemplate: https://docs.spring.io/spring-framework/reference/integration/rest-clients.html#rest-resttemplate
- RestTemplate JavaDoc: https://docs.spring.io/spring-framework/docs/current/javadoc-api/org/springframework/web/client/RestTemplate.html
- Spring Boot Reference - RestTemplateBuilder: https://docs.spring.io/spring-boot/docs/current/reference/html/io.html#io.rest-client.resttemplate
- Apache HttpClient 5 Connection Management: https://hc.apache.org/httpcomponents-client-5.2.x/current/httpclient5/apidocs/org/apache/hc/client5/http/impl/io/PoolingHttpClientConnectionManager.html
- Spring Security OAuth2 RestTemplate (legacy): https://docs.spring.io/spring-security/reference/servlet/oauth2/client/authorized-clients.html

---

## 15. Real-world Code

- Spring PetClinic (sử dụng RestTemplate trong microservices branch): https://github.com/spring-petclinic/spring-petclinic-microservices
- Baeldung RestTemplate tutorials — toàn diện với nhiều ví dụ: https://github.com/eugenp/tutorials/tree/master/spring-web-modules/spring-resttemplate
- Spring Framework Test source — MockRestServiceServer patterns: https://github.com/spring-projects/spring-framework/tree/main/spring-test/src/main/java/org/springframework/test/web/client
- Spring Boot Actuator samples với metrics: https://github.com/spring-projects/spring-boot/tree/main/spring-boot-tests

---

## 16. Community

- Stack Overflow — "RestTemplate getForObject with List": https://stackoverflow.com/questions/23674046/get-list-of-json-objects-with-spring-resttemplate
- Stack Overflow — "How to set timeout with Spring RestTemplate": https://stackoverflow.com/questions/13837012/spring-resttemplate-timeout
- Baeldung — Guide to RestTemplate: https://www.baeldung.com/rest-template
- Baeldung — MockRestServiceServer Tutorial: https://www.baeldung.com/spring-mock-rest-template
- Stack Overflow — "RestTemplate thread safety": https://stackoverflow.com/questions/22989500/is-resttemplate-thread-safe
- Reddit r/SpringBoot — "When will RestTemplate actually be removed": https://www.reddit.com/r/java/comments/resttemplate_removal_discussion/
