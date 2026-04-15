---
created: 2026-04-15
tags:
  - "#type/concept"
  - "#status/draft"
  - "#lang/java"
related: "[[Spring HTTP Core Classes]]"
---

## 1. What
**HTTP Client** trong Spring là các công cụ (libraries/utilities) cho phép ứng dụng Java gửi request đến các dịch vụ bên ngoài (External APIs) hoặc giao tiếp giữa các Microservices thông qua giao thức HTTP.

## 2. Why (problem it solves)
- Cho phép hệ thống lấy dữ liệu từ bên ngoài (ví dụ: lấy giá vàng, thời tiết, tỷ giá).
- Kết nối các dịch vụ trong kiến trúc Microservices.
- Tự động hóa việc Serialize/Deserialize dữ liệu (JSON/XML -> POJO) thay vì xử lý chuỗi thủ công.
- Quản lý Connection Pooling, Timeouts và Retries một cách tập trung.

## 3. Mental Model
> "Hãy tưởng tượng HTTP Client như một **Người đưa thư** (Messenger). 
> 1. Bạn viết một bức thư (`Request`).
> 2. Đưa cho người đưa thư (`HTTP Client`).
> 3. Người đưa thư đi đến địa chỉ đích (`URL`).
> 4. Người đưa thư mang thư phản hồi về (`Response`) cho bạn."

## 4. Where it fits (architecture)
`Service Layer → [HTTP Client] → External API / Microservice`

## 5. When to use
- Khi cần gọi REST API từ một provider khác (Stripe, Google Maps, v.v.).
- Khi triển khai giao tiếp Synchronous (đồng bộ) hoặc Asynchronous (bất đồng bộ) giữa các services.
- Khi cần mock dữ liệu từ API bên ngoài trong integration tests.

## 6. When NOT to use
- Nếu giao tiếp giữa các services cần độ trễ cực thấp và streaming mạnh mẽ (nên dùng gRPC).
- Nếu giao tiếp là Event-driven (nên dùng Kafka/RabbitMQ).
- Nếu chỉ gọi các phương thức trong cùng một ứng dụng (Monolith).

## 7. Trade-offs
| Client | Mô hình | Ưu điểm | Nhược điểm |
|--------|---------|---------|------------|
| **RestTemplate** | Blocking | Đơn giản, quen thuộc. | Hiệu năng kém khi concurrency cao (1 thread/request). |
| **WebClient** | Non-blocking | Hiệu năng cực cao, hỗ trợ Streaming. | Độ phức tạp cao (Project Reactor/Flux/Mono). |
| **RestClient** | Blocking (Modern) | API dạng Fluent (giống WebClient) nhưng đồng bộ. | Mới (từ Spring 6.1+), cần Java 17+. |

## 8. Alternatives (with comparison)
| Option | Đặc điểm |
|--------|----------|
| **OpenFeign** | Declarative (chỉ cần viết Interface), rất phổ biến trong Spring Cloud. |
| **Retrofit** | Thường dùng cho Android, cũng có thể dùng trong Java backend. |
| **Apache HttpClient** | Low-level, cấu hình cực sâu nhưng code dài dòng. |

## 9. How (minimal example)

### A. RestTemplate (Truyền thống)
```java
RestTemplate restTemplate = new RestTemplate();
Quote quote = restTemplate.getForObject("https://api.quotable.io/random", Quote.class);
```

### B. RestClient (Spring 6.1+ Modern Synchronous)
```java
RestClient restClient = RestClient.create();
String result = restClient.get()
  .uri("https://api.example.com/data")
  .retrieve()
  .body(String.class);
```

### C. WebClient (Reactive)
```java
WebClient webClient = WebClient.create();
Mono<String> response = webClient.get()
  .uri("https://api.example.com/data")
  .retrieve()
  .bodyToMono(String.class);
```

## 10. Production concerns
### Scaling
- **Connection Pool**: Phải cấu hình Connection Pool (ví dụ: thông qua Apache HttpComponents) để tránh việc mở/đóng socket quá nhiều gây overhead.
### Failure
- **Timeouts**: Luôn phải set `ConnectTimeout` và `ReadTimeout`. Mặc định của một số client là vô hạn (infinite), có thể làm treo toàn bộ ứng dụng nếu API đích bị chậm.
- **Circuit Breaker**: Nên kết hợp với Resilience4j để ngắt kết nối khi service đích bị lỗi liên tục.

## 11. Common mistakes / anti-patterns
- ❌ **Mistake**: Khởi tạo `new RestTemplate()` bên trong mỗi phương thức.
  ✅ **Fix**: Khai báo `RestTemplate` như một Bean để tái sử dụng connection pool.
- ❌ **Mistake**: Quên xử lý lỗi khi API trả về 4xx hoặc 5xx.
  ✅ **Fix**: Sử dụng `ResponseErrorHandler` hoặc block `try-catch` với `HttpStatusCodeException`.

## 12. Sample project (with constraint)
**Tên project**: "Multi-source Weather Aggregator"
**Constraint**: Gọi cùng lúc 3 API thời tiết khác nhau. Nếu 1 cái lỗi, vẫn phải trả về kết quả của 2 cái còn lại. Không được block thread chính quá 2 giây.
**Output**: Một JSON tổng hợp thông tin thời tiết trung bình từ các nguồn.

## 13. Interview
### Core Q&A
1. **Q**: Tại sao Spring khuyên dùng `WebClient` thay vì `RestTemplate`?
   **A**: Vì `RestTemplate` sẽ đi vào chế độ maintenance mode. `WebClient` hỗ trợ cả đồng bộ và bất đồng bộ, hiệu năng tốt hơn trong môi trường scale lớn nhờ cơ chế non-blocking.
2. **Q**: Làm sao để log lại body của cả request và response khi dùng HTTP Client?
   **A**: Có thể sử dụng `ClientHttpRequestInterceptor` (cho RestTemplate/RestClient) hoặc `ExchangeFilterFunction` (cho WebClient).
### Scenario
> "Tình huống: Bạn cần gọi một API mà họ giới hạn 10 request/giây (Rate Limit). Bạn xử lý thế nào?"
**A**: Em sẽ sử dụng thư viện **Resilience4j RateLimiter** bọc ngoài HTTP client call. Nếu vượt quá giới hạn, ứng dụng sẽ đợi hoặc trả về lỗi ngay lập tức thay vì spam server đối phương dẫn đến bị ban IP.
