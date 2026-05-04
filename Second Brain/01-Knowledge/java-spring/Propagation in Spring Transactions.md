---
created: 2026-05-04
tags:
  - "#type/concept"
  - "#status/draft"
  - "#lang/spring"
  - "#topic/database"
related:
  - "[[IOException]]"
---

## 1. What
`Propagation` (Sự lan truyền giao dịch) là một thuộc tính trong annotation `@Transactional` của Spring. nó định nghĩa cách một giao dịch (transaction) mới sẽ được xử lý khi một phương thức được gọi từ một phương thức khác đã có sẵn một giao dịch đang chạy.

## 2. Why
Trong các hệ thống phức tạp, các phương thức nghiệp vụ thường gọi lẫn nhau. Nếu không có `Propagation`, bạn sẽ không thể kiểm soát được việc: "Nếu phương thức B lỗi, phương thức A có phải rollback theo hay không?" hoặc "Phương thức B có nên chạy độc lập hoàn toàn với phương thức A không?". `Propagation` giúp duy trì tính toàn vẹn dữ liệu (ACID) trong các luồng xử lý lồng nhau.

## 3. Mental Model
Hãy tưởng tượng `Propagation` như quy tắc về **"Số phận của các hành khách trên một con tàu"**. 
- `REQUIRED`: "Tất cả chúng ta đi chung một tàu. Nếu tàu chìm (error), tất cả cùng chìm (rollback)".
- `REQUIRES_NEW`: "Tôi sẽ đi bằng tàu riêng của tôi. Tàu bạn chìm kệ bạn, tàu tôi vẫn về đích".
- `NESTED`: "Tôi đi trên một cái xuồng cứu sinh gắn sau tàu của bạn. Nếu tàu bạn chìm, tôi chìm theo. Nhưng nếu chỉ xuồng của tôi hỏng, tàu bạn vẫn có thể tiếp tục đi".

## 4. Where it fits
Nó nằm ở tầng Service, điều khiển luồng giao dịch giữa các Bean:
`Service A (Transactional) -> Calling -> Service B (Transactional with Propagation)`

## 5. When to use
- `REQUIRED` (Mặc định): Dùng cho hầu hết các trường hợp thông thường.
- `REQUIRES_NEW`: Dùng khi muốn một hành động luôn được thực hiện thành công ngay cả khi logic chính thất bại (ví dụ: Ghi log vào DB, trừ tiền phí giao dịch).
- `MANDATORY`: Dùng khi một phương thức bắt buộc phải được gọi trong một giao dịch có sẵn (tránh việc gọi sai từ Controller).

## 6. When NOT to use
- Không lạm dụng `REQUIRES_NEW` vì nó tạo ra nhiều kết nối database đồng thời, dễ dẫn đến nghẽn kết nối (connection pool exhaustion).
- Không dùng `@Transactional` (và Propagation) cho các phương thức không liên quan đến database (như tính toán RAM, gọi API bên thứ ba).

## 7. Trade-offs
| Propagation | Ý nghĩa | Rollback Behavior |
|-------------|---------|-------------------|
| `REQUIRED` | Dùng chung tx nếu có, nếu không tạo mới. | Một lỗi -> Cả chuỗi rollback. |
| `REQUIRES_NEW` | Luôn tạo tx mới, treo tx cũ lại. | Độc lập hoàn toàn. |
| `NESTED` | Tạo "savepoint" trong tx cũ. | Con rollback không ảnh hưởng cha, nhưng cha rollback ảnh hưởng con. |
| `SUPPORTS` | Có tx thì dùng, không có thì chạy bình thường. | Phụ thuộc vào việc có tx hay không. |

## 8. Alternatives
| Option | So sánh |
|--------|---------|
| `TransactionTemplate` | Kiểm soát giao dịch bằng code (programmatic), linh hoạt hơn nhưng rườm rà hơn annotation. |
| Manual JDBC Transaction | Quá thấp, khó quản lý trong ứng dụng lớn. |

## 9. How
```java
@Service
public class ParentService {
    @Autowired private ChildService childService;

    @Transactional(propagation = Propagation.REQUIRED)
    public void parentMethod() {
        // Logic A
        childService.childMethod(); // Gọi method con
        // Logic B
    }
}

@Service
public class ChildService {
    @Transactional(propagation = Propagation.REQUIRES_NEW)
    public void childMethod() {
        // Luôn chạy trong một transaction mới
        // Ngay cả khi parentMethod() lỗi, dữ liệu ở đây vẫn được commit
    }
}
```

## 10. Production concerns
### Scaling
`REQUIRES_NEW` và `NESTED` (nếu DB hỗ trợ) tốn nhiều tài nguyên hơn vì phải quản lý nhiều trạng thái giao dịch lồng nhau hoặc nhiều connection.

### Failure
Lỗi phổ biến nhất là gọi một phương thức `@Transactional` từ một phương thức khác **trong cùng một class** (Self-invocation). Do cơ chế Proxy của Spring, thuộc tính Propagation sẽ bị bỏ qua hoàn toàn.

### Monitoring
Theo dõi số lượng transaction active và thời gian sống của chúng để tránh tình trạng long-running transactions làm lock database.

## 11. Common mistakes
- **Mistake**: Dùng `REQUIRES_NEW` nhưng mong muốn nó rollback cùng phương thức cha.
  **Fix**: Dùng `REQUIRED`.

- **Mistake**: Gọi phương thức có `Propagation` khác nhau trong cùng một class.
  **Fix**: Tách ra các class (Beans) khác nhau để Proxy của Spring có thể can thiệp.

## 12. Sample project
Xây dựng một hệ thống đặt hàng:
1. Tạo đơn hàng (`REQUIRED`).
2. Gửi email thông báo.
3. Ghi log lịch sử vào database (`REQUIRES_NEW`).
Yêu cầu: Nếu tạo đơn hàng lỗi thì email không được gửi, nhưng bản ghi log (thử tạo đơn thất bại) vẫn phải được lưu lại.

## 13. Interview
### Core Q&A
1. **Q**: Propagation mặc định trong Spring là gì?
   **A**: Là `Propagation.REQUIRED`.
2. **Q**: Sự khác biệt giữa `REQUIRES_NEW` và `NESTED` là gì?
   **A**: `REQUIRES_NEW` tạo một kết nối database mới và độc lập hoàn toàn. `NESTED` sử dụng chung kết nối nhưng tạo ra các `Savepoint` (điểm lưu), cho phép rollback một phần.
3. **Q**: Tại sao `@Transactional` không hoạt động khi gọi nội bộ (self-invocation)?
   **A**: Vì Spring sử dụng AOP Proxy. Khi gọi nội bộ, phương thức được gọi trực tiếp qua `this` thay vì qua Proxy, nên code quản lý transaction không được thực thi.

### Scenario
**Tình huống**: Bạn muốn một phương thức CHỈ được phép gọi từ một phương thức khác đã mở giao dịch. Nếu gọi trực tiếp từ Controller (không có giao dịch), nó phải báo lỗi. Bạn dùng Propagation gì?
**Trả lời**: Tôi dùng `Propagation.MANDATORY`.

## 14. References
- Official Docs: [Spring Transaction Management - Propagation](https://docs.spring.io/spring-framework/docs/current/reference/html/data-access.html#tx-propagation)

## 15. Real-world Code
- `REQUIRES_NEW` cực kỳ hữu ích cho Audit Logging hoặc Sequence Generator.

## 16. Community
- Reddit: r/java, r/springboot
- Blog: "Transaction Propagation Explained" (Baeldung).
