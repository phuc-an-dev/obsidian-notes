---
created: 2026-05-04
tags:
  - "#type/concept"
  - "#status/draft"
  - "#lang/spring"
  - "#topic/database"
related:
  - "[[Pageable in Spring Data]]"
  - "[[PageRequest in Spring Data]]"
  - "[[specification-spring-data-jpa]]"
---

## 1. What
`Page` là một interface trong Spring Data đại diện cho một danh sách dữ liệu con (slice) đi kèm với thông tin metadata về tổng số phần tử (`totalElements`) và tổng số trang (`totalPages`) hiện có trong cơ sở dữ liệu. Nó được dùng để trả về kết quả của một truy vấn phân trang.

## 2. Why
Khi xử lý hàng triệu bản ghi, việc trả về toàn bộ dữ liệu cho client là không khả thi. `Page` không chỉ chứa dữ liệu cần hiển thị mà còn cung cấp các thông tin cần thiết để client xây dựng giao diện phân trang (như nút "Trang tiếp theo", "Trang cuối cùng", hoặc hiển thị "Hiển thị 10 trên 1000 kết quả").

## 3. Mental Model
Hãy tưởng tượng `Page` như một **"Trang sách trong một cuốn sách dày"**. Nội dung của trang là dữ liệu bạn đang đọc, nhưng ở dưới chân trang (metadata) luôn có ghi chú: "Bạn đang ở trang 5 của 100 trang" và "Tổng cộng có 2000 dòng văn bản". Nhờ thông tin này, bạn biết mình còn bao xa nữa mới hết cuốn sách.

## 4. Where it fits
Nó là kiểu dữ liệu trả về cuối cùng từ Repository và được truyền ra Controller:
`Database -> Repository (trả về Page) -> Service -> Controller (trả về JSON cho Client)`

## 5. When to use
- Khi cần hiển thị dữ liệu dạng bảng có phân trang trên UI (Web/Mobile).
- Khi client cần biết tổng số lượng bản ghi để tính toán logic hiển thị.
- Khi cần thực hiện các truy vấn yêu cầu cả dữ liệu và thống kê tổng số lượng trong một lần gọi duy nhất.

## 6. When NOT to use
- Khi bạn thực hiện "Infinite Scroll" (Cuộn vô hạn) mà không cần biết tổng số trang. Trong trường hợp này, hãy dùng `Slice` để tiết kiệm một câu lệnh `count` trong database.
- Khi danh sách dữ liệu rất nhỏ và cố định (dưới 100 bản ghi), trả về `List` sẽ đơn giản và nhanh hơn.

## 7. Trade-offs
| Pros | Cons |
|------|------|
| Cung cấp metadata đầy đủ cho UI. | Tốn thêm chi phí thực hiện câu lệnh `SELECT COUNT` bổ sung. |
| Tích hợp cực tốt với Spring Data JPA. | Metadata có thể làm phình to JSON response nếu không cần thiết. |
| Hỗ trợ sẵn các helper methods như `isFirst()`, `isLast()`. | |

## 8. Alternatives
| Option | So sánh |
|--------|---------|
| `Slice` | Không thực hiện `count`, chỉ biết có trang tiếp theo hay không. Hiệu năng tốt hơn `Page`. |
| `List` | Không có thông tin phân trang, phù hợp cho tập dữ liệu nhỏ. |
| `WindowIterator` | (Dùng trong Batch job) Để duyệt qua lượng dữ liệu khổng lồ. |

## 9. How
```java
import org.springframework.data.domain.Page;
import org.springframework.data.domain.PageRequest;
import org.springframework.data.domain.Pageable;
import org.springframework.web.bind.annotation.GetMapping;
import org.springframework.web.bind.annotation.RequestParam;
import org.springframework.web.bind.annotation.RestController;

@RestController
public class UserController {

    private final UserRepository userRepository;

    public UserController(UserRepository userRepository) {
        this.userRepository = userRepository;
    }

    @GetMapping("/users")
    public Page<User> getUsers(@RequestParam(defaultValue = "0") int page,
                               @RequestParam(defaultValue = "10") int size) {
        Pageable pageable = PageRequest.of(page, size);
        
        // Repository trả về Page<User>
        return userRepository.findAll(pageable);
    }
}
```

## 10. Production concerns
### Scaling
Câu lệnh `COUNT` để tính tổng số phần tử có thể trở nên rất chậm trên các bảng có hàng chục triệu bản ghi nếu không được index đúng cách hoặc có các điều kiện `WHERE` phức tạp.

### Failure
Nếu client yêu cầu một trang không tồn tại (ví dụ trang 100 trong khi chỉ có 10 trang), `Page` sẽ trả về một danh sách rỗng (Empty list) thay vì ném lỗi. Cần logic xử lý ở frontend cho trường hợp này.

### Monitoring
Theo dõi thời gian thực thi của câu lệnh `COUNT` so với câu lệnh `SELECT` dữ liệu để tối ưu hóa database.

## 11. Common mistakes
- **Mistake**: Sử dụng `Page` cho các ứng dụng mobile chỉ cần "Load more".
  **Fix**: Dùng `Slice` để tránh lãng phí câu lệnh `COUNT`.

- **Mistake**: Không map `Page<Entity>` sang `Page<DTO>` trước khi trả về Controller.
  **Fix**: Sử dụng `page.map(entity -> convertToDto(entity))` để bảo mật và tối ưu JSON.

## 12. Sample project
Xây dựng một API quản lý sản phẩm (E-commerce), trả về `Page<ProductDTO>`. Frontend sẽ sử dụng metadata từ `Page` để hiển thị bộ phân trang (Pagination bar) với các số trang 1, 2, 3...

## 13. Interview
### Core Q&A
1. **Q**: Sự khác biệt giữa `Page` và `Slice` là gì?
   **A**: `Page` thực hiện thêm một câu truy vấn `COUNT` để biết tổng số phần tử/trang, còn `Slice` chỉ biết có phần tử tiếp theo hay không bằng cách lấy dư 1 phần tử (limit + 1).
2. **Q**: Tại sao trả về `Page` lại tốn kém hơn `List`?
   **A**: Vì Spring Data JPA phải thực thi 2 câu lệnh SQL: một câu `SELECT` dữ liệu với `LIMIT/OFFSET` và một câu `SELECT COUNT` để lấy tổng số.
3. **Q**: Làm thế nào để convert `Page<Entity>` sang `Page<DTO>`?
   **A**: Sử dụng phương thức `.map()` có sẵn của interface `Page`.

### Scenario
**Tình huống**: Hệ thống của bạn bắt đầu chậm đi khi số lượng bản ghi đạt 10 triệu. Qua profiling, bạn thấy câu lệnh `COUNT` chiếm 80% thời gian truy vấn. Bạn giải quyết thế nào?
**Trả lời**: Tôi sẽ xem xét chuyển sang dùng `Slice` nếu UI không bắt buộc phải hiển thị tổng số trang. Nếu bắt buộc, tôi sẽ cân nhắc việc cache lại `totalElements` hoặc sử dụng một bảng thống kê riêng thay vì đếm trực tiếp trên bảng chính.

## 14. References
- Official Docs: [Spring Data Pagination](https://docs.spring.io/spring-data/jpa/docs/current/reference/html/#repositories.paging-and-sorting)

## 15. Real-world Code
- Luôn xuất hiện trong các API liệt kê (Listing API) của các hệ thống quản trị (Admin Dashboard).

## 16. Community
- Stack Overflow: Tag [spring-data-jpa], [pagination]
- Blog: "Pagination and Sorting with Spring Data JPA" (Baeldung).
