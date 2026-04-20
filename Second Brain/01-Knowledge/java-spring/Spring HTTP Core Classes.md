---
created: 2026-04-17
tags:
  - "#type/concept"
  - "#status/draft"
  - "#lang/spring"
  - "#topic/http"
related:
  - "[[HTTP Clients in Spring]]"
  - "[[RestTemplate in Spring Boot]]"
---

# Spring HTTP Core Classes

## 1. What

Package `org.springframework.http` cung cấp các class trừu tượng cốt lõi để biểu diễn các thành phần của giao thức HTTP trong Spring Framework. Các class này — `HttpHeaders`, `HttpStatus`, `ResponseEntity`, `RequestEntity`, `MediaType`, và `ProblemDetail` — là building blocks chung cho cả server-side (controller trả response) lẫn client-side (RestTemplate, WebClient, RestClient gửi request). Chúng tạo ra một lớp abstraction thống nhất, type-safe trên đầu HTTP thuần.

## 2. Why

Không có các class này, lập trình viên phải:

- Xử lý HTTP headers bằng `Map<String, String>` thủ công — dễ sai chính tả, không hỗ trợ multi-value cho cùng một header key (ví dụ: `Set-Cookie` có thể có nhiều giá trị).
- Dùng số nguyên cho status code (200, 404, 500) — code khó đọc, khó bảo trì.
- Không có cách chuẩn để đóng gói cả body + status + headers vào một object khi trả response từ controller.
- Phụ thuộc vào Servlet API (`HttpServletResponse`) để set status và headers — buộc chặt mã nguồn vào servlet container, khó viết unit test.
- Trước Spring 6, không có chuẩn hóa cho error response format — mỗi API trả lời lỗi theo cấu trúc khác nhau.

## 3. Mental Model

Hãy nghĩ đến hệ thống chuyển phát hàng hóa nội địa.

- **HttpHeaders** là tờ khai hàng — ghi chi tiết về kiện hàng (loại hàng, trọng lượng, người gửi, mã bưu chính, loại vận chuyển ưu tiên). Nó là metadata, không phải hàng hóa chính.
- **MediaType** là loại hàng — đồ uống, thực phẩm, hàng điện tử, tài liệu (APPLICATION_JSON = tài liệu JSON, APPLICATION_PDF = tài liệu PDF). Người nhận biết cách mở gói dựa vào loại này.
- **HttpStatus** là trạng thái giao hàng — "Đã giao thành công" (200 OK), "Không tìm thấy địa chỉ" (404 Not Found), "Kho hàng hết hàng" (503 Service Unavailable).
- **HttpEntity** là cả kiện hàng = tờ khai + hàng hóa bên trong. Nó là class cha chung.
- **RequestEntity** là kiện hàng đang gửi đi — có thêm thông tin về phương thức vận chuyển (GET, POST) và đích đến (URI).
- **ResponseEntity** là kiện hàng đã giao đến — có thêm trạng thái giao hàng (HttpStatus) và toàn bộ nội dung (body + headers).
- **ProblemDetail** là biên bản sự cố giao hàng — định dạng chuẩn (RFC 7807) để mô tả lỗi: kiểu lỗi, tiêu đề, chi tiết, URL tham khảo.

## 4. Where It Fits

```
Outgoing Request (client side):
  RestTemplate / WebClient / RestClient
      |
      v
  RequestEntity<T>  (body + method + URI + headers)
      |
      v
  [HTTP Wire]
      |
      v
  ResponseEntity<T> (body + status + headers) <-- parsed response

Incoming Request (server side):
  HTTP Request
      |
      v
  @RequestBody T  +  @RequestHeader HttpHeaders  +  HttpMethod
      |
      v
  Controller logic
      |
      v
  ResponseEntity<T>  (or ProblemDetail for errors)
      |
      v
  HTTP Response
```

## 5. When to Use

- `ResponseEntity<T>`: Mỗi khi bạn cần kiểm soát HTTP status code hoặc response headers từ controller method — không chỉ trả POJO.
- `HttpHeaders`: Khi cần set, đọc, hoặc copy HTTP headers một cách an toàn — khi gọi external API hoặc khi customize response.
- `MediaType`: Khi cần chỉ định `Content-Type` hoặc `Accept` header, hoặc khi cần kiểm tra MIME type trong logic xử lý.
- `HttpStatus`: Khi cần so sánh hoặc kiểm tra nhóm status code (is2xxSuccessful, is4xxClientError) một cách có ý nghĩa.
- `RequestEntity<T>`: Khi dùng `RestTemplate.exchange()` và cần full control trên request (method + URI + headers + body) trong một object.
- `ProblemDetail`: Khi xây dựng REST API mới với Spring 6+ và muốn tuân theo RFC 7807 cho error response.

## 6. When NOT to Use

- Không dùng `ResponseEntity` cho mọi method nếu không cần — nếu controller method luôn trả 200 OK và chỉ có body, trả POJO trực tiếp đơn giản hơn và code sạch hơn.
- Không tạo `HttpHeaders` mới cho mỗi request khi dùng WebClient hoặc RestClient — các client này có builder API để thiết lập headers một cách elegant hơn.
- Không dùng `HttpEntity` trong Service layer — các class này thuộc về tầng presentation/infrastructure. Service nên làm việc với domain object, không phải HTTP wrapper.
- Không sử dụng `HttpStatus` để hardcode logic nghiệp vụ ("nếu status == 404 thì làm X") — logic nghiệp vụ nên dựa trên exception hoặc result object, không phải HTTP status code.

## 7. Trade-offs

| Pros | Cons |
|---|---|
| Type-safe: tránh sai chính tả với enum và hằng số | Verbose hơn khi so với sử dụng Map/int thuần túy |
| Tương thích với toàn bộ Spring ecosystem | Cần hiểu rõ sự khác biệt giữa HttpEntity, RequestEntity, ResponseEntity |
| Hỗ trợ multi-value headers (MultiValueMap) | Một số builder method khó nhớ (phải tra JavaDoc) |
| ProblemDetail giúp chuẩn hóa error response (Spring 6+) | ProblemDetail cần `@EnableWebMvc` hoặc Spring MVC config để auto-resolve |
| Dễ test với MockHttpServletRequest trong unit test | ResponseEntity với generics có thể gây unchecked cast warning |

## 8. Alternatives

| Option | Phù hợp khi | Hạn chế |
|---|---|---|
| Servlet API HttpServletResponse | Xử lý low-level, streaming response | Phụ thuộc container, khó test |
| Native Java HttpRequest (java.net.http) | Không dùng Spring, Java 11+ | Không tích hợp Spring ecosystem |
| Apache HttpClient | Cần cấu hình connection pool phức tạp | Thêm dependency, không dùng Spring type |
| OkHttp | Android hoặc lightweight Java project | Không native trong Spring |
| JAX-RS (Jersey) | Đã dùng JAX-RS standard trước đây | Xung đột khi dùng cùng Spring MVC |

## 9. How

### HttpHeaders — xây dựng và đọc headers

```java
// Tạo headers với builder style
HttpHeaders headers = new HttpHeaders();
headers.setContentType(MediaType.APPLICATION_JSON);
headers.setAccept(List.of(MediaType.APPLICATION_JSON));
headers.setBearerAuth("eyJhbGci...");
headers.set("X-Request-ID", UUID.randomUUID().toString());
headers.set("X-Custom-Header", "custom-value");

// Đọc header
String contentType = headers.getFirst(HttpHeaders.CONTENT_TYPE);
List<String> acceptValues = headers.get(HttpHeaders.ACCEPT);

// Copy headers từ request (proxy usecase)
HttpHeaders copied = new HttpHeaders();
copied.addAll(headers);
copied.set("X-Proxied-By", "my-gateway");
```

### HttpStatus và HttpStatusCode — kiểm tra status

```java
HttpStatus status = HttpStatus.CREATED;                  // 201
HttpStatus notFound = HttpStatus.NOT_FOUND;              // 404

// Kiểm tra nhóm
status.is2xxSuccessful();        // true
notFound.is4xxClientError();     // true
notFound.is5xxServerError();     // false

// Từ Spring 6: HttpStatusCode (super interface)
HttpStatusCode code = HttpStatusCode.valueOf(429);
code.is4xxClientError();         // true
// Custom status code không nằm trong enum vẫn được hỗ trợ
```

### ResponseEntity<T> — trả response từ controller

```java
// Minimal: chỉ trả status
return ResponseEntity.ok(userDto);                   // 200 + body
return ResponseEntity.created(location).build();     // 201 + Location header, no body
return ResponseEntity.noContent().build();           // 204, no body
return ResponseEntity.notFound().build();            // 404, no body

// Đầy đủ: status + headers + body
return ResponseEntity
    .status(HttpStatus.ACCEPTED)
    .header("X-Process-ID", processId)
    .contentType(MediaType.APPLICATION_JSON)
    .body(responseDto);

// Trong controller
@GetMapping("/users/{id}")
public ResponseEntity<UserDto> getUser(@PathVariable Long id) {
    return userService.findById(id)
        .map(user -> ResponseEntity.ok(user))
        .orElse(ResponseEntity.notFound().build());
}
```

### RequestEntity<T> — đóng gói request cho RestTemplate

```java
// Xây dựng RequestEntity để gọi external API
URI uri = URI.create("https://api.example.com/orders");

RequestEntity<OrderRequest> requestEntity = RequestEntity
    .post(uri)
    .contentType(MediaType.APPLICATION_JSON)
    .accept(MediaType.APPLICATION_JSON)
    .header("Authorization", "Bearer " + token)
    .body(new OrderRequest("item-1", 2));

ResponseEntity<OrderResponse> response = restTemplate.exchange(
    requestEntity,
    OrderResponse.class
);

if (response.getStatusCode().is2xxSuccessful()) {
    OrderResponse order = response.getBody();
}
```

### MediaType — làm việc với content types

```java
// Các hằng số thường dùng
MediaType.APPLICATION_JSON              // application/json
MediaType.APPLICATION_JSON_VALUE        // String version
MediaType.APPLICATION_FORM_URLENCODED   // application/x-www-form-urlencoded
MediaType.MULTIPART_FORM_DATA           // multipart/form-data
MediaType.APPLICATION_PDF               // application/pdf
MediaType.IMAGE_PNG                     // image/png
MediaType.TEXT_HTML                     // text/html
MediaType.TEXT_PLAIN                    // text/plain

// Parse từ string
MediaType type = MediaType.parseMediaType("application/vnd.api+json");

// Kiểm tra compatibility
MediaType requested = MediaType.APPLICATION_JSON;
MediaType produced = MediaType.parseMediaType("application/json;charset=UTF-8");
requested.isCompatibleWith(produced);   // true
```

### ProblemDetail (RFC 7807) — error response chuẩn hóa

```java
// Spring 6+ tích hợp sẵn, dùng trong @ExceptionHandler
@ExceptionHandler(EntityNotFoundException.class)
public ResponseEntity<ProblemDetail> handleNotFound(EntityNotFoundException ex) {
    ProblemDetail problem = ProblemDetail
        .forStatusAndDetail(HttpStatus.NOT_FOUND, ex.getMessage());
    problem.setTitle("Resource Not Found");
    problem.setType(URI.create("https://api.example.com/errors/not-found"));
    problem.setProperty("resourceId", ex.getResourceId());   // custom extension
    return ResponseEntity.status(HttpStatus.NOT_FOUND).body(problem);
}

// Kết quả JSON theo RFC 7807:
// {
//   "type": "https://api.example.com/errors/not-found",
//   "title": "Resource Not Found",
//   "status": 404,
//   "detail": "User with id 42 not found",
//   "instance": "/users/42",
//   "resourceId": 42
// }

// Hoặc dùng ResponseEntityExceptionHandler (Spring MVC):
@ControllerAdvice
public class GlobalExceptionHandler extends ResponseEntityExceptionHandler {
    // Spring tự động convert MethodArgumentNotValidException, etc. sang ProblemDetail
}
```

## 10. Production Concerns

### Scaling

- `HttpHeaders` là thread-safe cho đọc, nhưng không share instance để modify ở nhiều thread đồng thời — tạo instance mới cho mỗi request.
- `ResponseEntity` là immutable sau khi build — an toàn trong môi trường concurrent.
- `MediaType.parseMediaType()` có chi phí parse — nên dùng các constant có sẵn thay vì parse động.

### Failure

- Khi nhận `ResponseEntity` từ RestTemplate, always check `getStatusCode().is2xxSuccessful()` trước khi gọi `getBody()` — body có thể null với 204 No Content hoặc redirect.
- Handle `HttpClientErrorException` (4xx) và `HttpServerErrorException` (5xx) riêng biệt vì chúng có ý nghĩa khác nhau cho logic retry.
- `ProblemDetail` từ Spring 6 giúp client code xử lý lỗi dễ hơn vì format nhất quán — đầu tư cấu hình `@EnableWebMvc` để bật tự động.

### Monitoring

- Log `HttpStatus.value()` và `HttpHeaders.get("X-Request-ID")` trong mỗi response để tracing.
- Với Spring Boot Actuator, expose `/actuator/httptrace` (hoặc `/actuator/httpexchanges` từ Spring Boot 3) để xem lịch sử request.
- Dùng Micrometer metrics để đo response time phân loại theo status code nhóm.

## 11. Common Mistakes

- Mistake: Trả POJO trực tiếp từ controller method thay vì dùng `ResponseEntity`, khiến status code luôn là 200 ngay cả khi resource được tạo mới (nên là 201 Created).
  Fix: Dùng `ResponseEntity.created(location).body(createdResource)` cho POST endpoint tạo resource. Nếu location URI không cần thiết, ít nhất dùng `ResponseEntity.status(HttpStatus.CREATED).body(...)`.

- Mistake: Dùng `Map<String, String>` để truyền headers vào RestTemplate.exchange() thay vì `HttpHeaders` — mất khả năng hỗ trợ multi-value header và không có type safety.
  Fix: Luôn dùng `HttpHeaders` class. Nó implements `MultiValueMap<String, String>` nên tự động xử lý đúng dấu nhiều giá trị cho cùng một key (Set-Cookie, Accept, v.v.).

- Mistake: Inject `HttpServletRequest` hoặc `HttpServletResponse` vào Service layer để đọc/set headers — ràng buộc Service với HTTP transport.
  Fix: Đọc headers ở Controller, extract thông tin cần thiết, truyền vào Service qua parameter method bình thường. Service không nên biết gì về HTTP.

- Mistake: So sánh HttpStatus bằng `==` thay vì `.equals()` hoặc dùng `.is2xxSuccessful()` — có thể fail với custom status code trong Spring 6 (`HttpStatusCode.valueOf()`).
  Fix: Dùng `.is2xxSuccessful()`, `.isSameCodeAs()`, hoặc `.value() == 200` khi cần kiểm tra giá trị số.

## 12. Sample Project

Xây dựng một **API Gateway Proxy** với ràng buộc cứng sau:

**Constraint:** Endpoint `/proxy/{targetService}/**` phải:
1. Đọc toàn bộ headers từ incoming request, loại bỏ headers nào bắt đầu bằng `X-Internal-` (internal headers).
2. Thêm header `X-Proxied-By: api-gateway` và `X-Request-ID: <uuid>`.
3. Forward request đến target service với cùng HTTP method và body.
4. Nếu target service trả 4xx, wrap response thành `ProblemDetail` chuẩn và trả về cho client.
5. Nếu target service trả 5xx, trả `503 Service Unavailable` với `ProblemDetail` có `detail` là "Upstream service unavailable".

**Output kỳ vọng:** Client nhận được response nhất quán theo RFC 7807 cho mọi trường hợp lỗi, và tất cả outgoing request đến target service đều có `X-Request-ID` để tracing.

## 13. Interview

### Core Q&A

**Q: Sự khác biệt giữa `HttpEntity`, `RequestEntity`, và `ResponseEntity` là gì?**
A: `HttpEntity<T>` là class cha — biểu diễn cả request lẫn response, chứa headers + body. `RequestEntity<T>` extend `HttpEntity` thêm `HttpMethod` và `URI` — dùng khi gửi request qua RestTemplate.exchange(). `ResponseEntity<T>` extend `HttpEntity` thêm `HttpStatusCode` — dùng khi nhận response hoặc khi trả response từ controller.

**Q: Tại sao phải dùng `HttpHeaders` thay vì `Map<String, String>` cho HTTP headers?**
A: HTTP headers có thể có nhiều giá trị cho cùng một key (multi-value). Ví dụ `Set-Cookie` có thể xuất hiện nhiều lần. `HttpHeaders` implements `MultiValueMap<String, String>` nên xử lý đúng điều này. Ngoài ra nó cũng cung cấp các method typed như `setContentType()`, `setBearerAuth()`, `setAccept()` giúp code rõ ràng và tránh sai chính tả.

**Q: `ProblemDetail` là gì và tại sao dùng nó?**
A: `ProblemDetail` là implementation của RFC 7807 (Problem Details for HTTP APIs), được thêm vào Spring 6. Nó cung cấp một format JSON chuẩn hóa cho error response với các trường cố định: `type`, `title`, `status`, `detail`, `instance`. Ưu điểm: client code không cần handle nhiều error format khác nhau; documentation tự mô tả bởi `type` URI; có thể mở rộng với custom properties.

**Q: `HttpStatus.Series` dùng để làm gì?**
A: Để phân nhóm status code theo loại: `INFORMATIONAL` (1xx), `SUCCESSFUL` (2xx), `REDIRECTION` (3xx), `CLIENT_ERROR` (4xx), `SERVER_ERROR` (5xx). Thường dùng trong error handling: `if (status.series() == HttpStatus.Series.SERVER_ERROR)` thay vì kiểm tra từng giá trị.

**Q: MediaType.isCompatibleWith() khác MediaType.equals() thế nào?**
A: `equals()` so sánh exact match. `isCompatibleWith()` hỗ trợ wildcard: `application/*` là compatible với `application/json`. Dùng `isCompatibleWith()` khi kiểm tra Accept header của client có khớp với loại Content-Type mà server sản xuất hay không.

### Scenario

**Scenario 1:** API của bạn cần trả về một file PDF và yêu cầu trình duyệt tải xuống thay vì hiển thị trực tiếp. Cấu hình `ResponseEntity` như thế nào?

Trả lời: Set `Content-Type` là `MediaType.APPLICATION_PDF` và thêm header `Content-Disposition` với giá trị `attachment; filename="document.pdf"` trong `HttpHeaders`. Dùng `ResponseEntity<byte[]>` hoặc `ResponseEntity<Resource>`. Content-Disposition với "attachment" chỉ thị trình duyệt tải xuống; "inline" sẽ hiển thị trực tiếp.

```java
HttpHeaders headers = new HttpHeaders();
headers.setContentType(MediaType.APPLICATION_PDF);
headers.setContentDispositionFormData("attachment", "report.pdf");
return new ResponseEntity<>(pdfBytes, headers, HttpStatus.OK);
```

**Scenario 2:** Bạn đang xây dựng `@ExceptionHandler` chung cho toàn bộ application. Làm sao đảm bảo tất cả lỗi đều trả về format nhất quán với Spring 6?

Trả lời: Extend `ResponseEntityExceptionHandler` và override các method cần thiết. Với Spring 6 và `spring.mvc.problemdetails.enabled=true` trong `application.properties`, Spring tự động convert các exception chuẩn (MethodArgumentNotValidException, etc.) sang ProblemDetail. Cho custom exception, dùng `ProblemDetail.forStatusAndDetail()` trong `@ExceptionHandler`.

**Scenario 3:** Khi dùng RestTemplate, bạn nhận ResponseEntity<MyDto> nhưng body là null dù status là 200 OK. Nguyên nhân có thể là gì?

Trả lời: Một số nguyên nhân: (1) Content-Type của response không phải `application/json` nên Jackson không deserialize, (2) Response body là empty string hoặc JSON null, (3) Đang dùng `exchange()` với kiểu trả về sai (phải dùng `ParameterizedTypeReference` cho generic type như `List<MyDto>`), (4) Server trả 204 No Content nhưng status được override thành 200 bởi proxy.

## 14. References

- Spring Framework Docs - HTTP Interface: https://docs.spring.io/spring-framework/reference/web/webmvc/mvc-ann-rest-exceptions.html
- Spring Framework Docs - ResponseEntity: https://docs.spring.io/spring-framework/reference/web/webmvc/mvc-ann-return-types.html#mvc-ann-return-types-response-entity
- JavaDoc - HttpHeaders: https://docs.spring.io/spring-framework/docs/current/javadoc-api/org/springframework/http/HttpHeaders.html
- JavaDoc - ResponseEntity: https://docs.spring.io/spring-framework/docs/current/javadoc-api/org/springframework/http/ResponseEntity.html
- JavaDoc - ProblemDetail: https://docs.spring.io/spring-framework/docs/current/javadoc-api/org/springframework/http/ProblemDetail.html
- RFC 7807 - Problem Details for HTTP APIs: https://www.rfc-editor.org/rfc/rfc7807
- Spring Blog - Error Handling in Spring MVC: https://spring.io/blog/2013/11/01/exception-handling-in-spring-mvc

## 15. Real-world Code

- Spring MVC source - ResponseEntity builder: https://github.com/spring-projects/spring-framework/blob/main/spring-web/src/main/java/org/springframework/http/ResponseEntity.java
- Spring MVC source - ProblemDetail: https://github.com/spring-projects/spring-framework/blob/main/spring-web/src/main/java/org/springframework/http/ProblemDetail.java
- Spring MVC source - HttpHeaders: https://github.com/spring-projects/spring-framework/blob/main/spring-web/src/main/java/org/springframework/http/HttpHeaders.java
- Baeldung examples - ResponseEntity: https://github.com/eugenp/tutorials/tree/master/spring-web-modules/spring-mvc-basics
- Spring PetClinic REST: https://github.com/spring-petclinic/spring-petclinic-rest

## 16. Community

- Stack Overflow - "ResponseEntity vs @ResponseBody": https://stackoverflow.com/questions/22725143/responseentity-vs-responsebody
- Stack Overflow - "Spring ProblemDetail RFC 7807": https://stackoverflow.com/questions/74138153/how-to-use-problemdetail-in-spring-boot-3
- Baeldung - ResponseEntity in Spring: https://www.baeldung.com/spring-response-entity
- Baeldung - ProblemDetail in Spring 6: https://www.baeldung.com/spring-6-error-handling
- Baeldung - HttpHeaders guide: https://www.baeldung.com/spring-rest-http-headers
