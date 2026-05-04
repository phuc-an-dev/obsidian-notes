---
created: 2026-05-04
tags:
  - "#type/library"
  - "#status/draft"
  - "#lang/java"
  - "#topic/clean-code"
related:
  - "[[Record]]"
  - "[[Compact Constructor]]"
---

## 1. What
`@RequiredArgsConstructor` là một annotation của thư viện Lombok dùng để tự động tạo ra một constructor chứa các tham số cho tất cả các trường (fields) được đánh dấu là `final` hoặc các trường có ràng buộc `@NonNull`.

## 2. Why
Trong Java, việc viết constructor để thực hiện Dependency Injection (DI) hoặc khởi tạo các hằng số thường dẫn đến rất nhiều mã lặp (boilerplate code). Khi một lớp có 5-7 dependencies, constructor sẽ rất dài và khó bảo trì. Lombok giúp loại bỏ phần mã này, làm cho lớp trở nên sạch sẽ và tập trung vào logic nghiệp vụ hơn.

## 3. Mental Model
Hãy tưởng tượng `@RequiredArgsConstructor` như một **"Máy lắp ráp tự động"**. Bạn chỉ cần dán nhãn **"Bắt buộc (final)"** lên các linh kiện cần thiết của một thiết bị. Khi thiết bị đi qua dây chuyền, cái máy này sẽ tự động biết cách lắp tất cả những linh kiện đó lại với nhau để tạo ra sản phẩm hoàn chỉnh mà bạn không cần phải tự tay cầm tua-vít vặn từng con ốc.

## 4. Where it fits
Thường được dùng trong các lớp `@Service`, `@Component`, hoặc `@Controller` của Spring Boot để thực hiện Constructor Injection thay vì dùng `@Autowired` trên field.

## 5. When to use
- Khi thực hiện Constructor Injection trong Spring Framework (đây là cách được khuyến nghị thay vì Field Injection).
- Khi tạo các lớp bất biến (immutable classes) có các trường `final`.
- Khi muốn giảm bớt số lượng dòng code "rác" trong class.

## 6. When NOT to use
- Khi bạn cần thực hiện logic kiểm tra (validation) phức tạp hoặc biến đổi dữ liệu ngay trong constructor (trường hợp này nên viết constructor thủ công).
- Khi lớp không có bất kỳ trường `final` nào (annotation này sẽ tạo ra một constructor không tham số).
- Khi bạn muốn kiểm soát thứ tự các tham số trong constructor một cách đặc biệt.

## 7. Trade-offs
| Pros | Cons |
|------|------|
| Giảm đáng kể mã Boilerplate, code cực kỳ gọn. | Tạo ra mã "ẩn" (magic), người mới có thể khó hiểu dữ liệu được truyền vào đâu. |
| Khuyến khích sử dụng Constructor Injection (tốt cho unit test). | Phụ thuộc vào thư viện bên ngoài và cấu hình IDE (Lombok plugin). |
| Dễ dàng thêm/bớt dependency mà không cần sửa constructor. | Khó debug trực tiếp vào dòng code của constructor. |

## 8. Alternatives
| Option | So sánh |
|--------|---------|
| Manual Constructor | Rõ ràng nhưng tốn công viết và sửa mỗi khi thêm field. |
| `@AllArgsConstructor` | Tạo constructor cho TẤT CẢ các field, kể cả những field không phải final. |
| Java Records | (JDK 16+) Giải pháp native của Java cho các lớp chứa dữ liệu bất biến. |

## 9. How
```java
import lombok.RequiredArgsConstructor;
import org.springframework.stereotype.Service;

@Service
@RequiredArgsConstructor // Lombok tự tạo constructor cho repository
public class UserService {

    private final UserRepository userRepository; // Bắt buộc phải có trong constructor
    private final EmailService emailService;     // Bắt buộc phải có trong constructor
    
    private String optionalConfig; // KHÔNG có trong constructor vì không phải final

    public void registerUser(User user) {
        userRepository.save(user);
        emailService.sendWelcomeEmail(user);
    }
}
```

## 10. Production concerns
### Scaling
Việc sử dụng `@RequiredArgsConstructor` giúp việc thêm các service mới vào một class rất nhanh chóng, nhưng hãy cẩn thận với "Constructor Hell" (quá nhiều tham số), đây là dấu hiệu của việc vi phạm Single Responsibility Principle (SRP).

### Failure
Nếu bạn quên từ khóa `final` cho một dependency, Spring sẽ không thể inject nó qua constructor và field đó sẽ bị `null` khi chạy ứng dụng.

### Monitoring
Không có tác động trực tiếp đến monitoring, nhưng giúp code dễ đọc hơn trong quá trình review lỗi.

## 11. Common mistakes
- **Mistake**: Quên khai báo `final` cho các biến phụ thuộc.
  **Fix**: Luôn kiểm tra xem các dependency đã có `final` chưa.
  
- **Mistake**: Sử dụng `@RequiredArgsConstructor` cùng với `@Autowired` trên field.
  **Fix**: Chỉ nên chọn một cách, và Constructor Injection (với `@RequiredArgsConstructor`) là cách tốt hơn.

## 12. Sample project
Tạo một Spring Boot Service để xử lý đơn hàng, yêu cầu inject 4-5 repositories và services khác nhau. Sử dụng `@RequiredArgsConstructor` để giữ cho class ngắn gọn dưới 50 dòng code.

## 13. Interview
### Core Q&A
1. **Q**: `@RequiredArgsConstructor` tạo constructor cho những loại field nào?
   **A**: Nó chỉ tạo constructor cho các field được khai báo là `final` hoặc được đánh dấu bằng `@NonNull`.
2. **Q**: Tại sao dùng `@RequiredArgsConstructor` lại tốt cho Unit Testing?
   **A**: Vì nó tạo ra Constructor Injection, cho phép bạn dễ dàng truyền các mock objects vào lớp cần test mà không cần dùng đến Reflection hay Spring Context.
3. **Q**: Nếu một class không có field `final` nào, `@RequiredArgsConstructor` sẽ làm gì?
   **A**: Nó sẽ tạo ra một constructor mặc định không tham số (no-args constructor).

### Scenario
**Tình huống**: Bạn thêm một repository mới vào Service và ứng dụng báo lỗi `NullPointerException` khi gọi đến repository đó, mặc dù bạn đã dùng `@RequiredArgsConstructor`. Nguyên nhân là gì?
**Trả lời**: Khả năng cao nhất là tôi đã quên khai báo từ khóa `final` cho repository mới đó. Do không có `final`, Lombok không đưa nó vào constructor, và Spring không thể inject nó, dẫn đến biến đó mang giá trị `null`.

## 14. References
- Official Docs: [https://projectlombok.org/features/constructor](https://projectlombok.org/features/constructor)
- GitHub Repo: [https://github.com/projectlombok/lombok](https://github.com/projectlombok/lombok)

## 15. Real-world Code
- Hầu hết các dự án Spring Boot hiện đại đều sử dụng pattern này để thay thế cho `@Autowired` trên field.

## 16. Community
- Reddit: r/java, r/springboot
- Stack Overflow: Tag [lombok]
- Blog: "Why you should use Constructor Injection in Spring"
