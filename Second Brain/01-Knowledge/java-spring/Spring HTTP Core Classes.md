---
created: 2026-04-15
tags:
  - "#type/concept"
  - "#status/draft"
  - "#lang/java"
related: "[[Spring Framework]]"
---

## 1. What
Gói **`org.springframework.http`** cung cấp các class trừu tượng cốt lõi để đại diện cho các thành phần của giao thức HTTP trong Spring Framework. Các class quan trọng bao gồm `HttpHeaders`, `HttpEntity`, `HttpMethod`, `HttpStatus`, và `ResponseEntity`.

## 2. Why (problem it solves)
Nếu không có các class này, lập trình viên sẽ phải:
- Xử lý HTTP Headers bằng `Map<String, String>` thủ công, dễ sai sót chính tả.
- Sử dụng các số nguyên (int) cho Status Code (ví dụ: 200, 404) dẫn đến code khó đọc.
- Khó khăn trong việc đóng gói cả Body và Headers khi gửi request qua `RestTemplate` hoặc `WebClient`.
- Phụ thuộc quá chặt vào Servlet API (`HttpServletRequest/Response`), gây khó khăn khi viết Unit Test.

## 3. Mental Model
> "Tưởng tượng `HttpEntity` như một **Thùng hàng chuyển phát nhanh**. 
> - `HttpHeaders` là cái **Nhãn dán** bên ngoài (ghi địa chỉ, loại hàng, mã vận đơn).
> - `Body` là **Nội dung bên trong** thùng.
> - `HttpMethod` là **Phương thức vận chuyển** (Giao hàng, Thu hồi, Kiểm tra).
> - `HttpStatus` là **Trạng thái giao hàng** (Thành công, Thất bại, Không tìm thấy người nhận)."

## 4. Where it fits (architecture)
`Client (RestTemplate/WebClient) ↔ [HttpEntity/HttpHeaders] ↔ Network ↔ [ResponseEntity] ↔ Controller`

## 5. When to use
- Khi xây dựng các REST API sử dụng `@RestController`.
- Khi gọi các dịch vụ bên ngoài (External APIs) bằng `RestTemplate` hoặc `WebClient`.
- Khi cần tùy chỉnh HTTP Headers (như `Authorization`, `Content-Type`) hoặc Status Code trả về cho client.

## 6. When NOT to use
- Khi làm việc với các giao thức không phải HTTP (gRPC, RSocket, WebSockets - mặc dù WebSocket có bắt đầu bằng HTTP Handshake).
- Trong các ứng dụng chỉ xử lý logic nội bộ (POJO Service) không liên quan đến giao tiếp mạng.

## 7. Trade-offs
| Pros | Cons |
|------|------|
| **Type Safety**: Tránh lỗi chính tả với các enum và hằng số có sẵn. | **Verbosity**: Đôi khi code trông dài dòng hơn so với dùng Map đơn thuần. |
| **Interoperability**: Hoạt động mượt mà với mọi module của Spring (Security, Web, Cloud). | **Learning Curve**: Cần nhớ cách phối hợp giữa `HttpEntity`, `ResponseEntity` và `RequestEntity`. |

## 8. Alternatives (with comparison)
| Option | Khi nào chọn |
|--------|-------------|
| **Native Java `HttpRequest`** | Khi không muốn phụ thuộc vào Spring (Java 11+). |
| **Apache HttpClient** | Khi cần cấu hình low-level cực kỳ chi tiết cho connection pool. |
| **OkHttp** | Phổ biến trong Android hoặc các project Java lightweight. |

## 9. How (minimal example)
```java
// 1. Tạo Headers
HttpHeaders headers = new HttpHeaders();
headers.setContentType(MediaType.APPLICATION_JSON);
headers.setBearerAuth("my-secret-token");

// 2. Đóng gói Body và Headers vào HttpEntity
String body = "{\"name\": \"Gemini\"}";
HttpEntity<String> request = new HttpEntity<>(body, headers);

// 3. Gửi Request bằng RestTemplate
RestTemplate restTemplate = new RestTemplate();
ResponseEntity<String> response = restTemplate.exchange(
    "https://api.example.com/data", 
    HttpMethod.POST, 
    request, 
    String.class
);

// 4. Kiểm tra Status Code
if (response.getStatusCode() == HttpStatus.OK) {
    System.out.println("Success: " + response.getBody());
}
```

## 10. Production concerns
### Scaling
- `HttpHeaders` là thread-safe cho việc đọc nhưng không nên share instance để modify ở nhiều nơi đồng thời.
### Failure
- Cần handle `HttpClientErrorException` (4xx) và `HttpServerErrorException` (5xx) khi nhận `ResponseEntity`.
### Monitoring
- Nên log `HttpStatus` và `headers.get("X-Request-ID")` để tracing request giữa các service.

## 11. Common mistakes / anti-patterns
- ❌ **Mistake**: Dùng `Map<String, String>` để truyền headers vào `RestTemplate`.
  ✅ **Fix**: Sử dụng class `HttpHeaders` vì nó hỗ trợ nhiều giá trị cho một key (MultiValueMap).
- ❌ **Mistake**: Trả về POJO trực tiếp từ Controller mà không có status code phù hợp (mặc định luôn là 200).
  ✅ **Fix**: Wrap POJO trong `ResponseEntity` để control cả Status Code và Headers.

## 12. Sample project (with constraint)
**Tên project**: "API Gateway Proxy"
**Constraint**: Tạo một Endpoint nhận request từ client, copy toàn bộ headers từ client request, thêm header `X-Proxy-By: Spring`, và forward tới một target server.
**Output**: Target server nhận được đầy đủ context từ client ban đầu cộng thêm header định danh proxy.

## 13. Interview
### Core Q&A
1. **Q**: Sự khác biệt giữa `HttpEntity` và `ResponseEntity` là gì?
   **A**: `HttpEntity` là class cha, đại diện cho cả Request hoặc Response (gồm headers + body). `ResponseEntity` là subclass của `HttpEntity` nhưng bổ sung thêm `HttpStatus`, chuyên dùng để trả về từ Controller.
2. **Q**: `HttpStatus.Series` là gì?
   **A**: Là cách phân nhóm status code (1xx: INFORMATIONAL, 2xx: SUCCESSFUL, 3xx: REDIRECTION, 4xx: CLIENT_ERROR, 5xx: SERVER_ERROR).
### Scenario
> "Tình huống: API của bạn cần trả về một file PDF và yêu cầu trình duyệt phải tải xuống thay vì hiển thị trực tiếp. Bạn dùng HttpHeaders như thế nào?"
**A**: Em sẽ sử dụng `ResponseEntity` và cấu hình `HttpHeaders`. Cụ thể là đặt `Content-Type` là `application/pdf` và quan trọng nhất là header `Content-Disposition` với giá trị `attachment; filename="document.pdf"`.
