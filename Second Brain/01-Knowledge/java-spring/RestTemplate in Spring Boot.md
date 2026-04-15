---
created: 2026-04-15
tags:
  - "#type/concept"
  - "#status/draft"
  - "#lang/java"
related: "[[HTTP Clients in Spring]]"
---

## 1. What
**RestTemplate** là một class trung tâm của Spring Framework dùng để thực hiện các HTTP requests phía client. nó cung cấp các phương thức tiện ích để tương tác với RESTful services một cách đồng bộ (blocking) và trừu tượng hóa việc xử lý HTTP.

## 2. Why (problem it solves)
- Loại bỏ code "boilerplate" khi sử dụng `HttpURLConnection` truyền thống của Java.
- Tự động chuyển đổi giữa JSON/XML và Java Objects thông qua `HttpMessageConverter`.
- Hỗ trợ đầy đủ các phương thức HTTP (GET, POST, PUT, DELETE, PATCH).
- Dễ dàng tích hợp với Spring Error Handling và Interceptors.

## 3. Mental Model
> "Tưởng tượng `RestTemplate` như một cái **Điều khiển từ xa (Remote Control)** đa năng. Bạn không cần biết sóng hồng ngoại hay bluetooth hoạt động ra sao (low-level socket), bạn chỉ cần nhấn nút 'Bật' (`GET`), 'Chuyển kênh' (`POST`) và cái TV (`Server`) sẽ phản hồi lại kết quả bạn muốn."

## 4. Where it fits (architecture)
`Service Layer → [RestTemplate] → [HttpMessageConverters] → External REST API`

## 5. When to use
- Trong các ứng dụng Spring Boot truyền thống (Servlet-based) nơi mô hình đồng bộ (1 request/1 thread) là đủ.
- Khi cần gọi các API bên ngoài một cách đơn giản và nhanh chóng.
- Trong các dự án legacy (di sản) đã sử dụng `RestTemplate` từ trước.

## 6. When NOT to use
- **Maintenance Mode**: Từ Spring 5.0, `RestTemplate` đã được đưa vào chế độ bảo trì. Spring khuyên dùng `WebClient` (Reactive) hoặc `RestClient` (Spring 6.1+).
- Trong các ứng dụng High-concurrency hoặc Reactive (WebFlux) vì nó sẽ chặn (block) thread.

## 7. Trade-offs
| Pros | Cons |
|------|------|
| API cực kỳ đơn giản và dễ hiểu. | Chế độ Blocking gây tốn tài nguyên thread. |
| Tài liệu và cộng đồng hỗ trợ rất lớn. | Không hỗ trợ tốt cho Streaming dữ liệu lớn. |
| Tích hợp sẵn trong `spring-boot-starter-web`. | Đang dần bị thay thế bởi các công cụ hiện đại hơn. |

## 8. Alternatives (with comparison)
| Option | So sánh |
|--------|---------|
| **WebClient** | Non-blocking, hỗ trợ cả sync và async, mạnh mẽ hơn. |
| **RestClient** | Cùng là đồng bộ nhưng có API dạng Fluent hiện đại hơn. |
| **Feign Client** | Cách tiếp cận hướng khai báo (Declarative), code sạch hơn. |

## 9. How (minimal example)

### Cấu hình Bean (Khuyên dùng)
```java
@Configuration
public class AppConfig {
    @Bean
    public RestTemplate restTemplate(RestTemplateBuilder builder) {
        return builder
            .setConnectTimeout(Duration.ofSeconds(3))
            .setReadTimeout(Duration.ofSeconds(3))
            .build();
    }
}
```

### Sử dụng trong Service
```java
@Service
public class PostService {
    @Autowired
    private RestTemplate restTemplate;

    public Post getPost(Long id) {
        String url = "https://jsonplaceholder.typicode.com/posts/" + id;
        return restTemplate.getForObject(url, Post.class);
    }

    public void createPost(Post post) {
        String url = "https://jsonplaceholder.typicode.com/posts";
        ResponseEntity<Post> response = restTemplate.postForEntity(url, post, Post.class);
        System.out.println("Status: " + response.getStatusCode());
    }
}
```

## 10. Production concerns
### Scaling
- Mặc định `RestTemplate` không sử dụng Connection Pool. Trong production, nên sử dụng `HttpComponentsClientHttpRequestFactory` kết hợp với Apache `HttpClient` để quản lý connection.
### Failure
- Luôn phải cấu hình **Timeouts**. Nếu không, một API chậm có thể làm cạn kiệt thread pool của toàn bộ ứng dụng.
### Monitoring
- Sử dụng `ClientHttpRequestInterceptor` để log URL, execution time và status code cho mọi request.

## 11. Common mistakes / anti-patterns
- ❌ **Mistake**: Sử dụng `getForObject` nhưng sau đó lại cần kiểm tra Status Code hoặc Headers.
  ✅ **Fix**: Sử dụng `getForEntity` hoặc `exchange` để lấy toàn bộ `ResponseEntity`.
- ❌ **Mistake**: Không xử lý ngoại lệ `RestClientException`.
  ✅ **Fix**: Bao quanh các lời gọi bằng `try-catch` hoặc sử dụng `ResponseErrorHandler` tùy chỉnh.

## 12. Sample project (with constraint)
**Tên project**: "Stock Price Notifier"
**Constraint**: Phải gọi API lấy giá chứng khoán mỗi 10 giây. Nếu API lỗi hoặc quá 2 giây không phản hồi, phải chuyển sang dùng giá dự phòng (Fallback Price) từ Database.
**Output**: Console log giá cổ phiếu cập nhật liên tục.

## 13. Interview
### Core Q&A
1. **Q**: `exchange()` khác gì với `getForObject()` hay `postForEntity()`?
   **A**: `exchange()` là phương thức linh hoạt nhất, cho phép tùy chỉnh mọi thứ: HttpMethod, HttpEntity (headers + body), và kiểu dữ liệu trả về (ParameterizedTypeReference cho List/Generic).
2. **Q**: Làm sao để gửi Custom Header trong `RestTemplate`?
   **A**: Tạo `HttpHeaders`, thêm header, bọc vào `HttpEntity`, sau đó truyền vào phương thức `exchange()`.
### Scenario
> "Tình huống: Bạn cần gọi một API trả về một danh sách `List<User>`. Tại sao `getForObject(url, List.class)` lại không an toàn và bạn giải quyết thế nào?"
**A**: Dùng `List.class` sẽ bị lỗi **Type Erasure**, Jackson sẽ deserialize thành `List<LinkedHashMap>` thay vì `List<User>`. Giải pháp là dùng `exchange()` với `new ParameterizedTypeReference<List<User>>() {}`.
