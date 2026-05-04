---
created: 2026-05-04
tags:
  - "#type/concept"
  - "#status/draft"
  - "#lang/spring"
  - "#topic/database"
related:
  - "[[Page in Spring Data]]"
  - "[[PageRequest in Spring Data]]"
---

## 1. What
`Pageable` là một interface trong Spring Data định nghĩa các thông tin cần thiết để thực hiện một truy vấn phân trang và sắp xếp, bao gồm số trang (`pageNumber`), kích thước trang (`pageSize`), và tiêu chí sắp xếp (`Sort`).

## 2. Why
Trong các ứng dụng thực tế, client (Web/Mobile) cần kiểm soát việc lấy dữ liệu theo từng phần. Thay vì phải truyền thủ công các tham số `offset`, `limit` và `order by` vào từng phương thức repository, `Pageable` cung cấp một cách tiếp cận chuẩn hóa và trừu tượng, giúp code sạch hơn và dễ bảo trì hơn.

## 3. Mental Model
Hãy tưởng tượng `Pageable` như một **"Phiếu yêu cầu mượn sách"**. Trên phiếu đó bạn ghi rõ: "Tôi muốn mượn sách ở kệ số 5 (pageNumber), mỗi lần tôi chỉ cầm được 10 cuốn (pageSize), và hãy xếp chúng theo bảng chữ cái (Sort)". Thủ thư (Repository) sẽ nhìn vào phiếu đó và đưa cho bạn chính xác những gì bạn yêu cầu.

## 4. Where it fits
Nó đóng vai trò là tham số đầu vào cho các phương thức trong Repository:
`Controller (nhận tham số) -> Pageable (Yêu cầu) -> Repository (Thực thi) -> Page/Slice (Kết quả)`

## 5. When to use
- Khi định nghĩa các phương thức tìm kiếm trong `PagingAndSortingRepository` hoặc `JpaRepository`.
- Khi muốn hỗ trợ phân trang động từ API của người dùng.
- Khi cần kết hợp phân trang với các bộ lọc phức tạp (như `Specification` hoặc `Querydsl`).

## 6. When NOT to use
- Khi bạn chắc chắn chỉ muốn lấy một bản ghi duy nhất hoặc toàn bộ danh sách nhỏ (dưới 50 bản ghi).
- Khi thực hiện các câu lệnh `native query` phức tạp mà Spring Data không thể tự động parse được `Pageable`.

## 7. Trade-offs
| Pros | Cons |
|------|------|
| Code cực kỳ gọn gàng, giảm lặp lại logic phân trang. | Gây khó khăn nếu muốn tùy chỉnh logic `OFFSET` đặc thù của một số DB cũ. |
| Tích hợp sẵn cơ chế Sorting mạnh mẽ. | Có thể gây hiểu nhầm về index trang (bắt đầu từ 0 thay vì 1). |
| Hỗ trợ tự động ánh xạ từ tham số URL trong Spring MVC. | |

## 8. Alternatives
| Option | So sánh |
|--------|---------|
| Thủ công `limit/offset` | Rườm rà, dễ lỗi, không nhất quán giữa các Database. |
| `OffsetScrollPosition` | (Mới trong Spring Data) Hỗ trợ phân trang dựa trên vị trí (cursor-based), tốt cho performance. |

## 9. How
```java
// Trong Repository
public interface ProductRepository extends JpaRepository<Product, Long> {
    Page<Product> findByCategory(String category, Pageable pageable);
}

// Trong Service/Controller
import org.springframework.data.domain.Pageable;
import org.springframework.data.domain.PageRequest;
import org.springframework.data.domain.Sort;

public void getProducts() {
    // Tạo Pageable: Trang 0, 20 phần tử, sắp xếp theo giá giảm dần
    Pageable pageable = PageRequest.of(0, 20, Sort.by("price").descending());
    
    Page<Product> products = productRepository.findByCategory("Electronics", pageable);
}
```

## 10. Production concerns
### Scaling
Sử dụng `Pageable` với `OFFSET` lớn (ví dụ trang 10.000) sẽ rất chậm vì Database vẫn phải quét qua 100.000 bản ghi đầu tiên rồi mới bỏ đi. Cần xem xét Cursor-based pagination cho các hệ thống cực lớn.

### Failure
Nếu client gửi `pageSize` quá lớn (ví dụ 1.000.000), hệ thống có thể bị treo. Luôn đặt giới hạn tối đa (Max Page Size) trong cấu hình ứng dụng.

### Monitoring
Theo dõi các request có `pageNumber` lớn để tối ưu hóa index hoặc tư vấn người dùng thu hẹp bộ lọc.

## 11. Common mistakes
- **Mistake**: Quên rằng `pageNumber` bắt đầu từ **0**. Gửi trang 1 từ UI sẽ lấy dữ liệu của trang 2.
  **Fix**: Luôn trừ đi 1 ở Controller nếu UI bắt đầu từ trang 1.

- **Mistake**: Không xử lý trường hợp `Sort` rỗng hoặc sai tên field.
  **Fix**: Sử dụng `Sort.unsorted()` hoặc kiểm tra field trước khi tạo `Sort`.

## 12. Sample project
Xây dựng một API tìm kiếm bài viết, cho phép người dùng truyền `page`, `size` và `sort` trực tiếp vào URL (ví dụ: `/posts?page=0&size=5&sort=createdAt,desc`). Spring MVC sẽ tự động convert các param này thành một đối tượng `Pageable`.

## 13. Interview
### Core Q&A
1. **Q**: Tại sao `Pageable` là một interface?
   **A**: Để cho phép nhiều implementation khác nhau (như `PageRequest`, `Unpaged`) và cho phép framework mở rộng các cách thức phân trang khác trong tương lai.
2. **Q**: Làm thế nào để lấy dữ liệu mà KHÔNG phân trang bằng Pageable?
   **A**: Sử dụng `Pageable.unpaged()`.
3. **Q**: Spring MVC hỗ trợ ánh xạ `Pageable` từ URL như thế nào?
   **A**: Thông qua `PageableHandlerMethodArgumentResolver`. Nó sẽ tìm các param `page`, `size`, `sort` trong request.

### Scenario
**Tình huống**: Bạn muốn sắp xếp kết quả theo hai tiêu chí: theo Ngày tạo giảm dần, sau đó theo ID tăng dần. Bạn tạo `Pageable` như thế nào?
**Trả lời**: Tôi dùng `PageRequest.of(page, size, Sort.by("createdAt").descending().and(Sort.by("id").ascending()))`.

## 14. References
- Official Docs: [Spring Data Domain - Pageable](https://docs.spring.io/spring-data/commons/docs/current/api/org/springframework/data/domain/Pageable.html)

## 15. Real-world Code
- Luôn là tham số cuối cùng của mọi phương thức query trong Repository.

## 16. Community
- Reddit: r/java, r/springboot
- Blog: Baeldung (Pagination and Sorting with Spring Data JPA).
