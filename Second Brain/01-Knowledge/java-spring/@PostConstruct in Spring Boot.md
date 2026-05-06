---
created: 2026-05-06
tags:
  - "#type/concept"
  - "#status/draft"
  - "#lang/java"
  - "#topic/lifecycle"
related:
  - "[[ConfigurationProperties in Spring Boot.md]]"
---

## 1. What
`@PostConstruct` là một annotation thuộc thư viện Jakarta Annotations (trước đây là Java EE). Trong Spring Boot, nó được sử dụng để đánh dấu một method cần được thực thi ngay lập tức sau khi Bean đã được khởi tạo thành công và tất cả các phụ thuộc (dependencies) đã được inject vào.

## 2. Why
Trong quá trình khởi tạo một Bean, Constructor của Class được gọi đầu tiên. Tuy nhiên, tại thời điểm Constructor chạy, các field được đánh dấu `@Autowired` có thể vẫn đang là `null`. `@PostConstruct` ra đời để giải quyết các nhu cầu:
- **Khởi tạo dữ liệu**: Thực hiện các logic cần đến các phụ thuộc đã được inject (ví dụ: load dữ liệu từ DB vào cache).
- **Kiểm tra cấu hình**: Đảm bảo các tham số từ `@Value` hoặc `@ConfigurationProperties` đã được nạp đúng và không hợp lệ thì báo lỗi ngay.
- **Mở kết nối**: Bắt đầu thiết lập các kết nối tới server bên ngoài ngay khi Bean sẵn sàng.

## 3. Mental Model
Hãy tưởng tượng một Bean giống như một **"Chiếc máy pha cà phê mới mua"**:
- **Constructor**: Là lúc máy được lắp ráp xong các bộ phận cơ bản.
- **Dependency Injection**: Là lúc nhân viên đến cắm điện, nối ống nước và đổ hạt cà phê vào máy.
- **@PostConstruct**: Là nút **"Tự kiểm tra và làm nóng"** (Self-test). Sau khi đã đủ điện, nước, hạt, máy sẽ tự chạy bước này để đảm bảo mọi thứ sẵn sàng trước khi bạn bấm nút pha ly cà phê đầu tiên.

## 4. Where it fits
Vị trí trong Bean Lifecycle:
`Instantiation (Constructor) -> Dependency Injection (@Autowired) -> @PostConstruct -> Bean is Ready -> @PreDestroy`

## 5. When to use
- Khi cần thực hiện logic khởi tạo mà logic đó phụ thuộc vào các `@Autowired` beans khác.
- Khi cần thực hiện các thao tác "Warm-up" cho ứng dụng (như điền dữ liệu vào Redis cache).
- Khi muốn in ra log xác nhận rằng một Component quan trọng đã được khởi tạo thành công với các thông số cấu hình cụ thể.

## 6. When NOT to use
- Khi logic khởi tạo quá nặng và có thể làm chậm đáng kể thời gian startup của ứng dụng (trong trường hợp này nên dùng `ApplicationRunner` hoặc chạy bất đồng bộ).
- Khi method cần ném ra các Checked Exception phức tạp mà Spring không thể handle tốt.
- Không dùng cho các static methods.

## 7. Trade-offs
| Pros | Cons |
|------|------|
| Đảm bảo các phụ thuộc luôn khả dụng khi logic chạy. | Nếu logic trong method này lỗi, toàn bộ ứng dụng sẽ không thể khởi động. |
| Code sạch sẽ, tách biệt logic khởi tạo khỏi Constructor. | Khó viết Unit Test cho logic này nếu không khởi tạo context của Spring. |
| Là chuẩn Java (Jakarta), không phụ thuộc chặt chẽ vào API của riêng Spring. | Chỉ được chạy duy nhất một lần trong vòng đời của Bean. |

## 8. Alternatives
| Option | So sánh |
|--------|---------|
| `InitializingBean` interface | Yêu cầu implement method `afterPropertiesSet()`, phụ thuộc vào Spring API. |
| `init-method` trong `@Bean` | Khai báo trong cấu hình, phù hợp khi dùng thư viện bên thứ ba mà bạn không thể sửa code. |
| `CommandLineRunner` | Chạy sau khi toàn bộ Application Context đã load xong, không phải từng Bean lẻ. |

## 9. How
Ví dụ sử dụng `@PostConstruct`:

```java
@Component
public class CacheWarmer {
    
    @Autowired
    private DataRepository repository;
    
    @Value("${app.cache.enabled}")
    private boolean isCacheEnabled;

    @PostConstruct
    public void init() {
        if (isCacheEnabled) {
            System.out.println("Starting cache warm-up...");
            // repository đã được inject thành công nên không bị NullPointerException
            List<String> data = repository.findAllNames();
            // Logic điền vào cache...
            System.out.println("Cache warmed with " + data.size() + " items.");
        }
    }
}
```

## 10. Production concerns
### Startup Timeout
Trong môi trường Cloud (như AWS ECS/Kubernetes), nếu `@PostConstruct` chạy quá lâu, Health Check của hệ thống sẽ fail và container sẽ bị kill liên tục. Luôn giữ logic ở đây nhẹ nhàng nhất có thể.

### Exceptions
Bất kỳ RuntimeException nào ném ra từ method này sẽ ngăn chặn ứng dụng khởi động thành công. Hãy log lỗi chi tiết để DevOps có thể xử lý nhanh.

## 11. Common mistakes
- Mistake: Gọi các method có đánh dấu `@Transactional` bên trong `@PostConstruct`. Spring Proxy chưa hoàn thiện tại thời điểm này nên `@Transactional` sẽ không có tác dụng.
  Fix: Sử dụng `afterSingletonsInstantiated()` hoặc gọi qua một service khác sau khi startup.

- Mistake: Quên rằng `@PostConstruct` chỉ chạy cho các Bean do Spring quản lý.

## 12. Sample project
Tạo một `SecurityConfigChecker` sử dụng `@PostConstruct` để quét toàn bộ file `.pem` trong thư mục cấu hình. Nếu không tìm thấy file nào, ứng dụng sẽ throw `IllegalStateException` để dừng startup ngay lập tức, tránh việc chạy ở trạng thái thiếu bảo mật.

## 13. Interview
### Core Q&A
1. Q: Tại sao không nên thực hiện logic khởi tạo phức tạp trong Constructor?
   A: Vì lúc đó các phụ thuộc chưa được inject và Object chưa thực sự ở trạng thái sẵn sàng theo quản lý của Spring.

2. Q: `@PostConstruct` được thực thi mấy lần?
   A: Chỉ duy nhất 1 lần ngay sau khi Bean được khởi tạo và nạp đủ phụ thuộc.

### Scenario
"Bạn có 2 Beans, A và B. Bean A có `@PostConstruct` gọi đến Bean B. Làm sao đảm bảo Bean B đã sẵn sàng?"
-> Trả lời: Spring tự động quản lý cây phụ thuộc. Nếu A cần B, Spring sẽ khởi tạo B hoàn chỉnh trước khi gọi `@PostConstruct` của A. Nếu có vòng lặp phụ thuộc (Circular Dependency), Spring sẽ báo lỗi ngay lúc khởi động.

## 14. References
- Oracle Documentation: [Common Annotations for the Java Platform](https://docs.oracle.com/javaee/7/api/javax/annotation/PostConstruct.html)
- Spring Guide: [The Bean Lifecycle](https://docs.spring.io/spring-framework/reference/core/beans/factory-nature.html#beans-factory-lifecycle-combined-effects)

## 15. Real-world Code
Xem mã nguồn của các `AutoConfiguration` trong Spring Boot, chúng sử dụng `@PostConstruct` rất nhiều để in ra các biểu ngữ khởi động hoặc kiểm tra điều kiện tiên quyết.

## 16. Community
- Baeldung: [Spring PostConstruct and PreDestroy](https://www.baeldung.com/spring-postconstruct-predestroy)
- Stack Overflow: Tag [postconstruct].
