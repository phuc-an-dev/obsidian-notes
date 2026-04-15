---
created: 2026-04-15
tags:
  - "#type/concept"
  - "#status/draft"
  - "#lang/java"
related: "[[HTTP Clients in Spring]]"
---

## 1. What
**RestClient** là một HTTP Client đồng bộ (synchronous) hiện đại được giới thiệu từ Spring Framework 6.1 và Spring Boot 3.2. Nó cung cấp một API dạng Fluent (method chaining) tương tự như `WebClient` nhưng hoạt động trên mô hình Blocking truyền thống của `RestTemplate`.

## 2. Why (problem it solves)
- **Hệ thống hóa API**: Trước đây, Spring có `RestTemplate` (đồng bộ) và `WebClient` (bất đồng bộ) với hai kiểu viết code hoàn toàn khác nhau. `RestClient` mang trải nghiệm viết code hiện đại của WebClient vào thế giới đồng bộ.
- **Thay thế RestTemplate**: Giải quyết các hạn chế về thiết kế của `RestTemplate` (quá nhiều overloaded methods khó nhớ) bằng một interface linh hoạt và dễ mở rộng hơn.
- **Không cần WebFlux**: Cho phép lập trình viên sử dụng API hiện đại mà không cần thêm dependency `spring-boot-starter-webflux` vào các ứng dụng Servlet truyền thống.

## 3. Mental Model
> "Tưởng tượng `RestClient` như một cái **Nâng cấp nội thất** cho một chiếc xe cổ. Bạn vẫn dùng động cơ đốt trong (Blocking thread), nhưng bảng điều khiển, ghế ngồi và hệ thống giải trí (`Fluent API`, `Error Handling`) đều xịn xò và hiện đại như một chiếc xe điện (`WebClient`)."

## 4. Where it fits (architecture)
`Service Layer → [RestClient] → [HTTP Interceptors] → [Message Converters] → External API`

## 5. When to use
- Trong các ứng dụng Spring Boot mới (v3.2+) sử dụng Spring MVC.
- Khi muốn nâng cấp từ `RestTemplate` sang một phong cách code hiện đại hơn.
- Khi cần gọi API đồng bộ nhưng muốn xử lý lỗi (Error Handling) và Interceptors một cách tường minh, dễ đọc.

## 6. When NOT to use
- Trong các ứng dụng Reactive (Spring WebFlux) - hãy dùng `WebClient`.
- Trong các project cũ không thể nâng cấp lên Spring 6.1 hoặc Java 17+.

## 7. Trade-offs
| Pros | Cons |
|------|------|
| API dạng Fluent cực kỳ dễ đọc và bảo trì. | Vẫn là Blocking client (tốn thread/request). |
| Tích hợp sâu với Spring infrastructure. | Mới xuất hiện, cộng đồng và thư viện bổ trợ chưa nhiều bằng RestTemplate. |
| Không yêu cầu Project Reactor (Mono/Flux). | Cần Java 17 trở lên. |

## 8. Alternatives (with comparison)
| Option | So sánh |
|--------|---------|
| **RestTemplate** | Cũ, nhiều phương thức bị chồng chéo, khó mở rộng. |
| **WebClient** | Mạnh mẽ nhất, hỗ trợ Async, nhưng yêu cầu kiến thức Reactive. |
| **OpenFeign** | Tốt cho Microservices giao tiếp nội bộ qua Interface. |

## 9. How (minimal example)

### Khởi tạo RestClient (Bean)
```java
@Bean
public RestClient restClient(RestClient.Builder builder) {
    return builder.baseUrl("https://api.example.com").build();
}
```

### GET Request (Simple)
```java
User user = restClient.get()
  .uri("/users/{id}", 1)
  .accept(MediaType.APPLICATION_JSON)
  .retrieve()
  .onStatus(HttpStatusCode::is4xxClientError, (request, response) -> {
      throw new MyCustomException("Client error occurred");
  })
  .body(User.class);
```

### POST Request (with Headers)
```java
ResponseEntity<Void> response = restClient.post()
  .uri("/posts")
  .contentType(MediaType.APPLICATION_JSON)
  .body(new Post("Hello", "World"))
  .retrieve()
  .toBodilessEntity();
```

## 10. Production concerns
### Scaling
- Mặc định sử dụng `JdkClientHttpRequestFactory`. Nên cấu hình sang `HttpComponentsClientHttpRequestFactory` (Apache HttpClient) để có Connection Pooling tốt hơn.
### Failure
- Sử dụng `.onStatus()` để xử lý lỗi một cách chi tiết cho từng nhóm mã lỗi HTTP thay vì dùng try-catch bọc quanh toàn bộ lời gọi.
### Monitoring
- Hỗ trợ tốt cho Micrometer Observation, cho phép tự động thu thập metrics và tracing (Brave/Zipkin).

## 11. Common mistakes / anti-patterns
- ❌ **Mistake**: Quên xử lý lỗi dẫn đến `RestClientResponseException` trôi nổi trong ứng dụng.
  ✅ **Fix**: Sử dụng `.onStatus()` hoặc `.onStatus(status -> status.value() == 404, ...)` để handle cụ thể.
- ❌ **Mistake**: Nghĩ rằng `RestClient` là non-blocking vì code giống WebClient.
  ✅ **Fix**: Luôn nhớ nó sẽ block thread hiện tại cho đến khi nhận được response.

## 12. Sample project (with constraint)
**Tên project**: "Modern API Wrapper"
**Constraint**: Xây dựng một thư viện wrapper cho API của bên thứ ba. Phải hỗ trợ tự động đính kèm API Key vào header cho mọi request mà không được viết code lặp lại ở từng phương thức.
**Output**: Sử dụng `requestInterceptor` để thêm header "X-API-KEY" tự động.

## 13. Interview
### Core Q&A
1. **Q**: `RestClient` và `RestTemplate` có dùng chung cơ sở hạ tầng (infrastructure) không?
   **A**: Có. Cả hai đều sử dụng chung `ClientHttpRequestFactory`, `HttpMessageConverter` và `ClientHttpRequestInterceptor`. `RestClient` thực chất là một lớp vỏ (facade) hiện đại hơn bao bọc lấy các thành phần này.
2. **Q**: Làm sao để xử lý Response có kiểu Generic như `List<User>` trong RestClient?
   **A**: Giống như RestTemplate, ta sử dụng `ParameterizedTypeReference`: `.body(new ParameterizedTypeReference<List<User>>() {})`.
### Scenario
> "Tình huống: Project của bạn đang dùng Java 21 với Virtual Threads. Bạn chọn WebClient hay RestClient?"
**A**: Em sẽ ưu tiên chọn **RestClient**. Vì Virtual Threads xử lý các Blocking IO cực kỳ hiệu quả mà không tốn tài nguyên thread thật (Platform Thread). RestClient giúp code đơn giản hơn nhiều so với WebClient/Reactive mà vẫn đạt được performance tương đương trong môi trường Virtual Threads.
