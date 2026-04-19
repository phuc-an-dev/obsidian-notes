---
created: 2026-04-19
tags:
  - "#type/concept"
  - "#status/draft"
  - "#lang/system-design"
  - "#topic/http"
related: "[[]]"
---

## 1. What
RESTful API là một giao diện lập trình ứng dụng (API) tuân thủ các ràng buộc của kiến trúc REST (Representational State Transfer). Nó sử dụng các phương thức HTTP tiêu chuẩn để thực hiện các thao tác trên tài nguyên (resources) được định danh bằng URI.

## 2. Why
Trước khi REST phổ biến, các giao thức như SOAP rất phức tạp, nặng nề và yêu cầu định dạng XML nghiêm ngặt. REST ra đời để cung cấp một cách tiếp cận đơn giản hơn, tận dụng tối đa các đặc tính sẵn có của giao thức HTTP, giúp hệ thống dễ dàng mở rộng và tương tác giữa các nền tảng khác nhau.

## 3. Mental Model
Hãy tưởng tượng RESTful API giống như một hệ thống Menu trong nhà hàng. Thực đơn (URI) liệt kê các món ăn (Resources). Bạn thực hiện các hành động (HTTP Methods) như: Đặt món (POST), Xem món (GET), Đổi món (PUT/PATCH), hoặc Hủy món (DELETE). Phục vụ (Server) sẽ trả lại món ăn hoặc thông báo trạng thái mà không cần phải nhớ bạn là ai giữa các lần gọi nếu bạn mang theo hóa đơn (Stateless).

## 4. Where it fits
REST đóng vai trò là lớp giao tiếp giữa Client và Server trong mô hình phân tầng.
Client (Mobile/Web) -> HTTP Request (Method + URI + Header + Body) -> REST API Layer -> Service/Database -> HTTP Response (Status Code + Body).

## 5. When to use
- Xây dựng Web Services cho các ứng dụng công khai.
- Các hệ thống yêu cầu khả năng mở rộng cao (Scalability).
- Khi cần sự tách biệt rõ ràng giữa Client và Server.
- Các ứng dụng di động cần giao tiếp với server trung tâm.

## 6. When NOT to use
- Các ứng dụng thời gian thực (Real-time) cường độ cao như Chat hoặc Game (nên dùng WebSocket).
- Khi cần truyền tải dữ liệu nhị phân cực lớn một cách liên tục.
- Hệ thống Microservices nội bộ yêu cầu hiệu năng cực cao và độ trễ thấp (nên cân nhắc gRPC).
- Các truy vấn dữ liệu phức tạp, lồng nhau nhiều lớp (nên cân nhắc GraphQL).

## 7. Trade-offs
| Pros | Cons |
|------|------|
| Đơn giản, dễ học và triển khai dựa trên HTTP. | Có thể dẫn đến tình trạng Over-fetching hoặc Under-fetching dữ liệu. |
| Stateless giúp dễ dàng mở rộng hệ thống (Scale out). | Khó quản lý các quan hệ dữ liệu phức tạp chỉ qua URI. |
| Khả năng Cache tốt giúp tăng hiệu năng. | Thiếu chuẩn hóa nghiêm ngặt về cấu trúc Response so với SOAP. |

## 8. Alternatives
| Option | So sánh |
|--------|---------|
| GraphQL | Cho phép Client chỉ định chính xác dữ liệu cần thiết, tránh Over-fetching. |
| gRPC | Hiệu năng cao hơn, sử dụng Protocol Buffers và HTTP/2, phù hợp giao tiếp nội bộ. |
| SOAP | Bảo mật chặt chẽ hơn, hỗ trợ ACID transaction, nhưng rất nặng nề. |
| Webhook | Server chủ động đẩy dữ liệu cho Client khi có sự kiện, thay vì Client phải Pull. |

## 9. How
Ví dụ tối thiểu cho một RESTful API quản lý sách bằng Java Spring Boot:
```java
@RestController
@RequestMapping("/api/v1/books")
public class BookController {

    @GetMapping("/{id}")
    public ResponseEntity<Book> getBook(@PathVariable Long id) {
        return ResponseEntity.ok(new Book(id, "Clean Code"));
    }

    @PostMapping
    public ResponseEntity<Book> createBook(@RequestBody Book book) {
        return new ResponseEntity<>(book, HttpStatus.CREATED);
    }
}
```

## 10. Production concerns
### Scaling
Nhờ tính chất Stateless, các REST API có thể dễ dàng chạy sau một Load Balancer. Bất kỳ instance nào cũng có thể xử lý request mà không cần quan tâm đến session của các request trước đó.

### Failure
Sử dụng các Standard HTTP Status Codes (4xx cho lỗi Client, 5xx cho lỗi Server) để Client có cơ chế xử lý lỗi phù hợp (ví dụ: Retry khi gặp 503).

### Monitoring
Theo dõi các chỉ số: Request per second (RPS), Latency (p95, p99), Error Rate, và Payload size.

## 11. Common mistakes
- Mistake: Sử dụng động từ trong URI (ví dụ: /getAllUsers, /createUser).
  Fix: Sử dụng danh từ và HTTP Methods (GET /users, POST /users).

- Mistake: Trả về HTTP 200 OK cho mọi trường hợp kể cả khi có lỗi bên trong logic.
  Fix: Sử dụng đúng Status Code (ví dụ: 404 cho Not Found, 400 cho Bad Request).

## 12. Sample project
Xây dựng một API quản lý Task (Todo List) với ràng buộc: Không sử dụng bất kỳ thư viện bên thứ ba nào ngoài framework chính, và phải hỗ trợ đầy đủ HATEOAS (Hypermedia as the Engine of Application State) để điều hướng giữa các tài nguyên.

## 13. Interview
### Core Q&A
1. Q: Idempotency trong REST là gì? Những phương thức nào là Idempotent?
   A: Idempotency là tính chất mà một thao tác thực hiện nhiều lần cũng cho cùng một kết quả như thực hiện một lần. GET, PUT, DELETE, HEAD, OPTIONS là idempotent. POST thì không.
2. Q: Sự khác biệt giữa PUT và PATCH là gì?
   A: PUT thay thế toàn bộ tài nguyên hiện tại bằng payload mới. PATCH thực hiện cập nhật một phần tài nguyên.

### Scenario
Khách hàng phàn nàn rằng ứng dụng mobile load dữ liệu rất chậm vì phải gọi quá nhiều API liên tiếp mới hiển thị đủ một màn hình Dashboard. Bạn sẽ giải quyết thế nào trong kiến trúc REST?
Gợi ý: Áp dụng Pattern "Backend For Frontend" (BFF) để tổng hợp dữ liệu hoặc tối ưu hóa resource representation.

## 14. References
- Official Docs: https://www.ics.uci.edu/~fielding/pubs/dissertation/rest_arch_style.htm (Luận văn của Roy Fielding)
- GitHub Repo: https://github.com/microsoft/api-guidelines
- Spec / RFC: RFC 7231 (HTTP/1.1 Semantics and Content)
- Changelog: N/A

## 15. Real-world Code
- GitHub: https://github.com/public-apis/public-apis (Danh sách các REST API công khai lớn nhất)
- GitHub: https://github.com/strapi/strapi (Headless CMS dựa trên REST/GraphQL)

## 16. Community
- Reddit: r/rest
- Stack Overflow: Tag [rest]
- Blog: Martin Fowler's guide to Richardson Maturity Model
- Talk: "REST: I don't think it means what you think it means" by Stefan Tilkov
