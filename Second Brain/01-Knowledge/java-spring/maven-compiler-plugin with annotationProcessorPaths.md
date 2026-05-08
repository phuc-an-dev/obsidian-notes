---
created: 2026-05-07
tags:
  - "#type/concept"
  - "#status/draft"
  - "#lang/java"
  - "#topic/performance"
related:
  - "[[mvnw.md]]"
  - "[[mvn clean package in Spring Boot.md]]"
---

## 1. What
`maven-compiler-plugin` là một plugin cốt lõi của Apache Maven dùng để biên dịch mã nguồn Java. Thuộc tính `annotationProcessorPaths` là một phần cấu hình bên trong plugin này (từ phiên bản 3.5 trở lên), cho phép chỉ định danh sách các dependencies chứa các bộ xử lý annotation (Annotation Processors) sẽ được thực thi trong quá trình biên dịch.

## 2. Why
Trước đây, các thư viện như Lombok hay MapStruct thường được khai báo như các dependency thông thường trong `pom.xml`. Tuy nhiên, điều này dẫn đến việc các JAR của bộ xử lý annotation bị đóng gói vào file JAR cuối cùng (runtime), làm tăng kích thước file không cần thiết. `annotationProcessorPaths` ra đời để:
- **Tách biệt hoàn toàn**: Chỉ sử dụng các processor này lúc compile-time, không đưa vào runtime.
- **Tránh xung đột**: Giảm thiểu nguy cơ xung đột dependency giữa các processor và ứng dụng chính.
- **Tối ưu hóa**: Giúp classpath của quá trình biên dịch gọn gàng và rõ ràng hơn.

## 3. Mental Model
Hãy tưởng tượng quá trình build ứng dụng giống như việc **"Xây một ngôi nhà"**.
- `maven-compiler-plugin` là **"Đội ngũ thợ xây"**.
- `annotationProcessorPaths` là **"Kiến trúc sư tư vấn"**: Ông ấy chỉ xuất hiện lúc xây dựng để hướng dẫn thợ xây tạo ra các bản vẽ chi tiết hoặc tự động lắp ráp một số bộ phận.
- Khi ngôi nhà hoàn thành và bạn dọn vào ở (Runtime), bạn không cần ông kiến trúc sư đó ở trong nhà với mình. `annotationProcessorPaths` đảm bảo ông ấy "biến mất" đúng lúc sau khi xong việc.

## 4. Where it fits
Source Code (`.java`) -> **`maven-compiler-plugin`** -> **Annotation Processors (`annotationProcessorPaths`)** -> Generated Source / Bytecode (`.class`).

## 5. When to use
- Khi dự án sử dụng **Lombok** để tự động tạo Getter/Setter/Constructor.
- Khi sử dụng **MapStruct** để tự động tạo code chuyển đổi dữ liệu (Mapper).
- Khi sử dụng các thư viện tạo mã nguồn lúc compile-time khác như QueryDSL hoặc Hibernate Static Metamodel.

## 6. When NOT to use
- Khi dự án của bạn thuần túy Java và không sử dụng bất kỳ thư viện nào yêu cầu Annotation Processing.
- Khi bạn đang dùng phiên bản Maven Compiler Plugin quá cũ (< 3.5), lúc đó bạn vẫn phải khai báo dependency thông thường.

## 7. Trade-offs
| Pros | Cons |
|------|------|
| Giảm kích thước file JAR cuối cùng (Artifact). | Cấu hình `pom.xml` dài dòng và phức tạp hơn một chút. |
| Phân tách rõ ràng trách nhiệm của từng dependency. | Cần chú ý thứ tự khai báo nếu các processor phụ thuộc lẫn nhau. |
| Tránh rò rỉ các thư viện xử lý vào classpath của ứng dụng. | N/A |

## 1. How
Cấu hình điển hình trong `pom.xml` kết hợp cả Lombok và MapStruct:

```xml
<build>
    <plugins>
        <plugin>
            <groupId>org.apache.maven.plugins</groupId>
            <artifactId>maven-compiler-plugin</artifactId>
            <version>3.11.0</version>
            <configuration>
                <source>17</source>
                <target>17</target>
                <annotationProcessorPaths>
                    <!-- Thứ tự quan quan trọng: Lombok nên đứng trước MapStruct -->
                    <path>
                        <groupId>org.projectlombok</groupId>
                        <artifactId>lombok</artifactId>
                        <version>1.18.30</version>
                    </path>
                    <path>
                        <groupId>org.mapstruct</groupId>
                        <artifactId>mapstruct-processor</artifactId>
                        <version>1.5.5.Final</version>
                    </path>
                </annotationProcessorPaths>
            </configuration>
        </plugin>
    </plugins>
</build>
```

## 9. Production concerns
### Classpath Leaks
Nếu bạn quên dùng `annotationProcessorPaths` mà vẫn để processor trong `<dependencies>`, hãy đảm bảo chúng có `<scope>provided</scope>` để tránh bị đóng gói vào artifact.

### Compilation Performance
Việc chạy nhiều annotation processors có thể làm chậm quá trình build. Tuy nhiên, lợi ích về việc giảm thiểu boilerplate code là rất lớn.

## 10. Common mistakes
- Mistake: Quên không khai báo `lombok-mapstruct-binding` khi dùng cả hai thư viện.
  Fix: Thêm `lombok-mapstruct-binding` vào `annotationProcessorPaths` để chúng hiểu được code của nhau.

- Mistake: Khai báo sai phiên bản plugin hoặc dependency bên trong path.
  Fix: Luôn kiểm tra tính tương thích giữa phiên bản Java và phiên bản plugin.

## 11. Sample project
Tạo một project Spring Boot đơn giản:
1. Định nghĩa một Entity với Lombok `@Data`.
2. Định nghĩa một DTO.
3. Dùng MapStruct tạo interface Mapper.
4. Cấu hình `annotationProcessorPaths` và chạy `mvn clean compile` để thấy các file `.class` và mã nguồn tự động được sinh ra trong thư mục `target/generated-sources`.

## 12. Interview
### Core Q&A
1. Q: Tại sao chúng ta nên dùng `annotationProcessorPaths` thay vì khai báo dependency thông thường?
   A: Để tách biệt các thư viện chỉ phục vụ quá trình sinh code (compile-time) ra khỏi artifact cuối cùng, giúp giảm kích thước file JAR và tránh xung đột thư viện lúc runtime.

2. Q: Điều gì xảy ra nếu hai annotation processors cùng can thiệp vào một file?
   A: Thứ tự khai báo trong `annotationProcessorPaths` sẽ quyết định thứ tự thực thi. Đây là lý do Lombok thường phải đứng trước các thư viện khác để code được sinh ra bởi Lombok sẵn sàng cho các thư viện sau xử lý.

### Scenario
"Dự án của bạn nâng cấp lên Java 17 và MapStruct không còn sinh code nữa. Bạn kiểm tra gì đầu tiên?"
-> Trả lời: Tôi sẽ kiểm tra phiên bản của `maven-compiler-plugin` và `mapstruct-processor` trong `annotationProcessorPaths`. Các phiên bản cũ có thể không tương thích với Java 17 hoặc yêu cầu cấu hình `source`/`target` rõ ràng.

## 13. References
- Maven Compiler Plugin Docs: https://maven.apache.org/plugins/maven-compiler-plugin/compile-mojo.html#annotationProcessorPaths
- MapStruct Installation Guide: https://mapstruct.org/documentation/installation/

## 14. Real-world Code
Nghiên cứu file `pom.xml` của các dự án Microservices chuyên nghiệp sử dụng kiến trúc Clean Architecture hoặc Hexagonal để thấy cách họ quản lý mappers và boilerplates.

## 15. Community
- Stack Overflow: Tag [maven-compiler-plugin] [annotation-processing].
- GitHub: Repository của Lombok và MapStruct.
- Blog: "Mastering Maven Annotation Processors" - Baeldung.
