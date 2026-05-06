---
created: 2026-05-06
tags:
  - "#type/concept"
  - "#status/draft"
  - "#lang/spring"
  - "#topic/performance"
related:
  - "[[mvn clean package in Spring Boot.md]]"
  - "[[spring-cloud-starter-aws.md]]"
  - "[[Dependency Tree in Spring Boot.md]]"
  - "[[Dependency Conflict in Spring Boot.md]]"
---

## 1. What
Transitive Dependency (Phụ thuộc bắc cầu) là một cơ chế trong các công cụ quản lý dự án như Maven hoặc Gradle, trong đó nếu Dự án A phụ thuộc vào Thư viện B, và Thư viện B lại phụ thuộc vào Thư viện C, thì Dự án A sẽ tự động có được Thư viện C mà không cần khai báo trực tiếp.

## 2. Why
Trong các ứng dụng hiện đại như Spring Boot, số lượng thư viện cần thiết là cực kỳ lớn. Nếu không có cơ chế bắc cầu, lập trình viên sẽ phải tự tay khai báo hàng trăm thư viện con, dẫn đến:
- **Dependency Hell**: Cực kỳ khó quản lý phiên bản và tính tương thích.
- **Sai sót**: Dễ thiếu các thư viện quan trọng khiến ứng dụng không thể khởi động.
Transitive dependency giúp quy trình khai báo trở nên gọn nhẹ (chỉ cần khai báo một "Starter") và đảm bảo tính đồng nhất.

## 3. Mental Model
Hãy tưởng tượng Transitive Dependency giống như việc **"Mời một người bạn đi tiệc"**:
- Bạn mời Anh A (Direct Dependency).
- Anh A là con của Ông B, và Anh A không thể đi đâu nếu thiếu Ông B đi cùng để lái xe (Transitive Dependency).
- Kết quả là bữa tiệc của bạn tự động có mặt cả Anh A và Ông B, dù bạn chỉ gửi thiệp mời cho mỗi Anh A.

## 4. Where it fits
Vị trí trong cấu trúc dự án:
`pom.xml / build.gradle -> Direct Dependency -> Build Tool (Resolver) -> Transitive Dependencies -> Classpath`

## 5. When to use
- Luôn luôn hiện diện khi sử dụng các "Starters" của Spring Boot (vd: `spring-boot-starter-web` mang theo Tomcat, Jackson, Hibernate Validator, v.v.).
- Khi muốn giữ file cấu hình dự án (`pom.xml`) sạch sẽ và chỉ tập trung vào các phụ thuộc cấp cao.

## 6. When NOT to use
- Khi thư viện bắc cầu gây ra xung đột phiên bản (Version Conflict) với một thư viện khác trong dự án.
- Khi thư viện bắc cầu chứa các lỗ hổng bảo mật (Vulnerabilities) đã được cảnh báo.
- Khi thư viện bắc cầu quá nặng và không thực sự cần thiết cho tính năng của ứng dụng (làm tăng kích thước Fat JAR).

## 7. Trade-offs
| Pros | Cons |
|------|------|
| Đơn giản hóa việc quản lý thư viện. | Gây khó khăn trong việc kiểm soát chính xác những gì có trong dự án. |
| Đảm bảo các thư viện con luôn tương thích với thư viện cha. | Dễ dẫn đến "phình to" ứng dụng (Project Bloat). |
| Tiết kiệm thời gian cấu hình ban đầu. | Rủi ro về bảo mật từ các thư viện "vô hình". |

## 8. Alternatives
| Option | So sánh |
|--------|---------|
| Exclusions | Loại bỏ một phụ thuộc bắc cầu cụ thể nhưng vẫn giữ thư viện cha. |
| Dependency Management | Ép buộc sử dụng một phiên bản cụ thể cho mọi phụ thuộc (kể cả bắc cầu). |
| Manual Declaration | Khai báo thủ công toàn bộ (Cực kỳ không khuyến khích). |

## 9. How
Cách kiểm tra và loại bỏ phụ thuộc bắc cầu:

### Kiểm tra bằng Maven
```bash
mvn dependency:tree
```

### Loại bỏ một thư viện cụ thể trong pom.xml
Ví dụ: Loại bỏ Tomcat mặc định để dùng Jetty:
```xml
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-web</artifactId>
    <exclusions>
        <exclusion>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-tomcat</artifactId>
        </exclusion>
    </exclusions>
</dependency>
```

## 10. Production concerns
### Security Scanning
Sử dụng các công cụ như `OWASP Dependency-Check` hoặc `Snyk` để quét các transitive dependencies trên Production nhằm phát hiện các thư viện lỗi thời có nguy cơ bị tấn công.

### Artifact Size
Trong môi trường Cloud/Docker, kích thước image rất quan trọng. Cần loại bỏ các thư viện rác (Unused transitive dependencies) để giảm thời gian build và deploy.

## 11. Common mistakes
- Mistake: Khai báo lại thư viện con với phiên bản khác thay vì dùng `exclusion`.
  Fix: Luôn dùng `exclusion` ở thư viện cha để tránh việc có 2 phiên bản khác nhau của cùng một thư viện trong classpath.

- Mistake: Không bao giờ chạy `mvn dependency:tree`.
  Fix: Chạy lệnh này định kỳ để hiểu rõ cấu trúc phụ thuộc của dự án.

## 12. Sample project
Tạo một dự án Spring Boot sử dụng `spring-boot-starter-data-jpa`. Thực hiện:
1. Chạy `mvn dependency:tree` để thấy Hibernate và HikariCP được mang vào.
2. Thử `exclude` HikariCP và quan sát lỗi ứng dụng khi khởi động.

## 13. Interview
### Core Q&A
1. Q: Làm thế nào để giải quyết xung đột khi hai thư viện cha mang vào hai phiên bản khác nhau của cùng một thư viện con?
   A: Maven sử dụng cơ chế "Nearest wins" (Thư viện nào gần node gốc hơn sẽ được chọn). Để ép buộc phiên bản, ta nên dùng phần `<dependencyManagement>`.

2. Q: Ý nghĩa của thẻ `<exclusions>` là gì?
   A: Dùng để ngắt kết nối bắc cầu của một thư viện con cụ thể, không cho nó xuất hiện trong classpath của dự án.

### Scenario
"Ứng dụng của bạn bị lỗi `NoSuchMethodError`. Bạn nghi ngờ do Transitive Dependency. Bạn sẽ làm gì?"
-> Trả lời:
1. Chạy `mvn dependency:tree -Dverbose` để tìm tất cả các phiên bản của thư viện bị nghi ngờ.
2. Xác định thư viện cha nào đang mang vào phiên bản sai.
3. Sử dụng `<exclusion>` ở thư viện cha đó hoặc khai báo trực tiếp phiên bản đúng ở đầu `pom.xml`.

## 14. References
- Maven Docs: [Introduction to the Dependency Mechanism](https://maven.apache.org/guides/introduction/introduction-to-dependency-mechanism.html)
- Spring Boot: [Managing Dependencies](https://docs.spring.io/spring-boot/docs/current/reference/html/using.html#using.build-systems.dependency-management)

## 15. Real-world Code
Hầu hết các "Cloud Starters" của Spring Cloud đều tận dụng tối đa transitive dependency để mang vào toàn bộ hệ sinh thái Netflix OSS hoặc AWS SDK.

## 16. Community
- Reddit: r/java, r/SpringBoot.
- Stack Overflow: Tag [maven-dependency], [transitive-dependency].
