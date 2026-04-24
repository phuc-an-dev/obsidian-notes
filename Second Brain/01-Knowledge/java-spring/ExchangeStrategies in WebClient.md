---
created: 2026-04-21
tags:
  - "#type/concept"
  - "#status/draft"
  - "#lang/spring"
  - "#topic/http"
  - "#topic/performance"
related:
  - "[[WebClient in Spring Boot]]"
---

# ExchangeStrategies in WebClient

## 1. What

`ExchangeStrategies` là một interface trong Spring WebFlux dùng để cấu hình cách `WebClient` xử lý việc mã hóa (encode) và giải mã (decode) dữ liệu HTTP (HTTP message readers và writers). Nó là trung tâm điều khiển cho các Codecs, quyết định cách chuyển đổi từ ByteBuf/DataBuffer sang Java Objects và ngược lại.

---

## 2. Why

Mặc định, `WebClient` (và cả WebFlux server) giới hạn dung lượng buffer lưu trữ dữ liệu trong bộ nhớ là **256KB** (`maxInMemorySize`). Điều này nhằm bảo vệ hệ thống khỏi các cuộc tấn công DoS hoặc tình trạng cạn kiệt bộ nhớ (OutOfMemory) khi xử lý các payload cực lớn.

Tuy nhiên, trong thực tế:
- Nhiều API trả về JSON có kích thước lớn hơn 256KB (danh sách sản phẩm, logs, data báo cáo).
- Cần tùy chỉnh Jackson `ObjectMapper` (thêm module, định dạng ngày tháng riêng).
- Cần hỗ trợ các format không mặc định như Protobuf, CSV, hoặc XML tùy biến.

---

## 3. Mental Model

Hãy tưởng tượng `ExchangeStrategies` là **Phòng Kiểm soát Đóng gói & Dịch thuật** của một trạm vận chuyển (`WebClient`):
- Khi hàng (data) về, trạm cần nhân viên (Decoders) để mở hộp và dịch nội dung.
- Mặc định, trạm chỉ có những chiếc bàn nhỏ (256KB). Nếu kiện hàng to hơn cái bàn, nhân viên sẽ từ chối xử lý (DataBufferLimitException).
- `ExchangeStrategies` cho phép bạn yêu cầu những chiếc bàn to hơn hoặc thuê những nhân viên biết dịch các ngôn ngữ đặc biệt (Custom JSON/XML/Protobuf).

---

## 4. Where it fits

`ExchangeStrategies` được tiêm vào `WebClient` thông qua Builder:

```
[WebClient.Builder]
       |
       v
[ExchangeStrategies] <--- [Custom Codecs / Max Memory Size]
       |
       v
[ExchangeFunction]
       |
       v
[Netty / HTTP Client]
```

---

## 5. When to use

- Khi gặp lỗi `DataBufferLimitException: Exceeded limit on max bytes to buffer : 262144`.
- Khi cần tích hợp Jackson module đặc biệt (như `JavaTimeModule` với cấu hình riêng).
- Khi làm việc với các hệ thống cũ (legacy) trả về XML phức tạp hoặc các format custom.
- Khi cần cấu hình logging cho form data hoặc dữ liệu đa phần (multipart data).

---

## 6. When NOT to use

- Nếu API của bạn chỉ trả về các object nhỏ và dùng cấu hình Jackson chuẩn của Spring Boot.
- Khi bạn có thể xử lý dữ liệu theo dạng stream (`Flux<DataBuffer>` hoặc `Flux<T>`) thay vì gom toàn bộ vào một `Mono<T>` lớn. Lưu ý: limit 256KB áp dụng cho mỗi item trong Flux nếu nó cần decode, nhưng thường gặp nhất là khi dùng `bodyToMono`.

---

## 7. Trade-offs

| Pros | Cons |
|------|------|
| Kiểm soát chi tiết cách xử lý dữ liệu | Tăng nguy cơ OutOfMemory nếu set limit quá lớn |
| Dễ dàng mở rộng cho các format mới | Cấu hình sai có thể làm hỏng cơ chế serialize mặc định của Spring |
| Centralized configuration cho toàn bộ WebClient | Phức tạp hơn so với việc chỉ dùng default |

---

## 8. Alternatives

| Option | So sánh |
|--------|---------|
| `WebClient.builder().codecs(...)` | Thực tế đây là shortcut để cấu hình `ExchangeStrategies`. Dễ dùng hơn cho các nhu cầu đơn giản. |
| Global WebFlux Config | Cấu hình cho toàn bộ application (cả server lẫn client), thiếu tính cô lập cho từng client cụ thể. |

---

## 9. How

### Cấu hình tăng giới hạn bộ nhớ (Phổ biến nhất)
```java
ExchangeStrategies strategies = ExchangeStrategies.builder()
    .codecs(clientCodecConfigurer -> {
        clientCodecConfigurer
            .defaultCodecs()
            .maxInMemorySize(16 * 1024 * 1024); // Tăng lên 16MB
    })
    .build();

WebClient webClient = WebClient.builder()
    .exchangeStrategies(strategies)
    .build();
```

### Cấu hình Custom Jackson ObjectMapper
```java
ObjectMapper customMapper = new ObjectMapper()
    .registerModule(new JavaTimeModule())
    .configure(DeserializationFeature.FAIL_ON_UNKNOWN_PROPERTIES, false);

ExchangeStrategies strategies = ExchangeStrategies.builder()
    .codecs(configurer -> {
        configurer.defaultCodecs().jackson2JsonEncoder(
            new Jackson2JsonEncoder(customMapper, MediaType.APPLICATION_JSON));
        configurer.defaultCodecs().jackson2JsonDecoder(
            new Jackson2JsonDecoder(customMapper, MediaType.APPLICATION_JSON));
    })
    .build();
```

---

## 10. Production concerns
### Memory Management
Việc tăng `maxInMemorySize` lên quá cao (ví dụ -1 cho không giới hạn) là một rủi ro lớn. Nếu downstream service trả về payload vài GB, ứng dụng của bạn sẽ chết ngay lập tức vì OOM. Hãy tính toán dựa trên RAM khả dụng và concurrency tối đa.

### Reusability
Nên định nghĩa `ExchangeStrategies` như một Bean hoặc dùng chung một instance Builder để tiết kiệm resource và đảm bảo tính nhất quán.

---

## 11. Common mistakes

- Mistake: Set `maxInMemorySize(-1)` (unlimited) trong production.
  Fix: Luôn đặt một con số cụ thể (ví dụ 10MB, 50MB) dựa trên yêu cầu thực tế của business.

- Mistake: Quên đăng ký `ExchangeStrategies` vào Builder khi tạo WebClient mới.
  Fix: Sử dụng một `WebClient.Builder` chung đã được cấu hình sẵn thông qua Dependency Injection.

---

## 12. Sample project
**Tên: Huge JSON Processor**
Tạo một service gọi đến một API giả lập trả về mảng 100,000 bản ghi JSON (~15MB).
- Yêu cầu: Không được dùng `.block()`.
- Ràng buộc: Cấu hình `ExchangeStrategies` chỉ đủ 20MB. Nếu quá 20MB phải log error và trả về empty list.

---

## 13. Interview
### Core Q&A
1. Q: Lỗi `DataBufferLimitException` là gì và tại sao nó tồn tại?
   A: Đây là lỗi khi payload response vượt quá 256KB mặc định của WebClient. Nó tồn tại như một cơ chế bảo vệ (backpressure/safety) để tránh việc một request tiêu tốn quá nhiều RAM, gây crash hệ thống.

2. Q: Làm thế nào để thay đổi giới hạn bộ nhớ cho một WebClient duy nhất?
   A: Sử dụng `ExchangeStrategies.builder().codecs(c -> c.defaultCodecs().maxInMemorySize(newSize)).build()` và truyền vào `webClientBuilder.exchangeStrategies()`.

3. Q: `ExchangeStrategies` có ảnh hưởng đến hiệu năng không?
   A: Có, nếu cấu hình quá nhiều codecs không cần thiết hoặc dùng ObjectMapper quá nặng. Tuy nhiên, tác động chính thường nằm ở việc quản lý bộ nhớ buffer.

### Scenario
**Tình huống:** API của đối tác bỗng nhiên thay đổi format ngày tháng từ ISO sang một format lạ, khiến WebClient của bạn không parse được. Bạn xử lý thế nào mà không làm ảnh hưởng đến các WebClient gọi API khác?
**Giải quyết:** Tạo một `ExchangeStrategies` riêng biệt với một `ObjectMapper` được cấu hình `SimpleDateFormat` cụ thể, sau đó build một `WebClient` instance dành riêng cho đối tác đó.

---

## 14. References
- Official Docs: https://docs.spring.io/spring-framework/docs/current/javadoc-api/org/springframework/web/reactive/function/client/ExchangeStrategies.html
- Spring Blog: https://spring.io/blog/2020/03/23/spring-framework-5-2-5-available-now (mentioning maxInMemorySize)

---

## 15. Real-world Code
- Spring Cloud Gateway dùng `ExchangeStrategies` để cấu hình giới hạn payload khi proxy request.
- Các library như `spring-cloud-openfeign` (khi dùng với WebClient capability) cũng cho phép cấu hình này.

---

## 16. Community
- Stack Overflow: "How to set maxInMemorySize in WebClient" (Một trong những câu hỏi phổ biến nhất về WebFlux).
- Blog: Baeldung - Spring WebClient Config.
