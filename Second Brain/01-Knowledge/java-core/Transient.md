---
created: 2026-04-23
tags:
  - "#type/concept"
  - "#status/draft"
  - "#lang/java"
  - "#topic/performance"
related:
  - "[[Record.md]]"
---

## 1. What
Trong hệ sinh thái Java, "Transient" mang ý nghĩa là "tạm thời" hoặc "không tồn tại lâu dài". Nó xuất hiện dưới hai hình thức chính:
- **Từ khóa `transient`**: Ngăn cản một field của object được serialized (chuyển thành byte stream).
- **Annotation `@Transient` (JPA/Hibernate)**: Ngăn cản một field của class được lưu trữ (persist) vào database.

## 2. Why
Nhu cầu loại bỏ dữ liệu "tạm thời" xuất hiện trong các trường hợp:
- **Bảo mật**: Không muốn lưu mật khẩu hoặc mã PIN vào file/DB.
- **Dữ liệu phái sinh (Derived data)**: Những field có thể tính toán lại từ các field khác (ví dụ: `age` tính từ `dateOfBirth`).
- **Tài nguyên không thể serialize**: Các đối tượng như `Thread`, `Socket`, `InputStream` không thể được đóng gói và gửi qua mạng.
- **Tối ưu bộ nhớ/băng thông**: Giảm kích thước file serialized hoặc payload gửi đi.

## 3. Mental Model
Hãy tưởng tượng bạn đang dán nhãn **"Đừng ghi lại"** lên một món đồ khi đóng gói hành lý. 
- Khi bạn chuyển nhà (Serialization/Persistence), bạn sẽ bỏ qua tất cả những món đồ có nhãn này. 
- Khi đến nhà mới (Deserialization/Load from DB), bạn vẫn có chỗ cho món đồ đó, nhưng nó sẽ trống rỗng (null hoặc giá trị mặc định) và bạn phải tự đi mua mới hoặc tạo lại nó.

## 4. Where it fits
- `transient` keyword: Nằm ở tầng **Core Java / JVM**, ảnh hưởng đến cơ chế `ObjectOutputStream` và `ObjectInputStream`.
- `@Transient`: Nằm ở tầng **Data Access (JPA/Hibernate)**, ảnh hưởng đến quá trình mapping Object-Relational (ORM).

## 5. When to use
- Dùng `transient` khi làm việc với `Serializable` interface và muốn bỏ qua các sensitive fields (password, token) hoặc logging objects.
- Dùng `@Transient` trong Entity khi bạn cần một field để tính toán logic trên UI nhưng không có cột tương ứng trong bảng database.

## 6. When NOT to use
- Không dùng nếu dữ liệu đó là bắt buộc để khôi phục trạng thái object (state consistency).
- Không lạm dụng để giấu dữ liệu "thừa" thay vì thiết kế DTO (Data Transfer Object) chuẩn mực.

## 7. Trade-offs
| Pros | Cons |
|------|------|
| Giảm kích thước data truyền tải/lưu trữ. | Object sau khi phục hồi sẽ có giá trị mặc định (null, 0, false). |
| Tăng tính bảo mật (masking data). | Cần logic bổ sung để tái tạo giá trị sau khi deserialization. |
| Ngăn lỗi `NotSerializableException`. | Dễ gây nhầm lẫn giữa keyword và annotation nếu không nắm vững. |

## 8. Alternatives
| Option | So sánh |
|--------|---------|
| `static` keyword | Field static cũng không được serialized, nhưng nó thuộc về class, không phải instance. |
| Jackson `@JsonIgnore` | Chỉ bỏ qua khi chuyển sang JSON (thường dùng trong REST API). |
| DTO Pattern | Tạo object riêng chỉ chứa dữ liệu cần thiết, giải pháp triệt để hơn. |

## 9. How
### Java Keyword
```java
public class User implements Serializable {
    private String username;
    private transient String password; // Sẽ không được serialized

    // Sau khi deserialize, password sẽ là null
}
```

### JPA Annotation
```java
@Entity
public class Employee {
    @Id
    private Long id;
    private LocalDate dateOfBirth;

    @Transient
    private int age; // Không có cột 'age' trong DB

    @PostLoad
    public void calculateAge() {
        this.age = Period.between(dateOfBirth, LocalDate.now()).getYears();
    }
}
```

## 10. Production concerns
### Scaling
Việc sử dụng `transient` giúp giảm payload trong Distributed Caching (như Redis) hoặc khi truyền message qua Kafka/RabbitMQ nếu dùng cơ chế serialization mặc định của Java.

### Failure
Sau khi deserialize, nếu logic nghiệp vụ quên kiểm tra hoặc khởi tạo lại các field `transient`, hệ thống có thể gặp `NullPointerException`.

### Monitoring
Theo dõi kích thước object sau serialization bằng các tool như JOL (Java Object Layout) để thấy rõ sự khác biệt khi dùng và không dùng `transient`.

## 11. Common mistakes
- Mistake: Nghĩ rằng `@Transient` của JPA cũng sẽ làm cho field đó `transient` khi serialize qua mạng.
  Fix: Hai thứ này độc lập. Một field có `@Transient` vẫn sẽ được serialized bình thường trừ khi có thêm từ khóa `transient` hoặc `@JsonIgnore`.

- Mistake: Quên khởi tạo lại giá trị cho field `transient` sau khi object được phục hồi.
  Fix: Sử dụng method `readObject()` (cho Java Serialization) hoặc `@PostLoad` (cho JPA) để tính toán lại giá trị.

## 12. Sample project
Tạo một hệ thống Session Management:
- Lưu session vào file/database.
- `sessionToken` phải là `transient` để đảm bảo mỗi lần load lại session, user phải re-authenticate hoặc token phải được fetch lại từ secure vault thay vì lưu trực tiếp.

## 13. Interview
### Core Q&A
1. Q: Sự khác biệt giữa `transient` và `volatile`?
   A: `transient` dùng để quản lý serialization (bỏ qua field). `volatile` dùng trong đa luồng (multi-threading) để đảm bảo tính hiển thị (visibility) của biến giữa các thread. Chúng hoàn toàn không liên quan đến nhau.

2. Q: Một biến `static` có thể là `transient` không?
   A: Có thể viết `public static transient int x`, nhưng `transient` sẽ không có tác dụng. Biến `static` vốn dĩ không thuộc về instance nên nó không bao giờ được serialized theo object state.

3. Q: `@Transient` của JPA nằm trong package nào?
   A: `jakarta.persistence.Transient` (trước đây là `javax.persistence.Transient`). Đừng nhầm với các annotation cùng tên của các framework khác.

### Scenario
Tình huống: Bạn có một class `Order` chứa danh sách `OrderItem`. Bạn muốn tính tổng tiền `totalPrice` và hiển thị nó ở mọi nơi, nhưng DBA không cho phép thêm cột `total_price` vào database để tránh redundancy. Bạn làm gì?
Giải pháp: Khai báo field `totalPrice` trong class `Order`, đánh dấu nó bằng `@Transient`. Sử dụng callback method `@PostLoad` hoặc logic trong getter để tính tổng từ list `OrderItem` mỗi khi object được load lên.

## 14. References
- Java Specs (Serialization): https://docs.oracle.com/javase/8/docs/platform/serialization/spec/serial-arch.html
- Hibernate User Guide (Basic Mapping): https://docs.jboss.org/hibernate/orm/current/userguide/html_single/Hibernate_User_Guide.html#mapping-model-introduction

## 15. Real-world Code
- `java.util.ArrayList`: Field `elementData` được đánh dấu `transient` để tự quản lý việc serialization thủ công nhằm tối ưu performance (chỉ serialize các phần tử thực có trong list thay vì toàn bộ array capacity).

## 16. Community
- Baeldung: Java transient keyword.
- Stack Overflow: Difference between @Transient and transient keyword.
