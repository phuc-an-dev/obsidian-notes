---
created: 2026-04-15
tags:
  - "#type/concept"
  - "#status/draft"
  - "#lang/java"
related: "[[HTTP Clients in Spring]]"
---

## 1. What
**WebClient** là một HTTP Client hiện đại, không chặn (non-blocking) và có tính phản ứng (reactive), thuộc module `Spring WebFlux`. Nó hỗ trợ cả lập trình đồng bộ và bất đồng bộ, cho phép truyền dữ liệu dạng streaming.

## 2. Why (problem it solves)
- **Cải thiện hiệu năng**: Khác với `RestTemplate` (1 thread/request), `WebClient` sử dụng cơ chế Event Loop, cho phép một lượng nhỏ thread xử lý hàng ngàn request đồng thời.
- **Resource Efficiency**: Giảm thiểu việc lãng phí bộ nhớ và CPU cho việc duy trì các thread đang ở trạng thái chờ (Wait/Idle).
- **Reactive Stream**: Tích hợp hoàn hảo với Project Reactor (`Mono`, `Flux`), cho phép xử lý dữ liệu ngay khi nó đang được tải về thay vì đợi tải xong toàn bộ.

## 3. Mental Model
> "Tưởng tượng `WebClient` như một **Nhà hàng Fast Food hiện đại**. Bạn gọi món, nhận một cái 'thẻ rung' (Promise/Mono), và quay về bàn làm việc tiếp. Khi món ăn xong, thẻ rung lên và bạn ra lấy. Bạn không cần đứng đợi ở quầy (block thread) và nhân viên có thể phục vụ hàng chục người khác trong lúc món của bạn đang được nấu."

## 4. Where it fits (architecture)
`Reactive Service → [WebClient] → [Project Reactor] → Event Loop → External API`

## 5. When to use
- Trong các ứng dụng xây dựng trên **Spring WebFlux**.
- Khi cần gọi nhiều API cùng lúc (Parallel calls) để tổng hợp dữ liệu.
- Khi làm việc với dữ liệu cực lớn hoặc streaming dữ liệu (Server-Sent Events).
- Trong các hệ thống High-concurrency yêu cầu khả năng mở rộng (scalability) cao.

## 6. When NOT to use
- Trong các ứng dụng Spring MVC đơn giản không có yêu cầu cao về concurrency (việc học WebFlux có thể gây overhead về thời gian).
- Khi đội ngũ chưa quen với lập trình Reactive (Functional style, Mono/Flux) vì rất khó debug.

## 7. Trade-offs
| Pros | Cons |
|------|------|
| Hiệu năng cực cao với ít tài nguyên. | Khó học (Learning curve) và khó debug (Stack trace phức tạp). |
| API dạng Fluent, dễ đọc và linh hoạt. | Dễ gây lỗi "Block" nếu không hiểu rõ cơ chế Reactive. |
| Hỗ trợ Streaming và Backpressure. | Cần thêm dependency `spring-boot-starter-webflux`. |

## 8. Alternatives (with comparison)
| Option | So sánh |
|--------|---------|
| **RestTemplate** | Đã cũ, chỉ chạy đồng bộ, tốn thread. |
| **RestClient** | API giống WebClient nhưng chạy đồng bộ (cho ứng dụng MVC). |
| **Java 11 HttpClient** | Build-in sẵn trong JDK, hỗ trợ Async nhưng không mạnh mẽ bằng WebClient. |

## 9. How (minimal example)

### Khởi tạo WebClient
```java
WebClient webClient = WebClient.builder()
    .baseUrl("https://api.github.com")
    .defaultHeader(HttpHeaders.CONTENT_TYPE, MediaType.APPLICATION_JSON_VALUE)
    .build();
```

### GET Request (Single Object)
```java
Mono<User> userMono = webClient.get()
    .uri("/users/{user}", "octocat")
    .retrieve()
    .bodyToMono(User.class);
    
userMono.subscribe(user -> System.out.println(user.getName()));
```

### POST Request
```java
Mono<Post> postMono = webClient.post()
    .uri("/posts")
    .bodyValue(new Post("Title", "Content"))
    .retrieve()
    .bodyToMono(Post.class);
```

## 10. Production concerns
### Scaling
- Cấu hình **Connection Pool** thông qua `HttpClient` của thư viện Netty (mặc định của WebClient).
### Failure
- Sử dụng `.retry()` hoặc `.retryWhen()` để tự động gọi lại khi lỗi network.
- Thiết lập `ResponseTimeout` và `ReadTimeout` ở mức Netty connector.
### Monitoring
- Sử dụng `.log()` để quan sát luồng dữ liệu hoặc tích hợp Micrometer để đo metrics.

## 11. Common mistakes / anti-patterns
- ❌ **Mistake**: Gọi `.block()` bên trong một luồng Reactive (Event Loop). Điều này sẽ làm treo toàn bộ ứng dụng.
  ✅ **Fix**: Luôn trả về `Mono` hoặc `Flux` lên tầng Controller.
- ❌ **Mistake**: Không xử lý lỗi (OnError).
  ✅ **Fix**: Sử dụng `.onErrorResume()` hoặc `.onErrorReturn()` để cung cấp dữ liệu dự phòng.

## 12. Sample project (with constraint)
**Tên project**: "Crypto Price Aggregator"
**Constraint**: Phải gọi 5 sàn giao dịch khác nhau cùng lúc. Chỉ lấy giá từ 3 sàn phản hồi nhanh nhất, 2 sàn chậm hơn sẽ bị hủy request ngay lập tức.
**Output**: Giá trung bình của đồng BTC từ các nguồn nhanh nhất.

## 13. Interview
### Core Q&A
1. **Q**: Tại sao WebClient lại nhanh hơn RestTemplate trong môi trường nhiều request?
   **A**: WebClient sử dụng **Non-blocking IO**, nó không giữ thread để đợi response từ server. Thread được giải phóng để làm việc khác, dẫn đến throughput (lưu lượng) cao hơn nhiều với cùng một lượng tài nguyên phần cứng.
2. **Q**: Sự khác biệt giữa `retrieve()` và `exchangeToMono()`?
   **A**: `retrieve()` là cách đơn giản nhất để lấy body trực tiếp. `exchangeToMono()` (trước đây là `exchange()`) cho phép truy cập sâu vào status code và headers nhưng bạn phải tự quản lý việc giải phóng bộ nhớ (memory leaks).
### Scenario
> "Tình huống: Bạn đang dùng WebClient trong một project Spring MVC (Servlet). Bạn có nên gọi `.block()` không?"
**A**: Trong project Spring MVC, việc gọi `.block()` là chấp nhận được vì thread đó vốn đã là thread dành riêng cho request đó. Tuy nhiên, nếu có thể, hãy cân nhắc dùng `RestClient` (Spring 6.1) để có API đẹp mà vẫn giữ mô hình đồng bộ thuần túy.
