---
created: 2026-05-07
tags:
  - "#type/concept"
  - "#status/draft"
  - "#lang/spring"
  - "#topic/performance"
related:
  - "[[mvn clean package in Spring Boot.md]]"
  - "[[mvnw.md]]"
  - "[[tdd.md]]"
  - "[[GitHub Actions MySQL Service Container.md]]"
---

## 1. What
`@SpringBootTest` là một annotation quan trọng trong Spring Boot Test framework, được sử dụng để thực hiện kiểm thử tích hợp (Integration Testing). Khi sử dụng annotation này, Spring Boot sẽ khởi tạo toàn bộ (hoặc một phần lớn) `ApplicationContext` của ứng dụng, giống như khi ứng dụng chạy trong môi trường thực tế.

## 2. Why
Unit test thường chỉ kiểm tra các phương thức riêng lẻ bằng cách dùng Mock. Tuy nhiên, tích hợp kiểm thử (`@SpringBootTest`) là cần thiết để:
- Đảm bảo các Bean trong `ApplicationContext` được nối dây (wiring) đúng cách.
- Kiểm tra sự tương tác giữa ứng dụng và các thành phần ngoại vi (Database, Message Broker, API bên thứ ba).
- Xác định xem cấu hình ứng dụng (như YAML, Environment Variables) có hoạt động chính xác hay không.

## 3. Mental Model
Hãy tưởng tượng ứng dụng của bạn là một **"Chiếc xe ô tô"**:
- **Unit Test** giống như việc bạn tháo rời từng bộ phận (piston, bugi) để kiểm tra xem chúng có hỏng không.
- **`@SpringBootTest`** giống như việc bạn **ngồi vào cabin và nổ máy xe**. Bạn kiểm tra xem khi đạp ga thì bánh xe có quay không, đèn có sáng không. Bạn kiểm tra toàn bộ hệ thống xe khi chúng được lắp ráp hoàn chỉnh.

## 4. Where it fits
Test Code -> **`@SpringBootTest`** -> Spring Boot ApplicationContext (Beans, Config, Profiles) -> Database/External Services -> Test Result.

## 5. When to use
- Khi cần kiểm tra các flow nghiệp vụ đi xuyên suốt nhiều tầng (Controller -> Service -> Repository).
- Khi kiểm tra tính đúng đắn của các câu truy vấn cơ sở dữ liệu thực tế (thường kết hợp với Testcontainers).
- Khi muốn kiểm tra cấu hình của ứng dụng trong môi trường gần giống production nhất.

## 6. When NOT to use
- **Unit Test thuần túy**: Nếu bạn chỉ muốn test logic trong 1 method, hãy dùng Mockito để nhanh hơn.
- **Slice Testing**: Nếu chỉ muốn test tầng Web, hãy dùng `@WebMvcTest`. Nếu chỉ muốn test tầng JPA, dùng `@DataJpaTest`. `@SpringBootTest` rất nặng và chậm vì nó load mọi thứ.

## 7. Trade-offs
| Pros | Cons |
|------|------|
| Độ tin cậy cực cao vì test trên môi trường thật. | Tốc độ chạy rất chậm vì phải khởi tạo toàn bộ Context. |
| Phát hiện được các lỗi cấu hình context (`BeanCreationException`). | Tốn nhiều tài nguyên RAM/CPU (dễ gây lỗi OutOfMemory nếu bộ test quá lớn). |
| Dễ dàng viết test cho các luồng end-to-end. | Cần phải dọn dẹp dữ liệu (database cleanup) sau mỗi lần chạy. |

## 8. Alternatives
| Option | So sánh |
|--------|---------|
| `@WebMvcTest` | Chỉ load tầng web, nhanh hơn nhiều cho controller test. |
| `@DataJpaTest` | Chỉ load tầng repository và database, tối ưu cho DB test. |
| `@RestClientTest` | Dùng để test các client gọi API bên ngoài (như RestTemplate, WebClient). |

## 9. How
Ví dụ cơ bản về `@SpringBootTest`:

```java
@SpringBootTest(webEnvironment = SpringBootTest.WebEnvironment.RANDOM_PORT)
class MyIntegrationTest {

    @LocalServerPort
    private int port;

    @Autowired
    private TestRestTemplate restTemplate;

    @Test
    void contextLoads() {
        // Kiểm tra xem ứng dụng có khởi động thành công không
    }

    @Test
    void shouldReturnDefaultMessage() {
        String url = "http://localhost:" + port + "/api/hello";
        String body = restTemplate.getForObject(url, String.class);
        assertThat(body).contains("Hello World");
    }
}
```

## 10. Production concerns
### Dirty Context
Nếu một test case làm thay đổi trạng thái của Bean (ví dụ dùng `@MockBean`), Spring sẽ phải reload context cho test case sau, làm chậm bộ test. Sử dụng `@DirtiesContext` chỉ khi cực kỳ cần thiết.

### Database management
Sử dụng các cấu hình database riêng cho test (ví dụ H2 hoặc Testcontainers) để tránh làm hỏng dữ liệu production.

## 11. Common mistakes
- Mistake: Lạm dụng `@SpringBootTest` cho mọi test case.
  Fix: Luôn ưu tiên Unit Test hoặc Slice Test (`@WebMvcTest`, `@DataJpaTest`) trước.

- Mistake: Không quản lý Port, dẫn đến xung đột khi chạy song song.
  Fix: Luôn sử dụng `webEnvironment = WebEnvironment.RANDOM_PORT`.

## 12. Sample project
Xây dựng bộ Integration Test cho một ứng dụng Order Management:
1. Dùng `@SpringBootTest` để khởi động app.
2. Dùng Testcontainers để chạy PostgreSQL thật.
3. Thực hiện gửi request POST tạo Order qua `TestRestTemplate`.
4. Kiểm tra xem bản ghi có xuất hiện trong DB không.

## 13. Interview
### Core Q&A
1. Q: `@SpringBootTest` khác gì với `@ContextConfiguration`?
   A: `@SpringBootTest` tự động tìm kiếm `@SpringBootApplication` và cấu hình mọi thứ mặc định của Spring Boot. `@ContextConfiguration` yêu cầu bạn tự chỉ định các class cấu hình thủ công.

2. Q: Các giá trị của `webEnvironment` là gì?
   A: 
   - `MOCK` (Mặc định): Không chạy server thật, dùng Mock servlet.
   - `RANDOM_PORT`: Chạy server thật trên port ngẫu nhiên.
   - `DEFINED_PORT`: Chạy trên port quy định (ví dụ 8080).
   - `NONE`: Không load web layer.

### Scenario
"Bộ test của bạn chạy mất 20 phút và 80% thời gian là chờ Context khởi động. Bạn tối ưu thế nào?"
-> Trả lời: 
1. Chuyển bớt sang Slice Testing (`@WebMvcTest`, v.v.).
2. Hạn chế dùng `@MockBean` bên trong `@SpringBootTest` để tận dụng cơ chế Context Caching của Spring.
3. Chia bộ test thành các nhóm nhỏ hơn.

## 14. References
- Spring Boot Docs: [Testing the Web Layer](https://spring.io/guides/gs/testing-web/)
- Baeldung: [Spring Boot Testing Guide](https://www.baeldung.com/spring-boot-testing)

## 15. Real-world Code
Nghiên cứu các test class được sinh ra mặc định bởi Spring Initializr (file `*ApplicationTests.java`).

## 16. Community
- Stack Overflow: Tag [spring-boot-test].
- Reddit: r/java, r/springboot.
