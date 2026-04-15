---
created: 2026-04-15
tags:
  - "#type/concept"
  - "#status/draft"
  - "#lang/java"
related: "[[Spring Framework]]"
---

## 1. What
**EventListener** trong Spring Boot là một cơ chế cho phép các component giao tiếp với nhau một cách lỏng lẻo (loose coupling) thông qua các sự kiện (events). Khi một hành động xảy ra (Event Publisher), các thành phần quan tâm (Event Listeners) sẽ nhận được thông báo và xử lý mà không cần biết ai đã tạo ra sự kiện đó.

## 2. Why (problem it solves)
Trước khi có EventListener, nếu `OrderService` cần gửi email và cập nhật kho sau khi đặt hàng thành công, nó phải inject trực tiếp `EmailService` và `InventoryService`. Điều này dẫn đến:
- **Tight Coupling**: `OrderService` phụ thuộc quá nhiều vào các service khác.
- **Violation of SRP**: `OrderService` phải quản lý logic không thuộc về nghiệp vụ chính của nó.
- **Khó mở rộng**: Mỗi khi thêm một hành động mới (ví dụ: tặng voucher), ta phải sửa code của `OrderService`.

## 3. Mental Model
> "Tưởng tượng như một hệ thống Loa Thông báo (Public Address System) trong trường học. Văn phòng hiệu trưởng (Publisher) phát đi thông báo 'Giờ giải lao bắt đầu' (Event). Học sinh, giáo viên, và nhân viên căng tin (Listeners) đều nghe thấy và tự thực hiện hành động riêng của mình (chơi đùa, nghỉ ngơi, chuẩn bị đồ ăn) mà văn phòng không cần phải gọi điện riêng cho từng người."

## 4. Where it fits (architecture)
`Caller → Service (Publisher) → [ApplicationEventPublisher] → [ApplicationEvent] → [EventListener] → Logic xử lý`

## 5. When to use
- Khi muốn tách biệt các logic phụ (side-effects) ra khỏi luồng xử lý chính.
- Khi một hành động cần kích hoạt nhiều hành động khác không liên quan trực tiếp đến nghiệp vụ chính (Logging, Analytics, Notification).
- Thực hiện các tác vụ bất đồng bộ (Async) để cải thiện performance cho main thread.

## 6. When NOT to use
- Khi các hành động phụ thuộc chặt chẽ vào nhau và cần nằm trong cùng một Transaction nguyên tử (Atomic). Mặc dù có `@TransactionalEventListener`, nhưng nếu quá lạm dụng sẽ làm luồng dữ liệu trở nên khó theo dõi (hidden logic).
- Trong các hệ thống quá phức tạp, việc dùng quá nhiều event khiến việc debug trở thành "ác mộng" vì không biết code đang nhảy đi đâu.

## 7. Trade-offs
| Pros | Cons |
|------|------|
| **Loose Coupling**: Các service không cần biết nhau. | **Traceability**: Khó lần theo luồng thực thi (flow) của code. |
| **Open/Closed Principle**: Dễ dàng thêm listener mới mà không sửa code cũ. | **Complexity**: Tăng độ phức tạp của hệ thống nếu dùng quá đà. |
| **Async Support**: Dễ dàng chuyển sang xử lý nền (background). | **Ordering**: Mặc định các listener không có thứ tự (cần dùng `@Order`). |

## 8. Alternatives (with comparison)
| Option | Khi nào chọn |
|--------|-------------|
| **Direct Method Call** | Khi logic đơn giản, cần sự chắc chắn và dễ debug. |
| **AOP (Aspect Oriented Programming)** | Khi muốn can thiệp vào các logic mang tính hệ thống (Logging, Security) mà không sửa code service. |
| **Message Broker (Kafka/RabbitMQ)** | Khi cần giao tiếp giữa các **Microservices** khác nhau thay vì trong cùng một JVM. |

## 9. How (minimal example)
```java
// 1. Define Event
public class OrderCreatedEvent {
    private String orderId;
    public OrderCreatedEvent(String orderId) { this.orderId = orderId; }
    public String getOrderId() { return orderId; }
}

// 2. Publish Event
@Service
public class OrderService {
    @Autowired
    private ApplicationEventPublisher eventPublisher;

    public void createOrder(String orderId) {
        System.out.println("Creating order: " + orderId);
        eventPublisher.publishEvent(new OrderCreatedEvent(orderId));
    }
}

// 3. Listen Event
@Component
public class EmailListener {
    @EventListener
    @Async // Optional: Chạy bất đồng bộ
    public void handleOrderCreated(OrderCreatedEvent event) {
        System.out.println("Sending email for order: " + event.getOrderId());
    }
}
```

## 10. Production concerns
### Scaling
- Nếu dùng `@Async`, cần cấu hình `TaskExecutor` để quản lý Thread Pool, tránh việc tạo quá nhiều thread dẫn đến cạn kiệt tài nguyên.
### Failure
- Mặc định, nếu một Listener ném exception, nó có thể ảnh hưởng đến Publisher (nếu chạy đồng bộ). 
- Cần có cơ chế Retry hoặc Dead Letter Queue nếu xử lý quan trọng.
### Monitoring
- Cần log lại thời điểm publish và nhận event để theo dõi latency giữa các bước.

## 11. Common mistakes / anti-patterns
- ❌ **Mistake**: Quên `@EnableAsync` khi sử dụng `@Async` trên EventListener.
  ✅ **Fix**: Thêm `@EnableAsync` vào Configuration class.
- ❌ **Mistake**: Thực hiện các tác vụ nặng (Blocking IO) trong Listener đồng bộ làm treo main thread.
  ✅ **Fix**: Chuyển sang `@Async` hoặc sử dụng hàng đợi bên ngoài.

## 12. Sample project (with constraint)
**Tên project**: "Smart Home Automation"
**Constraint**: Hệ thống phải hỗ trợ thêm mới thiết bị (Đèn, Điều hòa, Rèm) mà không được sửa code của bộ điều khiển trung tâm (Central Controller).
**Output**: Khi Central Controller phát event "Chế độ ngủ", tất cả các thiết bị tự động tắt/đóng tương ứng.

## 13. Interview
### Core Q&A
1. **Q**: Sự khác biệt giữa `@EventListener` và `@TransactionalEventListener` là gì?
   **A**: `@EventListener` chạy ngay lập tức khi event được phát. `@TransactionalEventListener` cho phép chọn thời điểm chạy dựa trên trạng thái transaction (ví dụ: `AFTER_COMMIT` - chỉ chạy sau khi DB đã lưu thành công).
2. **Q**: Làm sao để đảm bảo thứ tự thực thi của các Listener?
   **A**: Sử dụng annotation `@Order(n)` trên phương thức listener. Số nhỏ hơn sẽ chạy trước.
### Scenario
> "Tình huống: Hệ thống e-commerce của bạn thỉnh thoảng bị mất email xác nhận đơn hàng khi server quá tải. Bạn giải quyết thế nào bằng EventListener?"
**A**: Em sẽ chuyển EmailListener sang xử lý bất đồng bộ bằng `@Async` kết hợp với một Thread Pool Executor có giới hạn. Để tránh mất mát, em sẽ lưu Event vào một bảng `outbox` trong DB trước khi gửi, hoặc chuyển sang dùng Message Broker như RabbitMQ nếu hệ thống cần độ tin cậy cao.
