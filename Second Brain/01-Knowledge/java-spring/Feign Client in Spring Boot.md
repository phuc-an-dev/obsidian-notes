---
created: 2026-04-15
tags:
  - "#type/concept"
  - "#status/draft"
  - "#lang/java"
related: "[[HTTP Clients in Spring]]"
---

## 1. What
**Feign Client** (Spring Cloud OpenFeign) là một HTTP Client dạng khai báo (declarative). Thay vì viết code để thực hiện request, bạn chỉ cần định nghĩa một **Interface** và sử dụng các annotation của Spring Web để mô tả request đó. Spring sẽ tự động tạo ra implementation thực tế lúc runtime.

## 2. Why (problem it solves)
- **Boilerplate reduction**: Loại bỏ hoàn toàn việc sử dụng `RestTemplate` hay `WebClient` lặp đi lặp lại (xây dựng URL, gán header, xử lý response).
- **Service Discovery Integration**: Tự động tích hợp với Eureka/Consul. Bạn chỉ cần gọi tên service (ví dụ: `inventory-service`) thay vì dùng IP/Domain cứng.
- **Clean Architecture**: Chuyển đổi việc gọi API từ bên ngoài trông giống hệt như đang gọi một hàm Java nội bộ trong ứng dụng.

## 3. Mental Model
> "Tưởng tượng Feign Client như một **Cuốn thực đơn (Interface)**. Bạn không cần biết đầu bếp nấu ăn thế nào hay nhà bếp ở đâu. Bạn chỉ cần chỉ vào món ăn trên thực đơn và gọi. Một **Người phục vụ (Feign Proxy)** sẽ tự đi lấy món đó và mang về cho bạn đúng định dạng bạn yêu cầu."

## 4. Where it fits (architecture)
`Service A → [Feign Proxy] → [Load Balancer] → [Service Discovery] → Service B`

## 5. When to use
- Trong kiến trúc **Microservices** khi các dịch vụ cần giao tiếp với nhau.
- Khi sử dụng hệ sinh thái **Spring Cloud** (Eureka, Gateway, Config Server).
- Khi muốn code client gọn gàng, dễ đọc và dễ bảo trì theo phong cách hướng Interface.

## 6. When NOT to use
- Trong các ứng dụng Monolith đơn giản (chỉ gọi 1-2 API bên ngoài, dùng `RestClient` sẽ nhanh hơn).
- Khi cần kiểm soát cực kỳ chi tiết các thông số kỹ thuật thấp của HTTP (Low-level socket control).
- Nếu ứng dụng của bạn hoàn toàn là Reactive (WebFlux), hãy cân nhắc vì Feign mặc định là Blocking (dù đã có bản Reactive Feign nhưng chưa phổ biến bằng WebClient).

## 7. Trade-offs
| Pros | Cons |
|------|------|
| Code cực kỳ ngắn gọn, chuyên nghiệp. | Khó debug vì logic được sinh ra tự động (Proxy). |
| Dễ dàng unit test bằng cách mock interface. | Startup time của ứng dụng lâu hơn một chút do phải tạo proxy. |
| Tích hợp sẵn Load Balancing và Circuit Breaker. | Phụ thuộc vào hệ sinh thái Spring Cloud. |

## 8. Alternatives (with comparison)
| Option | So sánh |
|--------|---------|
| **RestTemplate** | Thủ công (Imperative), tốn nhiều code hơn. |
| **WebClient** | Tốt cho Reactive/Async, nhưng code phức tạp hơn Feign. |
| **RestClient** | API hiện đại, nhưng không có tính năng Declarative tự động như Feign. |

## 9. How (minimal example)

### 1. Kích hoạt Feign
```java
@SpringBootApplication
@EnableFeignClients
public class Application { ... }
```

### 2. Định nghĩa Interface Client
```java
@FeignClient(name = "inventory-service", path = "/api/v1")
public interface InventoryClient {

    @GetMapping("/products/{id}")
    ProductDTO getProductStock(@PathVariable("id") Long id);
}
```

### 3. Sử dụng trong Service
```java
@Service
public class OrderService {
    @Autowired
    private InventoryClient inventoryClient;

    public void createOrder(Long productId) {
        ProductDTO product = inventoryClient.getProductStock(productId);
        // Xử lý logic...
    }
}
```

## 10. Production concerns
### Scaling
- Tích hợp sẵn **Spring Cloud LoadBalancer** để phân phối request đều cho các instance của service đích.
### Failure
- Sử dụng **Resilience4j** để cấu hình `Fallback` (giá trị dự phòng) hoặc `FallbackFactory` (để log được lỗi cụ thể khi service đích chết).
### Monitoring
- Cấu hình `Logger.Level.FULL` để log toàn bộ request/response body trong môi trường dev/staging.

## 11. Common mistakes / anti-patterns
- ❌ **Mistake**: Quên `@EnableFeignClients` ở class main.
  ✅ **Fix**: Luôn kiểm tra annotation này để Spring quét được các interface client.
- ❌ **Mistake**: Không cấu hình Timeout. Mặc định Feign có timeout rất ngắn, dễ gây lỗi khi service đích xử lý chậm.
  ✅ **Fix**: Cấu hình `feign.client.config.default.readTimeout` trong `application.yml`.

## 12. Sample project (with constraint)
**Tên project**: "E-commerce Inter-service Call"
**Constraint**: Phải gọi Inventory Service để check kho trước khi tạo Order. Nếu Inventory Service không phản hồi trong 500ms, phải giả định là "Hết hàng" để đảm bảo trải nghiệm người dùng không bị treo.
**Output**: Sử dụng Feign với Fallback class trả về `stock = 0`.

## 13. Interview
### Core Q&A
1. **Q**: Feign hoạt động như thế nào dưới nền (under the hood)?
   **A**: Spring Cloud sử dụng **JDK Dynamic Proxy** để tạo ra một instance thực tế từ interface. Khi một phương thức được gọi, proxy sẽ chuyển đổi các annotation thành HTTP request, gửi đi qua một HTTP Client (mặc định là JDK hoặc Apache HttpClient) và parse response về POJO.
2. **Q**: Làm sao để truyền Header (như JWT Token) qua Feign?
   **A**: Sử dụng một **`RequestInterceptor`** để tự động đính kèm header vào mọi request đi ra từ Feign.
### Scenario
> "Tình huống: Bạn có 10 Feign Clients khác nhau. Bạn muốn tất cả chúng đều dùng chung một cấu hình timeout và logging. Bạn làm thế nào?"
**A**: Em sẽ tạo một class `FeignConfiguration` (không đánh dấu `@Configuration` để tránh global scan) và khai báo nó trong `@EnableFeignClients(defaultConfiguration = MyConfig.class)`.
