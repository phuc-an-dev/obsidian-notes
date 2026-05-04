---
created: 2026-05-04
tags:
  - "#type/concept"
  - "#status/draft"
  - "#lang/spring"
  - "#topic/database"
related:
  - "[[Propagation in Spring Transactions]]"
  - "[[TransactionSynchronizationManager]]"
---

## 1. What
`TransactionTemplate` là một lớp trong Spring Framework hỗ trợ quản lý giao dịch theo kiểu lập trình (**Programmatic Transaction Management**), thay vì sử dụng annotation `@Transactional` (Declarative).

## 2. Why
Mặc dù `@Transactional` rất tiện lợi, nhưng nó có những hạn chế như: chỉ hoạt động qua Proxy (không gọi được nội bộ class), phạm vi áp dụng là toàn bộ phương thức (không thể chia nhỏ transaction bên trong một method). `TransactionTemplate` cho phép bạn kiểm soát chính xác vị trí bắt đầu, kết thúc và các quy tắc rollback của một giao dịch bằng mã nguồn.

## 3. Mental Model
Hãy tưởng tượng `@Transactional` như một cái **"Thực đơn chọn sẵn (Set Menu)"**, bạn chỉ việc gọi món và nhà hàng lo mọi thứ. Còn `TransactionTemplate` giống như **"Nấu ăn tại bàn (DIY Cooking)"**. Bạn tự quyết định khi nào bật bếp, khi nào cho gia vị và khi nào tắt bếp. Nó đòi hỏi nhiều công sức hơn nhưng bạn có toàn quyền kiểm soát quy trình.

## 4. Where it fits
Nó bọc ngoài logic nghiệp vụ của bạn và tương tác với `PlatformTransactionManager`:
`Service Logic -> TransactionTemplate.execute() -> PlatformTransactionManager -> DB`

## 5. When to use
- Khi cần kiểm soát giao dịch ở mức độ chi tiết hơn một phương thức (ví dụ: chỉ transaction cho 3 dòng code ở giữa).
- Khi gặp vấn đề với Self-invocation (gọi phương thức trong cùng một class) mà không muốn tách class.
- Khi muốn thực hiện logic xử lý kết quả (return value) hoặc exception một cách tùy biến ngay trong khối transaction.

## 6. When NOT to use
- Trong hầu hết các trường hợp đơn giản (hãy ưu tiên `@Transactional` vì nó sạch và dễ bảo trì).
- Khi bạn muốn tách biệt hoàn toàn logic nghiệp vụ và logic hạ tầng (Infrastructure).

## 7. Trade-offs
| Pros | Cons |
|------|------|
| Kiểm soát chính xác phạm vi (scope) của transaction. | Làm code nghiệp vụ bị trộn lẫn với code hạ tầng. |
| Giải quyết triệt để lỗi gọi nội bộ (self-invocation). | Rườm rà hơn, khó đọc hơn so với annotation. |
| Dễ dàng rollback thủ công qua `TransactionStatus`. | Phải quản lý việc inject `PlatformTransactionManager`. |

## 8. Alternatives
| Option | So sánh |
|--------|---------|
| `@Transactional` | Khai báo, sạch sẽ, là tiêu chuẩn của Spring. |
| `PlatformTransactionManager` | (Cấp thấp nhất) Dùng `getTransaction`, `commit`, `rollback` thủ công. Rất khó dùng. |

## 9. How
```java
import org.springframework.transaction.support.TransactionTemplate;

@Service
public class PaymentService {
    private final TransactionTemplate transactionTemplate;

    public PaymentService(PlatformTransactionManager transactionManager) {
        this.transactionTemplate = new TransactionTemplate(transactionManager);
    }

    public void processPayment() {
        // Logic nằm ngoài transaction (ví dụ: gọi API bên thứ ba)
        callExternalApi();

        transactionTemplate.execute(status -> {
            try {
                // Logic bên TRONG transaction
                updateAccountBalance();
                saveTransactionHistory();
                return true;
            } catch (Exception e) {
                status.setRollbackOnly(); // Rollback thủ công
                return false;
            }
        });

        // Logic tiếp theo nằm ngoài transaction
        sendNotification();
    }
}
```

## 10. Production concerns
### Scaling
`TransactionTemplate` giúp giảm thời gian giữ connection bằng cách thu hẹp phạm vi transaction, từ đó giúp ứng dụng scale tốt hơn trong môi trường nhiều request.

### Failure
Hãy cẩn thận với việc nuốt ngoại lệ (swallow exception) bên trong callback. Nếu không ném ra hoặc không gọi `status.setRollbackOnly()`, Spring có thể vẫn commit dữ liệu sai.

### Monitoring
Dễ dàng thêm log trước và sau câu lệnh `execute()` để đo lường chính xác thời gian thực thi của một transaction cụ thể.

## 11. Common mistakes
- **Mistake**: Khởi tạo `new TransactionTemplate()` mà không truyền `TransactionManager`.
  **Fix**: Luôn inject `PlatformTransactionManager` từ Spring context.

- **Mistake**: Đặt quá nhiều logic nặng (như gọi API chậm) bên trong khối `execute()`.
  **Fix**: Chỉ để logic thao tác Database bên trong template.

## 12. Sample project
Xây dựng một hệ thống nạp tiền:
1. Xác thực thẻ (ngoài TX).
2. Thực hiện nạp tiền và cập nhật số dư (trong `TransactionTemplate`).
3. Gửi tin nhắn xác nhận thành công (ngoài TX).

## 13. Interview
### Core Q&A
1. **Q**: Tại sao dùng `TransactionTemplate` lại tránh được lỗi self-invocation?
   **A**: Vì nó không dựa trên cơ chế AOP Proxy. Nó là một đối tượng Java bình thường thực thi code bên trong một callback, nên nó hoạt động bất kể được gọi từ đâu.
2. **Q**: Làm thế nào để quy định `Propagation` hoặc `Isolation` cho `TransactionTemplate`?
   **A**: Có thể thiết lập trực tiếp qua các setter: `template.setPropagationBehavior(Propagation.REQUIRES_NEW.value())`.
3. **Q**: Sự khác biệt giữa `execute` và `executeWithoutResult` là gì?
   **A**: `execute` cho phép trả về một giá trị, còn `executeWithoutResult` (thường dùng thông qua `TransactionCallbackWithoutResult`) thì không.

### Scenario
**Tình huống**: Bạn muốn thực hiện 1000 lượt lưu dữ liệu, mỗi lượt là một transaction độc lập. Bạn dùng gì?
**Trả lời**: Tôi sẽ dùng `TransactionTemplate` bên trong một vòng lặp. Cấu hình `propagation` là `REQUIRES_NEW` hoặc đơn giản là gọi `execute()` trong mỗi vòng lặp để đảm bảo mỗi lượt là một đơn vị công việc riêng biệt.

## 14. References
- Official Docs: [Programmatic Transaction Management](https://docs.spring.io/spring-framework/docs/current/reference/html/data-access.html#tx-prog-template)

## 15. Real-world Code
- Thường dùng trong các Batch job hoặc các Service cần tối ưu hiệu năng cực cao bằng cách thu hẹp phạm vi transaction.

## 16. Community
- Stack Overflow: Tag [transactiontemplate]
- Blog: Baeldung (Programmatic Transactions with Spring).
