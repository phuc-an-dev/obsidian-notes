---
created: 2026-05-06
tags:
  - "#type/concept"
  - "#status/draft"
  - "#lang/spring"
  - "#topic/performance"
related:
  - "[[BOM in Spring Boot.md]]"
  - "[[Dependency Conflict in Spring Boot.md]]"
---

## 1. What
`dependencyManagement` là một phần trong file `pom.xml` của Maven dùng để tập trung quản lý thông tin phiên bản của các phụ thuộc. Nó không trực tiếp thêm thư viện vào classpath mà chỉ đóng vai trò "áp đặt" một phiên bản cụ thể nếu thư viện đó được sử dụng ở bất kỳ đâu trong dự án (kể cả bắc cầu).

## 2. Why
Trong các dự án phức tạp, một thư viện có thể bị kéo vào bởi nhiều nguồn khác nhau với các phiên bản khác nhau. `dependencyManagement` giúp:
- **Force Version**: Ép buộc toàn bộ dự án (bao gồm các module con) phải dùng duy nhất một phiên bản đã định nghĩa, bất kể thư viện cha yêu cầu gì.
- **Tránh lặp lại**: Không cần khai báo `<version>` ở từng block `<dependency>` cụ thể, giúp file POM gọn gàng và dễ cập nhật.

## 3. Mental Model
Hãy tưởng tượng `dependencyManagement` giống như một **"Quy định của Chính phủ"**:
- Các block `<dependency>` thông thường là các "Hợp đồng dân sự" tự do thỏa thuận phiên bản.
- Khi Chính phủ ban hành quy định (`dependencyManagement`), mọi hợp đồng dân sự liên quan đến mặt hàng đó (thư viện đó) đều phải tuân theo khung giá (phiên bản) mà Chính phủ đã ấn định, không được tự ý dùng bản khác.

## 4. Where it fits
Thứ tự ưu tiên trong Maven:
`Direct Dependency Version > dependencyManagement Version > Transitive Dependency Version`

## 5. When to use
- Khi muốn cập nhật phiên bản của một thư viện bắc cầu (Transitive) bị lỗi bảo mật mà không thể sửa trực tiếp ở thư viện cha.
- Khi quản lý dự án Multi-module (Parent-Child) để đảm bảo tính đồng nhất phiên bản trên toàn hệ thống.
- Khi sử dụng các BOM (Bill of Materials) như Spring Boot hoặc Spring Cloud.

## 6. When NOT to use
- Khi bạn muốn để Maven tự động quyết định phiên bản tối ưu nhất theo cơ chế "Nearest Wins".
- Đối với các phụ thuộc chỉ dùng duy nhất một lần ở một module và không có nguy cơ xung đột.

## 7. Trade-offs
| Pros | Cons |
|------|------|
| Kiểm soát tuyệt đối phiên bản thư viện. | Có thể gây ra lỗi không tương thích nếu ép dùng bản quá mới hoặc quá cũ mà thư viện cha không hỗ trợ. |
| Giảm thiểu rủi ro "Jar Hell". | Làm file POM cha trở nên dài hơn. |
| Dễ dàng nâng cấp toàn bộ hệ thống bằng cách sửa một dòng duy nhất. | |

## 8. Alternatives
| Option | So sánh |
|--------|---------|
| `<exclusions>` | Loại bỏ hẳn thư viện con, trong khi `dependencyManagement` vẫn giữ thư viện nhưng đổi version. |
| Properties | Chỉ là đặt tên biến cho version, không có tính ép buộc mạnh mẽ như `dependencyManagement`. |

## 9. How
Cách ép phiên bản cho một thư viện bắc cầu:

```xml
<project>
    <!-- 1. Định nghĩa phiên bản áp đặt -->
    <dependencyManagement>
        <dependencies>
            <dependency>
                <groupId>org.yaml</groupId>
                <artifactId>snakeyaml</artifactId>
                <version>2.0</version> <!-- Ép buộc dùng bản 2.0 dù Spring Boot có thể đòi bản 1.x -->
            </dependency>
        </dependencies>
    </dependencyManagement>

    <!-- 2. Khai báo sử dụng (không cần ghi version) -->
    <dependencies>
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter</artifactId>
            <!-- snakeyaml bên trong starter này sẽ tự động bị đổi thành 2.0 -->
        </dependency>
    </dependencies>
</project>
```

## 10. Production concerns
### Regression Testing
Khi ép một phiên bản mới (đặc biệt là Major version), bắt buộc phải chạy lại toàn bộ Regression Tests vì thư viện cha có thể gọi đến các method không còn tồn tại trong bản mới.

### Security Patches
Đây là cách nhanh nhất để vá lỗ hổng bảo mật (CVE) khi nhà phát triển thư viện cha chưa kịp ra bản cập nhật.

## 11. Common mistakes
- Mistake: Khai báo thư viện trong `dependencyManagement` nhưng quên không khai báo trong `dependencies`. Kết quả là thư viện không bao giờ được tải về.
  Fix: Nhớ rằng `dependencyManagement` chỉ là "lời hứa" về phiên bản, không phải lệnh "tải về".

- Mistake: Đặt `dependencyManagement` ở file POM con thay vì POM cha trong dự án đa module.

## 12. Sample project
Dự án Spring Boot 2.x mặc định dùng `SnakeYAML 1.x`. Hãy dùng `dependencyManagement` để ép dự án dùng `SnakeYAML 2.x` nhằm vá lỗi bảo mật, sau đó chạy `mvn dependency:tree` để kiểm chứng.

## 13. Interview
### Core Q&A
1. Q: `dependencyManagement` có làm tăng kích thước file JAR không?
   A: Không. Nó chỉ là thông tin cấu hình cho Maven Resolve, nó không thêm bất kỳ thư viện nào nếu thư viện đó không được gọi ở phần `dependencies`.

2. Q: Tại sao dùng Spring Boot lại hiếm khi phải ghi `<version>` trong file POM?
   A: Vì Spring Boot Parent POM đã định nghĩa sẵn hàng trăm thư viện phổ biến trong `dependencyManagement`.

### Scenario
"Thư viện A phụ thuộc B(v1.0), thư viện C phụ thuộc B(v2.0). Maven chọn v1.0 nhưng bạn muốn dùng v2.0. Bạn làm thế nào?"
-> Trả lời: Tôi sẽ khai báo B(v2.0) vào trong block `dependencyManagement`. Điều này sẽ override cơ chế resolution mặc định và ép Maven chọn v2.0 cho cả A và C.

## 14. References
- Maven Guide: [Introduction to the Dependency Management](https://maven.apache.org/guides/introduction/introduction-to-dependency-mechanism.html#Dependency_Management)
- Spring Boot: [Overriding Managed Versions](https://docs.spring.io/spring-boot/docs/current/reference/html/using.html#using.build-systems.maven.parent-pom.overriding-versions)

## 15. Real-world Code
Xem file `spring-boot-dependencies-x.y.z.pom` trên Maven Central để thấy cách đội ngũ Spring quản lý hàng ngàn phiên bản thư viện.

## 16. Community
- Maven User List.
- Stack Overflow: Tag [dependencymanagement].
