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
---

## 1. What
`Sort` là một lớp trong Spring Data dùng để định nghĩa các tiêu chí sắp xếp kết quả truy vấn. Nó cho phép bạn chỉ định một hoặc nhiều trường (fields) cần sắp xếp cùng với hướng sắp xếp (tăng dần - ASC hoặc giảm dần - DESC).

## 2. Why
Trong các ứng dụng thực tế, việc sắp xếp dữ liệu (ví dụ: bài viết mới nhất, giá rẻ nhất) là cực kỳ quan trọng. `Sort` cung cấp một cách tiếp cận hướng đối tượng (Object-oriented) để định nghĩa logic này, thay vì phải viết cứng các chuỗi "ORDER BY" vào SQL, giúp code an toàn và dễ tái sử dụng hơn.

## 3. Mental Model
Hãy tưởng tượng `Sort` như một **"Danh sách ưu tiên"** bạn đưa cho một người phụ tá. Bạn nói: "Hãy tìm cho tôi các sản phẩm, trước tiên hãy xếp theo **Ngày nhập (ASC)**, nếu ngày giống nhau thì hãy xếp theo **Giá (DESC)**". Người phụ tá sẽ dựa vào danh sách này để trình bày kết quả cho bạn.

## 4. Where it fits
Nó thường là một phần của đối tượng `Pageable` hoặc được truyền trực tiếp vào các phương thức repository:
`Logic nghiệp vụ -> Sort -> Pageable/Repository -> SQL ORDER BY`

## 5. When to use
- Khi cần sắp xếp dữ liệu trả về từ Repository.
- Khi muốn kết hợp nhiều tiêu chí sắp xếp phức tạp.
- Khi muốn cho phép người dùng cuối tự chọn cách sắp xếp từ giao diện (UI).

## 6. When NOT to use
- Khi tiêu chí sắp xếp quá phức tạp hoặc phụ thuộc vào logic tính toán phía Database mà Spring Data không hỗ trợ (trường hợp này dùng Native Query).
- Khi bạn chỉ sắp xếp một mảng nhỏ trong RAM (dùng `Collections.sort()` hoặc Stream API sẽ nhanh hơn).

## 7. Trade-offs
| Pros | Cons |
|------|------|
| API mượt mà (Fluent API), dễ đọc. | Tên field truyền vào dạng String, dễ lỗi nếu gõ sai (không type-safe). |
| Kết hợp được nhiều field dễ dàng. | Có thể gây chậm truy vấn nếu không có Index trên các field sắp xếp. |
| Tích hợp sẵn với `Pageable`. | |

## 8. Alternatives
| Option | So sánh |
|--------|---------|
| `OrderBy` trong Query Method | Tiện nhưng làm tên phương thức repository rất dài (ví dụ `findAllByOrderByCreatedAtDesc`). |
| `JpaSort` | Hỗ trợ sắp xếp theo các hàm của JPA (như `LENGTH(name)`). |
| `Querydsl Sort` | Type-safe, tránh lỗi gõ sai tên field. |

## 9. How
```java
import org.springframework.data.domain.Sort;
import org.springframework.data.domain.Sort.Order;

public class SortExample {
    public void createSort() {
        // 1. Sắp xếp đơn giản
        Sort sort1 = Sort.by("name").ascending();

        // 2. Sắp xếp giảm dần
        Sort sort2 = Sort.by("createdAt").descending();

        // 3. Kết hợp nhiều field (Cách 1)
        Sort multiSort1 = Sort.by("status").ascending()
                          .and(Sort.by("priority").descending());

        // 4. Kết hợp nhiều field (Cách 2 - dùng List Order)
        Sort multiSort2 = Sort.by(
            Order.asc("category"),
            Order.desc("price")
        );

        // Truyền vào Repository
        // productRepository.findAll(multiSort1);
    }
}
```

## 10. Production concerns
### Scaling
Việc sắp xếp trên các bảng lớn mà không có Index phù hợp sẽ khiến Database thực hiện "File Sort" - cực kỳ chậm và tốn tài nguyên. Luôn đảm bảo các field hay được sort đã được đánh Index.

### Failure
Nếu truyền sai tên field (ví dụ `created_at` thay vì `createdAt` của Entity), Spring Data sẽ ném ra `PropertyReferenceException` khi thực thi.

### Monitoring
Theo dõi các câu lệnh SQL có `ORDER BY` trong Slow Query Log của Database.

## 11. Common mistakes
- **Mistake**: Sắp xếp theo các field không tồn tại trong Entity.
  **Fix**: Luôn dùng tên biến của Entity, không dùng tên cột trong DB.

- **Mistake**: Quên xử lý trường hợp các giá trị null (`nulls first` hoặc `nulls last`).
  **Fix**: Sử dụng `Order.asc("name").nullsLast()`.

## 12. Sample project
Xây dựng một API tìm kiếm phim, cho phép người dùng chọn sắp xếp theo "Điểm đánh giá" (giảm dần) hoặc "Năm phát hành" (mới nhất). Sử dụng `Sort` để tạo tiêu chí động dựa trên lựa chọn của người dùng.

## 13. Interview
### Core Q&A
1. **Q**: `Sort` trong Spring Data có an toàn về kiểu dữ liệu (type-safe) không?
   **A**: Mặc định là không, vì nó dùng String để chỉ tên field. Để type-safe, có thể dùng `TypedSort` (trong các bản Spring mới) hoặc Querydsl.
2. **Q**: Làm thế nào để kết hợp hai đối tượng `Sort` lại với nhau?
   **A**: Dùng phương thức `.and()`. Ví dụ: `sortA.and(sortB)`.
3. **Q**: Sự khác biệt giữa `Sort.by("name")` và `Sort.Order.asc("name")` là gì?
   **A**: `Sort` là tập hợp của một hoặc nhiều `Order`. `Sort.by()` là phương thức factory tiện lợi để tạo nhanh đối tượng `Sort`.

### Scenario
**Tình huống**: Bạn muốn sắp xếp một danh sách sao cho những bản ghi có `isFeatured = true` luôn hiện lên đầu, sau đó mới đến ngày tạo. Bạn làm thế nào?
**Trả lời**: Tôi dùng `Sort.by(Order.desc("isFeatured"), Order.desc("createdAt"))`. Vì `true` (1) lớn hơn `false` (0) nên sắp xếp DESC sẽ đưa `isFeatured = true` lên trước.

## 14. References
- Official Docs: [Spring Data Sort API](https://docs.spring.io/spring-data/commons/docs/current/api/org/springframework/data/domain/Sort.html)

## 15. Real-world Code
- Luôn đi kèm với `PageRequest` trong các Service layer.

## 16. Community
- Stack Overflow: Tag [spring-data], [sorting]
- Blog: Baeldung (Sorting in Spring Data).
