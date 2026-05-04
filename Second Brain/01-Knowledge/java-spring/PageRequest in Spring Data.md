---
created: 2026-05-04
tags:
  - "#type/concept"
  - "#status/draft"
  - "#lang/spring"
  - "#topic/database"
related:
  - "[[Pageable in Spring Data]]"
  - "[[Page in Spring Data]]"
---

## 1. What
`PageRequest` là một lớp thực thi (implementation) cụ thể của interface `Pageable` trong Spring Data. Nó được sử dụng để tạo ra các đối tượng yêu cầu phân trang và sắp xếp một cách thủ công trong mã nguồn.

## 2. Why
Vì `Pageable` là một interface, bạn không thể khởi tạo nó trực tiếp bằng từ khóa `new`. `PageRequest` cung cấp các phương thức tĩnh (static factory methods) như `.of(...)` để bạn dễ dàng tạo ra một yêu cầu phân trang với các tham số cụ thể từ logic nghiệp vụ của mình.

## 3. Mental Model
Nếu `Pageable` là cái **"Mẫu phiếu mượn sách"** (interface), thì `PageRequest` chính là cái **"Phiếu đã được điền đầy đủ thông tin"** (implementation) mà bạn nộp cho thủ thư. Nó chứa các con số cụ thể như: "Tôi mượn trang 2, mỗi trang 5 cuốn".

## 4. Where it fits
Nó nằm ở tầng Service hoặc Controller, nơi bạn quyết định trang nào và bao nhiêu dữ liệu sẽ được lấy từ Database:
`Logic nghiệp vụ -> PageRequest.of() -> Repository.findAll(Pageable)`

## 5. When to use
- Khi bạn cần tạo yêu cầu phân trang thủ công trong code (Service layer).
- Khi muốn áp dụng một logic sắp xếp (Sorting) mặc định trước khi gọi Repository.
- Khi viết các Unit Test/Integration Test cho Repository yêu cầu tham số phân trang.

## 6. When NOT to use
- Khi bạn muốn Spring MVC tự động ánh xạ tham số từ URL (Spring sẽ tự dùng resolver để tạo Pageable, bạn không cần dùng `PageRequest` thủ công).
- Khi bạn muốn tắt tính năng phân trang (dùng `Pageable.unpaged()`).

## 7. Trade-offs
| Pros | Cons |
|------|------|
| Dễ sử dụng với các factory methods tiện lợi. | Bắt buộc phải truyền đúng tham số (page, size) nếu không sẽ ném `IllegalArgumentException`. |
| Hỗ trợ tích hợp sẵn với lớp `Sort`. | Không hỗ trợ "Key-based" pagination (Cursor). |
| Code tường minh, dễ hiểu. | |

## 8. Alternatives
| Option | So sánh |
|--------|---------|
| `QPageRequest` | Implementation dùng cho Querydsl, hỗ trợ type-safe sorting. |
| `Pageable.ofSize(n)` | Cách tạo nhanh một Pageable chỉ với size (mặc định trang 0). |

## 9. How
```java
import org.springframework.data.domain.PageRequest;
import org.springframework.data.domain.Pageable;
import org.springframework.data.domain.Sort;

public class MyService {

    public void processData() {
        // 1. Phân trang đơn giản (Trang 0, 10 phần tử)
        Pageable firstPageWithTenElements = PageRequest.of(0, 10);

        // 2. Phân trang có sắp xếp (Trang 1, 5 phần tử, theo 'name' tăng dần)
        Pageable secondPageWithFiveElements = PageRequest.of(1, 5, Sort.by("name"));

        // 3. Phân trang sắp xếp phức tạp
        Pageable complexRequest = PageRequest.of(0, 20, 
            Sort.by("price").descending().and(Sort.by("name").ascending()));
            
        // Truyền vào Repository
        // myRepository.findAll(complexRequest);
    }
}
```

## 10. Production concerns
### Scaling
Dùng `PageRequest` để tạo ra các request có `pageSize` quá lớn có thể làm cạn kiệt bộ nhớ của ứng dụng khi load quá nhiều Entity vào Persistence Context.

### Failure
Nếu bạn truyền `page < 0` hoặc `size < 1`, `PageRequest` sẽ ném ra lỗi ngay lập tức. Luôn validate tham số từ người dùng trước khi truyền vào.

### Monitoring
Log lại các tham số `page` và `size` trong các câu query chậm để tìm ra các yêu cầu bất thường từ phía người dùng.

## 11. Common mistakes
- **Mistake**: Khởi tạo `PageRequest` bằng constructor `new PageRequest(...)`.
  **Fix**: Constructor này đã bị deprecated hoặc protected trong các bản Spring Data mới. Luôn dùng `PageRequest.of(...)`.

- **Mistake**: Truyền tham số `page` từ client (thường bắt đầu từ 1) trực tiếp vào `PageRequest.of()`.
  **Fix**: Luôn kiểm tra và trừ đi 1: `PageRequest.of(userPage - 1, size)`.

## 12. Sample project
Xây dựng một tính năng "Gợi ý sản phẩm" trong Service, mặc định luôn lấy 5 sản phẩm mới nhất (sắp xếp theo `createdAt` DESC) bằng cách sử dụng `PageRequest.of(0, 5, Sort.by("createdAt").descending())`.

## 13. Interview
### Core Q&A
1. **Q**: Tại sao nên dùng `PageRequest.of()` thay vì `new PageRequest()`?
   **A**: Vì `PageRequest.of()` là một Static Factory Method, nó giúp code sạch hơn và framework có thể thực hiện một số kiểm tra hoặc tối ưu hóa bên trong trước khi trả về object.
2. **Q**: `PageRequest` có thread-safe không?
   **A**: Có, nó là một đối tượng **Immutable** (bất biến). Sau khi tạo xong, các giá trị page và size không thể thay đổi.
3. **Q**: Làm thế nào để tạo một `PageRequest` mà không có Sorting?
   **A**: Chỉ cần gọi `PageRequest.of(page, size)` hoặc `PageRequest.of(page, size, Sort.unsorted())`.

### Scenario
**Tình huống**: Bạn muốn viết một Test Case để kiểm tra xem Repository có phân trang đúng không. Bạn làm thế nào?
**Trả lời**: Tôi sẽ tạo một đối tượng `PageRequest.of(0, 5)`, truyền nó vào method của Repository, sau đó assert rằng kết quả `Page.getSize()` trả về đúng là 5 và `Page.getNumber()` trả về 0.

## 14. References
- Official Docs: [Spring Data - PageRequest API](https://docs.spring.io/spring-data/commons/docs/current/api/org/springframework/data/domain/PageRequest.html)

## 15. Real-world Code
- Được dùng phổ biến nhất ở tầng Service để chuẩn hóa yêu cầu trước khi gọi Repository.

## 16. Community
- Stack Overflow: Tag [spring-data], [pagerequest]
- Blog: "Mastering Pagination in Spring Data JPA".
