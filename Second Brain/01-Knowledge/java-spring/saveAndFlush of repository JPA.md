---
created: 2026-04-21
tags:
  - "#type/concept"
  - "#status/draft"
  - "#lang/spring"
  - "#topic/database"
related:
  - "[[specification-spring-data-jpa]]"
  - "[[criteria-api]]"
---

# saveAndFlush of Spring Data JPA Repository

## 1. What
`saveAndFlush` là một phương thức được cung cấp bởi `JpaRepository` trong Spring Data JPA. Nó thực hiện đồng thời hai thao tác: lưu (hoặc cập nhật) entity vào persistence context và ngay lập tức đồng bộ hóa (flush) các thay đổi đó xuống database.

## 2. Why
Trong JPA, các thay đổi trên entity thường được giữ trong persistence context (L1 cache) và chỉ được đẩy xuống database khi transaction commit hoặc khi bộ nhớ đệm đầy. Có những trường hợp logic nghiệp vụ cần dữ liệu phải có trong database ngay lập tức trước khi transaction kết thúc (ví dụ: cần trigger database, ràng buộc foreign key, hoặc cần lấy giá trị auto-generated của ID ngay sau khi lưu).

## 3. Mental Model
Hãy tưởng tượng persistence context là một "bản nháp" trên bàn làm việc:
- `save()`: Bạn ghi chú vào bản nháp đó. Nhân viên kho chỉ nhận được thông tin khi bạn rời khỏi văn phòng (commit).
- `saveAndFlush()`: Bạn ghi chú vào bản nháp và **gọi nhân viên kho đến nhận ngay lập tức**. Dữ liệu đã vào kho, dù bạn vẫn đang ngồi làm việc tiếp.

## 4. Where it fits
```
[Repository.saveAndFlush(entity)] 
       |
       v
[Persistence Context] (L1 Cache)
       |
       v (Flush - immediate SQL INSERT/UPDATE)
[Database]
       |
       v (Continue business logic)
[Transaction Commit]
```

## 5. When to use
- Khi cần lấy giá trị ID (được sinh tự động từ DB) ngay sau khi gọi save.
- Khi cần đảm bảo dữ liệu đã tồn tại trong DB để kích hoạt database trigger hoặc các stored procedure.
- Khi cần xử lý lỗi SQL ràng buộc (unique constraint) ngay tại dòng code đó thay vì đợi đến khi commit.

## 6. When NOT to use
- Đừng lạm dụng trong vòng lặp (loop): `saveAndFlush` gây ra nhiều câu lệnh SQL lẻ tẻ, ảnh hưởng đến hiệu năng đáng kể. Dùng `saveAll` hoặc để JPA tự động flush khi kết thúc transaction.
- Không cần dùng nếu không có ràng buộc chặt chẽ về thứ tự ghi xuống DB trong cùng một transaction.

## 7. Trade-offs
| Pros | Cons |
|------|------|
| Dữ liệu được persist tức thời | Giảm hiệu năng do ép flush liên tục |
| Thích hợp cho các ràng buộc dữ liệu đặc thù | Bỏ qua cơ chế batching mặc định của Hibernate |

## 8. Alternatives
| Option | So sánh |
|--------|---------|
| `save()` | Default, hiệu năng tốt hơn nhờ batching và lazy writing |
| `flush()` | Chỉ đẩy thay đổi xuống mà không cần gọi lại `save()` trên entity |

## 9. How
```java
@Transactional
public void createOrder(Order order) {
    // Lưu vào DB ngay lập tức
    Order savedOrder = repository.saveAndFlush(order);
    
    // ID lúc này đã có giá trị từ DB
    System.out.println("Generated ID: " + savedOrder.getId());
    
    // Logic tiếp theo phụ thuộc vào dữ liệu đã lưu
    sendNotification(savedOrder);
}
```

## 10. Production concerns
- **Performance**: Việc gọi `saveAndFlush` nhiều lần trong một transaction lớn sẽ làm chậm ứng dụng đáng kể. 
- **Deadlocks**: Vì dữ liệu được đẩy xuống DB sớm, nó có thể giữ lock trên các row lâu hơn mức cần thiết, dễ dẫn đến tranh chấp lock (deadlock) trong môi trường concurrency cao.

## 11. Common mistakes
- Mistake: Gọi `saveAndFlush` trong vòng lặp `for`.
  Fix: Gom dữ liệu lại và dùng `saveAll()` hoặc để Spring tự flush.
- Mistake: Nghĩ rằng `saveAndFlush` đã commit transaction.
  Fix: Nhớ rằng `saveAndFlush` chỉ thực hiện `flush` (SQL statements), transaction vẫn mở và chỉ kết thúc khi method kết thúc hoặc xảy ra exception.

## 12. Sample project
Sử dụng một entity `User` với `GenerationType.IDENTITY`. Thử nghiệm lưu `User` và in ra ID ngay sau đó. Sau đó thử dùng `save()` và kiểm tra khi nào ID mới có giá trị.

## 13. Interview
### Core Q&A
1. Q: Sự khác biệt giữa `save()` và `saveAndFlush()`?
   A: `save()` chỉ cập nhật persistence context và Hibernate quản lý việc flush khi commit. `saveAndFlush()` ép buộc gửi lệnh SQL (INSERT/UPDATE) xuống DB ngay lập tức.
2. Q: Tại sao không nên dùng `saveAndFlush` trong vòng lặp?
   A: Vì nó vô hiệu hóa các tối ưu hóa của Hibernate (như batching) và tăng đáng kể số lượng round-trip tới database.

## 14. References
- Spring Data JPA Docs: https://docs.spring.io/spring-data/jpa/reference/
- Hibernate Flush Guide: https://hibernate.org/orm/documentation/
