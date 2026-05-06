---
created: 2026-05-06
tags:
  - "#type/concept"
  - "#status/draft"
  - "#lang/spring"
  - "#topic/performance"
related:
  - "[[Dependency Tree in Spring Boot.md]]"
  - "[[Force Dependency Version in Spring Boot.md]]"
  - "[[Transitive Dependency in Spring Boot.md]]"
---

## 1. What
Dependency Conflict (Xung đột phụ thuộc) là tình trạng xảy ra khi một dự án vô tình nạp vào hai hoặc nhiều phiên bản khác nhau của cùng một thư viện (cùng GroupID và ArtifactID) thông qua các đường dẫn phụ thuộc khác nhau. Điều này dẫn đến sự không nhất quán và gây lỗi khi ứng dụng thực thi.

## 2. Why
Trong các ứng dụng lớn, việc sử dụng nhiều thư viện bên thứ ba là không thể tránh khỏi. Mỗi thư viện lại có các phụ thuộc riêng (Transitive Dependencies). Xung đột xảy ra khi:
- Thư viện A yêu cầu `Jackson v2.10`.
- Thư viện B yêu cầu `Jackson v2.15`.
Maven/Gradle chỉ có thể chọn một phiên bản duy nhất để đưa vào classpath, và phiên bản bị loại bỏ có thể chứa các class hoặc method mà thư viện yêu cầu nó đang cần, dẫn tới lỗi đổ vỡ.

## 3. Mental Model
Hãy tưởng tượng Dependency Conflict giống như một **"Cuộc tranh cãi về phiên bản linh kiện"**:
- Bạn đang lắp ráp một chiếc xe máy.
- Hệ thống phanh (Thư viện A) yêu cầu con ốc 10mm.
- Hệ thống bánh xe (Thư viện B) yêu cầu con ốc 12mm.
- Tuy nhiên, trên lỗ cắm của xe (Classpath) chỉ có chỗ cho đúng 1 con ốc.
- Nếu bạn chọn ốc 10mm, bánh xe sẽ bị lỏng. Nếu chọn 12mm, phanh sẽ không lắp vừa. Đó chính là xung đột.

## 4. Where it fits
Giai đoạn xảy ra:
`Build Time (Resolution) -> Conflict Detected -> Build Tool Selection -> Runtime Error (nếu chọn sai)`

## 5. When to use
- Khi gặp các lỗi Runtime phổ biến: `java.lang.NoSuchMethodError`, `java.lang.ClassNotFoundException`, `java.lang.AbstractMethodError`.
- Khi ứng dụng hoạt động không đúng kỳ vọng sau khi thêm một thư viện mới.

## 6. When NOT to use
- (Không áp dụng - Xung đột là vấn đề cần tránh hoặc giải quyết, không phải một tính năng để "dùng").

## 7. Trade-offs
| Resolution Strategy | Pros | Cons |
|---------------------|------|------|
| **Nearest Wins (Maven)** | Đơn giản, dễ đoán dựa trên cây phụ thuộc. | Thư viện ở xa gốc hơn dễ bị "hy sinh" dù nó có thể cần bản mới hơn. |
| **Latest Wins (Gradle)** | Luôn dùng bản mới nhất (thường tốt hơn). | Có thể gây lỗi nếu bản mới xóa bỏ các tính năng cũ (Breaking changes). |

## 8. Alternatives
| Option | So sánh |
|--------|---------|
| Shading (Relocation) | Đổi tên package của thư viện bị xung đột để dùng cả hai bản cùng lúc (Phức tạp, làm tăng dung lượng JAR). |
| OSGi | Cho phép chạy nhiều phiên bản thư viện trong các classloader tách biệt (Cực kỳ phức tạp). |

## 9. How
Quy trình giải quyết xung đột:

1. **Phát hiện**: Chạy lệnh xem cây phụ thuộc.
   `mvn dependency:tree -Dverbose` (để thấy các bản bị omitted).
2. **Phân tích**: Tìm xem phiên bản nào đang thắng và nó đến từ đâu.
3. **Xử lý**:
   - Cách 1: Dùng `<exclusion>` ở thư viện mang bản sai vào.
   - Cách 2: Khai báo trực tiếp phiên bản đúng trong phần `dependencies` của dự án (vì Direct Dependency thắng Transitive).
   - Cách 3: Dùng `dependencyManagement` để ép phiên bản trên toàn dự án.

## 10. Production concerns
### Silent Conflicts
Nguy hiểm nhất là các xung đột không gây lỗi ngay khi khởi động mà chỉ gây lỗi khi ứng dụng chạy vào một logic cụ thể (vd: khi gọi đến một hàm hiếm dùng của thư viện bị lệch version).

### Binary Compatibility
Luôn kiểm tra tính tương thích nhị phân (Binary Compatibility) giữa phiên bản bạn chọn và các thư viện phụ thuộc vào nó.

## 11. Common mistakes
- Mistake: Thêm bừa bãi các thư viện vào file POM mà không kiểm tra cây phụ thuộc.
  Fix: Chạy `mvn dependency:tree` sau mỗi lần thêm thư viện lớn.

- Mistake: Nghĩ rằng version mới nhất luôn là tốt nhất.
  Fix: Version tốt nhất là version mà tất cả các thành phần trong hệ thống đều đồng thuận sử dụng.

## 12. Sample project
1. Thêm `spring-boot-starter-web`.
2. Thêm một thư viện cũ (vd: một bản SDK rất cũ) mà yêu cầu phiên bản Jackson lạc hậu.
3. Quan sát lỗi `NoSuchMethodError` khi Spring cố gắng serialize JSON.
4. Dùng `exclusion` để sửa lỗi.

## 13. Interview
### Core Q&A
1. Q: Maven giải quyết xung đột như thế nào?
   A: Sử dụng cơ chế "Nearest Wins" (Đường dẫn nào ngắn nhất tới node gốc sẽ thắng). Nếu độ dài bằng nhau, cái nào khai báo trước sẽ thắng.

2. Q: Tại sao xung đột phụ thuộc lại dẫn đến lỗi `NoSuchMethodError`?
   A: Vì tại thời điểm Compile, Java thấy method đó (trong bản build). Nhưng tại thời điểm Runtime, Classloader nạp bản thư viện khác (do xung đột) không có method đó, dẫn đến crash.

### Scenario
"Sau khi thêm thư viện AWS SDK, ứng dụng của bạn không khởi động được do lỗi trùng lặp thư viện logging. Bạn làm gì?"
-> Trả lời:
1. Chạy `mvn dependency:tree` để tìm các nhánh mang thư viện logging vào.
2. Xác định các bản duplicate (thường là giữa Log4j và Logback).
3. Sử dụng `<exclusion>` trong phần khai báo AWS SDK để loại bỏ bộ logging của nó, nhường chỗ cho bộ logging chuẩn của Spring Boot.

## 14. References
- Maven: [Dependency Mediation](https://maven.apache.org/guides/introduction/introduction-to-dependency-mechanism.html#Dependency_Mediation)
- Cloudflare: [Dealing with dependency conflicts](https://blog.cloudflare.com/dealing-with-dependency-conflicts-in-java/)

## 15. Real-world Code
Nghiên cứu plugin `maven-enforcer-plugin` với rule `DependencyConvergence` để chủ động ngăn chặn các bản build có xung đột phụ thuộc.

## 16. Community
- Reddit: r/java.
- Stack Overflow: Tag [dependency-conflict], [jar-hell].
