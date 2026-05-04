---
created: 2026-05-04
tags:
  - "#type/concept"
  - "#status/draft"
  - "#lang/spring"
  - "#topic/database"
related:
  - "[[TransactionSynchronizationManager]]"
  - "[[Propagation in Spring Transactions]]"
---

## 1. What
`TransactionSynchronization` là một interface trong Spring Framework cung cấp các phương thức callback để người dùng có thể can thiệp vào các mốc thời gian (lifecycle) của một giao dịch, ví dụ như trước/sau khi commit hoặc khi rollback.

## 2. Why
Đôi khi bạn cần thực hiện một hành động ngoài Database (như gửi Email, đẩy tin nhắn vào Kafka, xóa file tạm) nhưng chỉ khi và chỉ khi giao dịch Database đã được **commit thành công**. Nếu bạn gọi các hành động này trực tiếp trong Service, và sau đó Database bị rollback, bạn sẽ rơi vào tình trạng "dữ liệu giả" (Email đã gửi nhưng đơn hàng không tồn tại). `TransactionSynchronization` giúp đồng bộ hóa các hành động ngoại vi này với trạng thái của transaction.

## 3. Mental Model
Hãy tưởng tượng `TransactionSynchronization` như là các **"Điều khoản bổ sung trong một hợp đồng"**. Khi bạn ký hợp đồng mua nhà (Transaction), bạn ghi thêm: "Nếu việc sang tên thành công (afterCommit), hãy đưa chìa khóa cho tôi. Nếu việc sang tên thất bại (afterCompletion), hãy trả lại tiền cọc cho tôi". Các hành động này chỉ được thực hiện dựa trên kết quả cuối cùng của bản hợp đồng chính.

## 4. Where it fits
Nó hoạt động như một Listener cho `PlatformTransactionManager`:
`Transaction Start -> Business Logic -> [Callback: beforeCommit] -> DB Commit -> [Callback: afterCommit] -> Transaction End`

## 5. When to use
- Khi cần gửi thông báo (Email, SMS, Push) sau khi dữ liệu đã được lưu vĩnh viễn vào DB.
- Khi cần xóa cache hoặc cập nhật Search Index (Elasticsearch) đồng bộ với DB.
- Khi cần dọn dẹp tài nguyên (file tạm, socket) sau khi giao dịch kết thúc.

## 6. When NOT to use
- Khi hành động đó có thể được thực hiện bằng giải pháp mạnh mẽ hơn như **Transactional Outbox Pattern** (đảm bảo tính tin cậy cao hơn).
- Khi hành động đó không quan trọng và có thể chạy bất đồng bộ mà không cần quan tâm đến kết quả transaction.

## 7. Trade-offs
| Pros | Cons |
|------|------|
| Đảm bảo tính nhất quán giữa DB và các hệ thống bên ngoài. | Phức tạp hơn việc dùng `@Transactional` thông thường. |
| Giảm thiểu rủi ro "side effects" khi rollback. | Nếu hành động sau commit bị lỗi, DB đã commit rồi nên không thể rollback lại được nữa. |
| Code sạch sẽ, tách biệt logic nghiệp vụ và logic đồng bộ. | Khó debug nếu có quá nhiều synchronization lồng nhau. |

## 8. Alternatives
| Option | So sánh |
|--------|---------|
| `@TransactionalEventListener` | Cách tiếp cận hiện đại hơn của Spring, dùng Event thay vì đăng ký interface trực tiếp. |
| Chạy thủ công sau khi thoát `@Transactional` | Dễ quên và khó quản lý nếu logic phức tạp. |

## 9. How
```java
import org.springframework.transaction.support.TransactionSynchronization;
import org.springframework.transaction.support.TransactionSynchronizationManager;

@Service
public class OrderService {

    @Transactional
    public void createOrder(Order order) {
        // 1. Lưu vào DB
        repository.save(order);

        // 2. Đăng ký hành động sau khi commit thành công
        if (TransactionSynchronizationManager.isSynchronizationActive()) {
            TransactionSynchronizationManager.registerSynchronization(new TransactionSynchronization() {
                @Override
                public void afterCommit() {
                    // Chỉ chạy nếu DB commit thành công
                    emailService.sendOrderConfirmation(order);
                }
            });
        }
    }
}
```

## 10. Production concerns
### Scaling
Các hành động trong `afterCommit` nên được thực hiện nhanh hoặc đẩy vào một thread pool khác (Async) để tránh giữ kết nối database quá lâu (mặc dù transaction đã commit nhưng connection có thể chưa được trả về pool ngay lập tức).

### Failure
Nếu logic trong `afterCommit` ném ngoại lệ, nó sẽ không làm rollback dữ liệu trong DB (vì đã commit rồi). Cần có cơ chế log và retry cho các hành động này.

### Monitoring
Theo dõi số lượng synchronization được đăng ký trong một request để tránh rò rỉ bộ nhớ nếu code logic bị lặp.

## 11. Common mistakes
- **Mistake**: Đăng ký synchronization khi không có transaction nào đang chạy.
  **Fix**: Luôn kiểm tra `TransactionSynchronizationManager.isSynchronizationActive()` trước khi đăng ký.

- **Mistake**: Thực hiện các truy vấn DB mới trong `afterCommit` bằng cùng một transaction.
  **Fix**: Transaction đã kết thúc, nếu cần dùng DB tiếp hãy dùng một transaction mới (REQUIRES_NEW).

## 12. Sample project
Xây dựng một tính năng Upload video:
1. Lưu thông tin video vào DB.
2. Sau khi commit thành công, kích hoạt một Job ở server khác để bắt đầu xử lý (transcoding) video đó.

## 13. Interview
### Core Q&A
1. **Q**: `afterCommit()` và `afterCompletion(int status)` khác nhau gì?
   **A**: `afterCommit()` chỉ chạy khi thành công. `afterCompletion()` luôn chạy bất kể thành công hay thất bại (trạng thái thành công/thất bại được truyền qua tham số `status`).
2. **Q**: Chuyện gì xảy ra nếu `beforeCommit()` ném ngoại lệ?
   **A**: Transaction sẽ bị rollback ngay lập tức và `afterCommit()` sẽ không được gọi.
3. **Q**: Tại sao nên dùng `@TransactionalEventListener` thay vì interface này?
   **A**: Vì nó giúp code lỏng lẻo (decoupled) hơn, sử dụng cơ chế Event-Driven thay vì phải đăng ký trực tiếp trong code nghiệp vụ.

### Scenario
**Tình huống**: Bạn muốn xóa ảnh cũ trên S3 sau khi user cập nhật ảnh mới thành công trong DB. Bạn dùng phương thức nào của `TransactionSynchronization`?
**Trả lời**: Tôi dùng `afterCommit()`. Nếu update DB lỗi, ảnh cũ trên S3 vẫn còn, đảm bảo tính an toàn.

## 14. References
- Official Docs: [TransactionSynchronization API](https://docs.spring.io/spring-framework/docs/current/javadoc-api/org/springframework/transaction/support/TransactionSynchronization.html)

## 15. Real-world Code
- Thường thấy trong các thư viện tích hợp (như Spring Kafka, Spring AMQP) để gửi message sau khi DB commit.

## 16. Community
- Reddit: r/java, r/springboot
- Blog: "Transaction Synchronization in Spring" (Baeldung).
