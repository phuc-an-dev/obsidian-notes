---
created: 2026-05-07
tags:
  - "#type/concept"
  - "#status/draft"
  - "#lang/java"
  - "#topic/database"
related:
  - "[[MySQL on EC2 vs AWS RDS.md]]"
  - "[[mvn clean package in Spring Boot.md]]"
---

## 1. What
Đây là lỗi phổ biến khi nâng cấp phiên bản MySQL Connector/J (JDBC driver) lên phiên bản 8.0 trở lên. Tên lớp driver cũ `com.mysql.jdbc.Driver` đã bị khai tử (deprecated) và thay thế bằng `com.mysql.cj.jdbc.Driver`. Nếu vẫn sử dụng tên lớp cũ, ứng dụng có thể ném ra ngoại lệ `ClassNotFoundException` hoặc cảnh báo `Loading class 'com.mysql.jdbc.Driver'. This is deprecated`.

## 2. Why
MySQL Connector/J 8.0 là một bản làm lại lớn:
- **CJ** là viết tắt của **Connector/J**. Việc đưa `cj` vào package name giúp phân biệt rõ ràng kiến trúc mới.
- Hỗ trợ tốt hơn cho MySQL 8.0 (với các tính năng như X Protocol).
- Khắc phục các vấn đề về múi giờ (timezone) và hỗ trợ tốt hơn cho các chuẩn Java mới.

## 3. Mental Model
Hãy tưởng tượng driver giống như một **"Số điện thoại tổng đài"**:
- Số cũ (`com.mysql.jdbc.Driver`) đã quá tải và lỗi thời.
- Nhà mạng (Oracle/MySQL) đã chuyển sang một số mới hiện đại hơn (`com.mysql.cj.jdbc.Driver`).
- Họ vẫn để số cũ hoạt động một thời gian kèm thông báo "Vui lòng gọi số mới", nhưng đến một lúc nào đó, số cũ sẽ bị cắt hoàn toàn (ClassNotFoundException). Bạn cần cập nhật "Danh bạ" (Configuration) của mình ngay.

## 4. Where it fits
Application Configuration (`application.properties` / `pom.xml`) -> **JDBC Driver Class Name** -> JVM ClassLoader -> Database Connection.

## 5. When to use
- Luôn sử dụng `com.mysql.cj.jdbc.Driver` cho mọi dự án Java mới sử dụng MySQL 5.7 hoặc 8.0+.
- Cần cập nhật ngay khi nâng cấp dependency `mysql-connector-java` lên bản 8.x.

## 6. When NOT to use
- Khi bạn bắt buộc phải dùng các phiên bản MySQL Connector cực cũ (5.1.x trở về trước) cho các hệ thống legacy cổ xưa (không khuyến khích).

## 7. Trade-offs
| Pros (Cập nhật) | Cons (Cập nhật) |
|------|------|
| Tương thích hoàn toàn với MySQL 8.0. | Phải cấu hình thêm múi giờ (`serverTimezone`). |
| Hiệu năng và bảo mật tốt hơn. | Cần thay đổi cấu hình ở nhiều nơi (Code, Tool, Properties). |
| Không còn nhìn thấy các cảnh báo phiền phức. | |

## 8. Alternatives
| Option | So sánh |
|--------|---------|
| MariaDB Driver | Có thể kết nối tới MySQL, hiệu năng cao, đôi khi nhẹ hơn. |
| HikariCP | Không phải driver nhưng là connection pool, tự động xử lý tốt các driver hiện đại. |

## 9. How
### Cách sửa trong Spring Boot (`application.properties`)
```properties
# Sửa tên driver class
spring.datasource.driver-class-name=com.mysql.cj.jdbc.Driver

# QUAN TRỌNG: Với driver mới, bạn thường cần chỉ định serverTimezone trong URL
spring.datasource.url=jdbc:mysql://localhost:3306/db_name?serverTimezone=UTC&useSSL=false
```

### Cách kiểm tra dependency trong `pom.xml`
```xml
<dependency>
    <groupId>com.mysql</groupId>
    <artifactId>mysql-connector-j</artifactId>
    <version>8.2.0</version>
</dependency>
```
*Lưu ý: Artifact ID mới là `mysql-connector-j` thay vì `mysql-connector-java` từ bản 8.0.31.*

## 10. Production concerns
### Timezone Error
Lỗi phổ biến nhất khi chuyển sang driver mới là: `The server time zone value '...' is unrecognized`. 
**Fix**: Luôn thêm `?serverTimezone=UTC` (hoặc múi giờ của bạn) vào chuỗi kết nối JDBC.

### SSL Requirement
Driver mới mặc định có thể yêu cầu SSL. Nếu server không hỗ trợ, bạn cần thêm `useSSL=false`.

## 11. Common mistakes
- Mistake: Chỉ đổi version trong `pom.xml` mà quên đổi `driver-class-name` trong cấu hình.
  Fix: Luôn kiểm tra cả hai nơi.
- Mistake: Quên tham số `serverTimezone`, khiến ứng dụng không thể khởi động.
  Fix: Coi `serverTimezone=UTC` là tham số bắt buộc.

## 12. Sample project
Thực hiện nâng cấp một dự án Legacy Spring Boot 1.5 lên 3.0:
1. Cập nhật Parent Starter.
2. Đổi dependency sang `mysql-connector-j`.
3. Sửa file `yml` từ `com.mysql.jdbc.Driver` sang `com.mysql.cj.jdbc.Driver`.
4. Cập nhật JDBC URL để bao gồm các tham số an toàn mới.

## 13. Interview
### Core Q&A
1. Q: Tại sao driver MySQL mới lại yêu cầu `serverTimezone`?
   A: Để đảm bảo việc chuyển đổi dữ liệu kiểu DateTime giữa Java và MySQL luôn chính xác, tránh các sai lệch do chênh lệch múi giờ giữa máy chủ ứng dụng và máy chủ database.

2. Q: `com.mysql.cj.jdbc.Driver` có dùng được cho MySQL 5.7 không?
   A: Có, nó tương thích ngược tốt với MySQL 5.7.

### Scenario
"Ứng dụng của bạn đang chạy bình thường, sau khi gõ `./mvnw clean package` và deploy bản mới thì bị lỗi 'com.mysql.jdbc.Driver not found'. Bạn nghi ngờ gì?"
-> Trả lời: Tôi nghi ngờ file `pom.xml` đã nâng cấp phiên bản driver lên 8.x (có thể do dùng bản starter mới) nhưng cấu hình trong file `properties` vẫn đang trỏ về class cũ. Tôi sẽ kiểm tra file cấu hình và cập nhật tên class driver thành `com.mysql.cj.jdbc.Driver`.

## 14. References
- MySQL Release Notes: [Connector/J 8.0 Changes](https://dev.mysql.com/doc/connector-j/8.0/en/connector-j-versions-migration.html)
- Baeldung: [Configuring MySQL with Spring Boot](https://www.baeldung.com/spring-boot-connect-to-mysql)

## 15. Real-world Code
Nghiên cứu cách Spring Boot Autoconfiguration tự động tìm kiếm driver: `DataSourceAutoConfiguration.java`.

## 16. Community
- Stack Overflow: [Loading class 'com.mysql.jdbc.Driver'. This is deprecated.](https://stackoverflow.com/questions/48248832/)
- Reddit: r/java - Thảo luận về việc đổi tên package của MySQL Connector.
