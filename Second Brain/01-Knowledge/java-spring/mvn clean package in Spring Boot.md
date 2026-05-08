---
created: 2026-05-06
tags:
  - "#type/concept"
  - "#status/draft"
  - "#lang/spring"
  - "#topic/performance"
related:
  - "[[Nginx Configuration for Spring Boot in EC2.md]]"
  - "[[mvnw.md]]"
  - "[[maven-compiler-plugin with annotationProcessorPaths.md]]"
  - "[[@SpringBootTest in Spring Boot.md]]"
  - "[[MySQL JDBC Driver Deprecation.md]]"
---

## 1. What
`mvn clean package` là một lệnh trong Apache Maven được sử dụng để xây dựng (build) một ứng dụng Java, cụ thể là Spring Boot. Lệnh này thực hiện hai nhiệm vụ chính: dọn dẹp các tệp tin đã được build trước đó (`clean`) và đóng gói mã nguồn đã biên dịch thành một tệp tin thực thi (`package`), thường là định dạng `.jar` hoặc `.war`.

## 2. Why
Trước khi triển khai lên server (deployment), ứng dụng cần được đóng gói thành một tệp tin duy nhất chứa đầy đủ mã nguồn và các thư viện phụ thuộc (dependencies). Lệnh này đảm bảo rằng:
- Không còn các tệp tin cũ gây xung đột hoặc lỗi build (`clean`).
- Toàn bộ mã nguồn được biên dịch và kiểm thử (nếu không skip test).
- Tạo ra một Fat JAR (Uber JAR) chứa cả ứng dụng và embedded server (Tomcat/Netty) để có thể chạy độc lập.

## 3. Mental Model
Hãy tưởng tượng `mvn clean package` giống như quy trình **"Dọn dẹp và đóng gói kiện hàng"** trước khi gửi đi:
- **Clean**: Bạn dọn dẹp bàn làm việc, vứt bỏ những mảnh vụn, tài liệu cũ của lần đóng gói trước.
- **Package**: Bạn gom tất cả các bộ phận của sản phẩm, sách hướng dẫn (libraries), cho vào một chiếc hộp duy nhất, dán băng keo và dán nhãn (version) để sẵn sàng mang ra bưu điện (server).

## 4. Where it fits
Vị trí trong luồng CI/CD:
`Code -> git push -> Checkout -> mvn clean package -> Artifact (.jar) -> Docker Build/Deploy -> Server`

## 5. When to use
- Khi chuẩn bị bàn giao sản phẩm để triển khai lên môi trường Staging/Production.
- Khi muốn kiểm tra xem toàn bộ dự án có build thành công và vượt qua các unit test hay không.
- Trong các script của GitHub Actions, Jenkins hoặc GitLab CI.

## 6. When NOT to use
- Khi bạn đang trong quá trình code và chỉ muốn chạy ứng dụng nhanh trên local (nên dùng `mvn spring-boot:run` hoặc nút Run trong IDE).
- Khi bạn chỉ thay đổi một file nhỏ và không muốn tốn thời gian chạy lại toàn bộ quy trình đóng gói và test.

## 7. Trade-offs
| Pros | Cons |
|------|------|
| Đảm bảo môi trường build sạch sẽ, không lỗi cũ. | Thời gian build có thể lâu nếu dự án lớn và nhiều test. |
| Tạo ra artifact ổn định để triển khai. | Tiêu tốn tài nguyên CPU/RAM khi thực hiện đóng gói. |
| Tự động chạy Unit Test để đảm bảo chất lượng. | |

## 8. Alternatives
| Option | So sánh |
|--------|---------|
| `mvn install` | Giống package nhưng còn copy tệp .jar vào local repository của máy (thư mục .m2). |
| `gradle build` | Công cụ build khác (Gradle), nhanh hơn nhờ cơ chế incremental build nhưng cấu hình khác. |

## 9. How
Cú pháp phổ biến khi deploy:

```bash
# Build cơ bản
mvn clean package

# Build bỏ qua Unit Test (để tăng tốc độ)
mvn clean package -DskipTests

# Build với Profile cụ thể (ví dụ Production)
mvn clean package -Pprod

# Build và chỉ định file cấu hình settings riêng
mvn clean package -s settings.xml
```

Sau khi chạy xong, tệp tin sẽ nằm trong thư mục `target/` của dự án.

## 10. Production concerns
### Fat JAR Size
Spring Boot tạo ra Fat JAR chứa toàn bộ dependencies, dẫn đến kích thước file lớn (vài chục đến vài trăm MB). Cần quản lý băng thông khi truyền tải file này lên server.

### Build Profiles
Sử dụng Maven Profiles kết hợp với Spring Profiles để đóng gói các cấu hình (database, credentials) phù hợp cho Production.

### Test Failures
Nếu một test case bị fail, quy trình `package` sẽ dừng lại. Điều này rất tốt cho Production vì nó ngăn chặn việc deploy code lỗi.

## 11. Common mistakes
- Mistake: Không dùng `clean`, dẫn đến việc file JAR mới vẫn chứa các resource cũ đã bị xóa.
  Fix: Luôn sử dụng combo `clean package`.

- Mistake: Lạm dụng `-DskipTests` trên môi trường CI Production.
  Fix: Chỉ skip test ở máy local khi cần nhanh, trên server deploy tuyệt đối phải chạy test.

## 12. Sample project
Thiết lập một GitHub Action workflow:
1. Setup JDK 17.
2. Chạy `mvn clean package`.
3. Lưu trữ (Upload artifact) file `.jar` từ thư mục `target/` để dùng cho các step deploy sau.

## 13. Interview
### Core Q&A
1. Q: Sự khác biệt giữa `mvn package` và `mvn install` là gì?
   A: Cả hai đều build và đóng gói. Nhưng `install` sẽ thêm một bước là copy file đã đóng gói vào thư mục `.m2` của máy local để các dự án khác trên cùng máy đó có thể sử dụng như một dependency.

2. Q: Tại sao Spring Boot lại tạo ra file JAR có thể chạy được (`java -jar`)?
   A: Nhờ `spring-boot-maven-plugin`. Plugin này đóng gói các thư viện phụ thuộc vào bên trong file JAR và thêm một `Launcher` để khởi động embedded web server.

### Scenario
"Build của bạn bị treo ở bước chạy test trong 20 phút. Bạn xử lý thế nào?"
-> Trả lời:
1. Kiểm tra log để xem test case nào đang bị treo (thường do database connection hoặc infinite loop).
2. Tạm thời dùng `-DskipTests` để kiểm tra xem quá trình đóng gói có lỗi không.
3. Sử dụng `mvn clean package -T 4` (nếu Maven hỗ trợ) để build đa luồng nhằm tăng tốc.

## 14. References
- Maven Lifecycle: [Introduction to the Build Lifecycle](https://maven.apache.org/guides/introduction/introduction-to-the-lifecycle.html)
- Spring Boot Maven Plugin: [Official Documentation](https://docs.spring.io/spring-boot/docs/current/maven-plugin/reference/htmlsingle/)

## 15. Real-world Code
Kiểm tra file `pom.xml`, mục `<build><plugins>` để thấy cách `spring-boot-maven-plugin` được cấu hình để tạo ra executable JAR.

## 16. Community
- Apache Maven User List.
- Stack Overflow: Tag [maven] [spring-boot].
- Blog: Baeldung (Maven guides).
