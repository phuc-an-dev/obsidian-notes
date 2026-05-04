---
created: 2026-05-04
tags:
  - "#type/concept"
  - "#status/draft"
  - "#lang/spring"
  - "#topic/database"
related:
  - "[[specification-spring-data-jpa]]"
  - "[[Pageable in Spring Data]]"
---

## 1. What
`Specification` là một interface trong Spring Data JPA dựa trên **Criteria API** của JPA. Nó cho phép lập trình viên xây dựng các câu truy vấn (query) một cách động, linh hoạt và có thể tái sử dụng bằng cách kết hợp các điều kiện tìm kiếm lại với nhau.

## 2. Why
Thông thường, khi viết các phương thức tìm kiếm trong Repository (như `findByFirstNameAndLastName`), số lượng phương thức sẽ bùng nổ nếu bạn có nhiều tiêu chí lọc (filter) kết hợp với nhau. `Specification` giải quyết vấn đề này bằng cách cho phép bạn định nghĩa các điều kiện nhỏ lẻ (atomic) và "lắp ghép" chúng lại tùy theo tham số mà người dùng gửi lên.

## 3. Mental Model
Hãy tưởng tượng `Specification` như một bộ các **"Mảnh ghép lọc nước"**. Bạn có mảnh ghép "Lọc cát" (check tên), mảnh ghép "Lọc kim loại" (check tuổi), mảnh ghép "Lọc vi khuẩn" (check trạng thái). Tùy vào nguồn nước bẩn thế nào (input từ user), bạn sẽ lắp các mảnh ghép đó lại với nhau để có được nước sạch (kết quả truy vấn).

## 4. Where it fits
Nó thay thế cho các Query Methods rườm rà trong Repository:
`Controller (nhận filter) -> Specification Builder -> Repository.findAll(spec) -> Database`

## 5. When to use
- Khi ứng dụng có màn hình tìm kiếm nâng cao với nhiều trường lọc không bắt buộc (Optional filters).
- Khi muốn tái sử dụng một logic lọc dữ liệu ở nhiều nơi khác nhau.
- Khi cần xây dựng các câu truy vấn phức tạp mà Query Methods không thể đáp ứng.

## 6. When NOT to use
- Khi truy vấn cực kỳ đơn giản (chỉ lọc theo 1-2 trường cố định).
- Khi bạn không dùng JPA (Specification là đặc thù của Spring Data JPA).
- Khi hiệu năng Criteria API không đáp ứng được (truy vấn quá phức tạp, cần Native SQL).

## 7. Trade-offs
| Pros | Cons |
|------|------|
| Truy vấn động cực kỳ mạnh mẽ và linh hoạt. | Cú pháp Criteria API khá rườm rà và khó đọc. |
| Code sạch, tránh được tình trạng "nổ" phương thức repository. | Khó debug hơn so với SQL thuần. |
| Có khả năng tái sử dụng cao (Composition). | Phụ thuộc chặt chẽ vào cấu trúc Entity. |

## 8. Alternatives
| Option | So sánh |
|--------|---------|
| `Querydsl` | Mạnh mẽ hơn, type-safe hơn nhưng cần cấu hình code generation. |
| `ExampleMatcher` | Dễ dùng cho các case đơn giản nhưng không hỗ trợ các phép so sánh phức tạp (like, between). |
| Native Query | Linh hoạt nhất nhưng không thể lắp ghép động dễ dàng như Specification. |

## 9. How
```java
// 1. Repository phải extends JpaSpecificationExecutor
public interface CustomerRepository extends JpaRepository<Customer, Long>, JpaSpecificationExecutor<Customer> {
}

// 2. Định nghĩa các Specification
public class CustomerSpecs {
    public static Specification<Customer> hasName(String name) {
        return (root, query, cb) -> cb.like(root.get("name"), "%" + name + "%");
    }

    public static Specification<Customer> isVip() {
        return (root, query, cb) -> cb.equal(root.get("status"), "VIP");
    }
}

// 3. Sử dụng trong Service
Specification<Customer> spec = Specification.where(CustomerSpecs.hasName("An"))
                                            .and(CustomerSpecs.isVip());
List<Customer> customers = customerRepository.findAll(spec);
```

## 10. Production concerns
### Scaling
Việc sử dụng `root.get("field")` nhiều lần trong Specification có thể gây ra các câu lệnh `JOIN` ngầm nếu không cẩn thận. Cần sử dụng `root.fetch()` để tối ưu hóa Eager Loading nếu cần thiết.

### Failure
Nếu tên field truyền vào `root.get()` bị sai, lỗi sẽ chỉ xuất hiện khi ứng dụng chạy (Runtime). Sử dụng **JPA Metamodel** để giúp Specification trở nên type-safe.

### Monitoring
Kiểm tra các câu lệnh SQL sinh ra bởi Hibernate để đảm bảo Specification không tạo ra các câu query quá cồng kềnh hoặc thiếu Index.

## 11. Common mistakes
- **Mistake**: Quên kiểm tra null cho tham số đầu vào trước khi tạo Specification.
  **Fix**: Luôn kiểm tra `if (param != null)` để tránh đưa các điều kiện vô nghĩa vào query.

- **Mistake**: Không sử dụng `JpaSpecificationExecutor` cho Repository.
  **Fix**: Repository bắt buộc phải extends interface này để có các phương thức `findAll(Specification)`.

## 12. Sample project
Tạo một bộ lọc tìm kiếm xe cũ với các tiêu chí: Hãng xe, Giá từ - đến, Màu sắc, và Năm sản xuất. Các tiêu chí này có thể có hoặc không. Sử dụng Specification để xây dựng câu query động.

## 13. Interview
### Core Q&A
1. **Q**: `Specification` hoạt động dựa trên thư viện nào bên dưới?
   **A**: Nó hoạt động dựa trên **JPA Criteria API**.
2. **Q**: Làm thế nào để kết hợp nhiều Specification?
   **A**: Sử dụng các phương thức tĩnh `where()`, `and()`, `or()`, và `not()`.
3. **Q**: `JpaSpecificationExecutor` cung cấp những phương thức nào?
   **A**: Nó cung cấp các phương thức như `findOne(Spec)`, `findAll(Spec)`, `findAll(Spec, Pageable)`, `count(Spec)`.

### Scenario
**Tình huống**: Bạn muốn thực hiện một câu query lấy các User có tuổi > 18 VÀ (có tên là "Admin" HOẶC có quyền "SUPER_USER"). Bạn viết Specification như thế nào?
**Trả lời**:
```java
Specification.where(isAdult())
             .and(hasName("Admin").or(hasRole("SUPER_USER")))
```

## 14. References
- Official Docs: [Spring Data JPA Specifications](https://docs.spring.io/spring-data/jpa/docs/current/reference/html/#specifications)

## 15. Real-world Code
- Được dùng rất nhiều trong các hệ thống quản trị (Back-office) có bộ lọc tìm kiếm phức tạp.

## 16. Community
- Stack Overflow: Tag [spring-data-jpa-specs]
- Blog: "Advanced Spring Data JPA Specifications" (Baeldung).
