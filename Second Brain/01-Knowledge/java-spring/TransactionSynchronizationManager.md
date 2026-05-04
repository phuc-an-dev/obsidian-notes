---
created: 2026-05-04
tags:
  - "#type/concept"
  - "#status/draft"
  - "#lang/spring"
  - "#topic/database"
related:
  - "[[TransactionSynchronization]]"
  - "[[TransactionTemplate]]"
---

## 1. What
`TransactionSynchronizationManager` là một lớp tiện ích (central delegate) trong Spring Framework quản lý các tài nguyên và các đồng bộ hóa (synchronizations) liên quan đến giao dịch theo từng luồng xử lý (**Thread-local**).

## 2. Why
Trong một ứng dụng đa luồng, mỗi request thường chạy trên một thread riêng. Để đảm bảo rằng tất cả các thao tác (như Hibernate Session, JDBC Connection, transaction callbacks) trong cùng một request đều dùng chung một transaction, Spring cần một "nơi lưu trữ" bí mật gắn liền với thread đó. `TransactionSynchronizationManager` chính là cái kho lưu trữ đó.

## 3. Mental Model
Hãy tưởng tượng `TransactionSynchronizationManager` như một cái **"Tủ đồ cá nhân (Locker)"** của mỗi nhân viên (Thread) trong một công ty (Ứng dụng). Khi một nhân viên bắt đầu ca làm việc (Transaction), anh ta cất các công cụ (Connection, Session) vào tủ đồ riêng của mình. Bất cứ khi nào anh ta cần dùng, anh ta chỉ cần mở tủ đồ của *chính mình* ra lấy, không sợ bị nhầm với đồ của nhân viên khác.

## 4. Where it fits
Nó là trái tim điều phối tài nguyên ở tầng thấp (low-level) của Spring Transaction:
`PlatformTransactionManager <-> TransactionSynchronizationManager <-> Resource Holders (JDBC/JPA)`

## 5. When to use
- Khi bạn cần kiểm tra xem có một giao dịch nào đang hoạt động hay không (`isActualTransactionActive()`).
- Khi cần lấy tên của giao dịch hiện tại (`getCurrentTransactionName()`) để ghi log/audit.
- Khi cần đăng ký các callback `TransactionSynchronization` thủ công.
- Khi viết các thư viện tích hợp sâu với Spring Transaction.

## 6. When NOT to use
- Trong hầu hết các logic nghiệp vụ thông thường (hãy dùng `@Transactional`).
- Khi bạn không thực sự hiểu về cơ chế Thread-local, vì việc can thiệp sai có thể dẫn đến rò rỉ tài nguyên (resource leak).

## 7. Trade-offs
| Pros | Cons |
|------|------|
| Quản lý tài nguyên cực kỳ chính xác theo từng thread. | Gắn chặt với mô hình "One thread per request". |
| Cho phép can thiệp sâu vào vòng đời transaction. | Khó debug nếu xảy ra lỗi liên quan đến thread-local. |
| Là nền tảng cho sự linh hoạt của Spring Transaction. | Không hoạt động tự nhiên trong các môi trường Reactive (Project Reactor) do thread bị chuyển đổi liên tục. |

## 8. Alternatives
| Option | So sánh |
|--------|---------|
| `TransactionContext` | (Trong Spring WebFlux/Reactive) Dùng thay thế cho Thread-local để quản lý context. |
| Thủ công truyền `Connection` | Rất khó quản lý và dễ gây lỗi. |

## 9. How
```java
import org.springframework.transaction.support.TransactionSynchronizationManager;

public class TransactionHelper {
    public void checkStatus() {
        // 1. Kiểm tra transaction có đang chạy không
        boolean isActive = TransactionSynchronizationManager.isActualTransactionActive();
        
        // 2. Lấy tên transaction (nếu có đặt tên)
        String txName = TransactionSynchronizationManager.getCurrentTransactionName();
        
        // 3. Kiểm tra xem có đang ở chế độ Read-only không
        boolean isReadOnly = TransactionSynchronizationManager.isCurrentTransactionReadOnly();
        
        System.out.println("TX Status: " + isActive + ", Name: " + txName);
    }
}
```

## 10. Production concerns
### Scaling
Vì sử dụng `ThreadLocal`, dữ liệu sẽ tồn tại cho đến khi thread đó kết thúc hoặc được dọn dẹp. Spring tự động dọn dẹp sau khi transaction kết thúc, nên rủi ro rò rỉ bộ nhớ là rất thấp nếu dùng đúng cách.

### Failure
Nếu bạn sử dụng các thư viện chạy đa luồng bên trong một phương thức `@Transactional` (như `CompletableFuture`), các thread con sẽ không thể truy cập vào dữ liệu trong `TransactionSynchronizationManager` của thread cha.

### Monitoring
Theo dõi số lượng tài nguyên (Resources) và đồng bộ hóa (Synchronizations) đang được giữ bởi manager để phát hiện các bất thường trong hệ thống.

## 11. Common mistakes
- **Mistake**: Giả định rằng transaction context sẽ tự động "bay" sang các thread mới tạo.
  **Fix**: Phải sử dụng các giải pháp truyền context thủ công hoặc dùng `DelegatingSecurityContextExecutor` (nếu liên quan đến Security).

- **Mistake**: Tự ý gọi `clear()` hoặc `initSynchronization()` mà không hiểu rõ hậu quả.
  **Fix**: Hãy để Spring tự quản lý các phương thức này.

## 12. Sample project
Tạo một `TransactionLogger` định kỳ kiểm tra trạng thái của `TransactionSynchronizationManager` và log lại tên của các transaction đang chạy lâu hơn 5 giây để cảnh báo performance.

## 13. Interview
### Core Q&A
1. **Q**: `TransactionSynchronizationManager` lưu trữ dữ liệu dựa trên cơ chế nào?
   **A**: Nó dựa trên **ThreadLocal** để đảm bảo dữ liệu của mỗi thread là độc lập.
2. **Q**: Tại sao nó không hoạt động tốt trong ứng dụng Reactive?
   **A**: Vì trong lập trình Reactive, một request có thể được xử lý bởi nhiều thread khác nhau, khiến cho ThreadLocal không còn duy trì được tính nhất quán của context.
3. **Q**: Làm sao để biết một phương thức đang chạy trong một Transaction "thực sự" (không phải chỉ là đồng bộ hóa)?
   **A**: Sử dụng `TransactionSynchronizationManager.isActualTransactionActive()`.

### Scenario
**Tình huống**: Bạn đang debug một lỗi mà dường như `@Transactional` bị bỏ qua. Bạn dùng `TransactionSynchronizationManager` như thế nào để kiểm tra?
**Trả lời**: Tôi sẽ đặt một breakpoint hoặc thêm log gọi `isSynchronizationActive()`. Nếu nó trả về `false`, chứng tỏ cơ chế Proxy của Spring chưa được kích hoạt cho phương thức đó.

## 14. References
- Official Docs: [TransactionSynchronizationManager API](https://docs.spring.io/spring-framework/docs/current/javadoc-api/org/springframework/transaction/support/TransactionSynchronizationManager.html)

## 15. Real-world Code
- Lớp `DataSourceUtils` của Spring dùng manager này để lấy đúng `Connection` hiện tại của transaction.

## 16. Community
- Stack Overflow: Tag [spring-transactions]
- Blog: "Under the hood of Spring Transactions".
