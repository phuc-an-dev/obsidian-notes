---
created: 2026-05-06
tags:
  - "#type/concept"
  - "#status/draft"
  - "#lang/spring"
  - "#topic/performance"
related:
  - "[[Transitive Dependency in Spring Boot.md]]"
  - "[[Dependency Conflict in Spring Boot.md]]"
  - "[[mvn clean package in Spring Boot.md]]"
---

## 1. What
Dependency Tree (Cây phụ thuộc) là một biểu đồ phân cấp hiển thị toàn bộ các thư viện (dependencies) mà một dự án đang sử dụng, bao gồm cả các phụ thuộc trực tiếp (Direct) và các phụ thuộc bắc cầu (Transitive). Trong Maven, cây này được tạo ra bằng lệnh `mvn dependency:tree`.

## 2. Why
Trong các dự án Spring Boot, một Starter đơn lẻ có thể kéo theo hàng chục thư viện khác. Nếu không có Dependency Tree, lập trình viên sẽ:
- Không biết chính xác phiên bản nào của thư viện đang được nạp vào classpath.
- Khó khăn trong việc tìm ra nguồn gốc của các thư viện gây xung đột (Dependency Conflict).
- Không thể tối ưu hóa kích thước ứng dụng bằng cách loại bỏ các thư viện thừa.

## 3. Mental Model
Hãy tưởng tượng Dependency Tree giống như một **"Gia phả dòng họ"**:
- Bạn là node gốc (Root).
- Cha mẹ bạn là Direct Dependencies.
- Ông bà, tổ tiên là Transitive Dependencies.
- Nhìn vào gia phả, bạn biết chính xác mình thừa hưởng "gen" (code/logic) từ ai và nếu có hai người trùng tên (xung đột phiên bản), bạn biết họ đến từ nhánh nào.

## 4. Where it fits
Vị trí trong quy trình phát triển:
`Build Tool -> Resolution -> Dependency Tree -> Classpath -> Runtime`

## 5. When to use
- Khi gặp lỗi `NoSuchMethodError` hoặc `ClassNotFoundException` (dấu hiệu của sai phiên bản).
- Khi muốn kiểm tra xem một thư viện cụ thể được mang vào dự án bởi Starter nào.
- Trước khi thực hiện `exclusion` một thư viện để đảm bảo không ngắt nhầm phụ thuộc quan trọng.

## 6. When NOT to use
- Khi dự án cực kỳ nhỏ, chỉ có 1-2 phụ thuộc và không có vấn đề gì phát sinh.
- Trong môi trường runtime (lệnh này chỉ chạy ở môi trường build/development).

## 7. Trade-offs
| Pros | Cons |
|------|------|
| Hiển thị cấu trúc phân cấp cực kỳ tường minh. | Output có thể rất dài và rối mắt đối với các dự án microservices lớn. |
| Giúp xác định chính xác vị trí cần dùng `exclusion`. | Cần cài đặt Maven/Gradle trên máy để chạy lệnh. |
| Hỗ trợ export ra file để phân tích sâu hơn. | |

## 8. Alternatives
| Option | So sánh |
|--------|---------|
| IDE Dependency Analyzer | Trực quan hơn (có UI, biểu đồ tròn/lưới) nhưng đôi khi không chính xác bằng CLI. |
| `mvn dependency:analyze` | Tìm kiếm các phụ thuộc được khai báo nhưng không dùng (Unused) và ngược lại. |

## 9. How
Các lệnh phổ biến để xem cây phụ thuộc:

### Xem toàn bộ cây (Maven)
```bash
mvn dependency:tree
```

### Lọc theo một thư viện cụ thể
```bash
mvn dependency:tree -Dincludes=org.hibernate
```

### Xuất ra file để đọc
```bash
mvn dependency:tree -DoutputFile=tree.txt
```

### Xem cây phụ thuộc trong Gradle
```bash
./gradlew dependencies
```

## 10. Production concerns
### Audit Logs
Nên lưu lại output của `dependency:tree` như một phần của build artifact hoặc log CI/CD để có thể tra cứu lại cấu hình thư viện của một bản build cụ thể trong quá khứ.

### Security
Sử dụng cây phụ thuộc để làm đầu vào cho các công cụ quét lỗ hổng (Vulnerability Scanners) nhằm đảm bảo không có thư viện "đen" nào chui vào Production.

## 11. Common mistakes
- Mistake: Chỉ nhìn vào tầng 1 (Direct dependencies) mà quên các tầng sâu hơn.
  Fix: Luôn đọc kỹ các node con trong cây để thấy toàn bộ bức tranh.

- Mistake: Giả định cây phụ thuộc của Maven và Gradle giống hệt nhau.
  Fix: Mỗi công cụ có thuật toán resolution khác nhau (Maven dùng "Nearest Wins", Gradle dùng "Latest Wins").

## 12. Sample project
1. Tạo project Spring Boot Web.
2. Chạy `mvn dependency:tree`.
3. Tìm node `spring-boot-starter-logging` và liệt kê các thư viện Logback, SLF4J bên dưới nó.

## 13. Interview
### Core Q&A
1. Q: Ký hiệu `(omitted for conflict)` hoặc `(omitted for duplicate)` trong cây phụ thuộc Maven nghĩa là gì?
   A: Nghĩa là Maven đã phát hiện có nhiều đường dẫn dẫn tới cùng một thư viện nhưng với các phiên bản khác nhau, và nó đã loại bỏ phiên bản này để chọn một phiên bản khác theo cơ chế "Nearest Wins".

2. Q: Làm sao để thấy các phiên bản bị loại bỏ (conflict) trong Maven?
   A: Dùng tham số `-Dverbose` (Lưu ý: Maven 3.x đã hạn chế tính năng này, cần dùng Maven 2.x hoặc plugin thay thế).

### Scenario
"Dự án của bạn báo lỗi Class A có trong cả thư viện X và thư viện Y. Bạn dùng lệnh gì để tìm ra X và Y đến từ đâu?"
-> Trả lời: Tôi dùng `mvn dependency:tree -Dincludes=groupId:artifactId` cho Class A hoặc thư viện chứa nó để truy vết ngược lên các Direct Dependencies cha.

## 14. References
- Maven Dependency Plugin: [Usage Guide](https://maven.apache.org/plugins/maven-dependency-plugin/usage.html)
- Baeldung: [Maven Dependency Tree](https://www.baeldung.com/maven-dependency-tree)

## 15. Real-world Code
Hầu hết các file Jenkinsfile chuyên nghiệp đều có bước chạy `mvn dependency:tree` để ghi lại metadata của bản build.

## 16. Community
- Stack Overflow: Tag [maven-dependency-plugin].
- GitHub: `maven-dependency-tree` source code.
