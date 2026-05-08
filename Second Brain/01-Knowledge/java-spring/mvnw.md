---
created: 2026-05-07
tags:
  - "#type/concept"
  - "#status/draft"
  - "#lang/spring"
  - "#topic/performance"
related:
  - "[[mvn clean package in Spring Boot.md]]"
  - "[[BOM in Spring Boot.md]]"
  - "[[maven-compiler-plugin with annotationProcessorPaths.md]]"
  - "[[@SpringBootTest in Spring Boot.md]]"
---

## 1. What
Maven Wrapper (tên file thực thi là `mvnw` cho Linux/macOS và `mvnw.cmd` cho Windows) là một công cụ đi kèm với dự án Java, cho phép bạn thực hiện các lệnh Maven mà không cần phải cài đặt sẵn Apache Maven trên máy tính. Nó tự động tải về phiên bản Maven đúng yêu cầu và sử dụng nó để build dự án.

## 2. Why
Trước khi có Maven Wrapper, mỗi lập trình viên hoặc server CI/CD phải tự cài đặt Maven thủ công. Điều này dẫn đến vấn đề:
- **Xung đột phiên bản**: Dự án yêu cầu Maven 3.8 nhưng máy bạn cài 3.6, dẫn đến lỗi build không mong muốn.
- **Tốn công thiết lập**: Mỗi khi có thành viên mới gia nhập team hoặc setup server mới, lại phải đi cài đặt và cấu hình biến môi trường `PATH` cho Maven.
- **Tính nhất quán**: Đảm bảo mọi môi trường (Dev, Staging, Prod) đều dùng chính xác một phiên bản Maven để build.

## 3. Mental Model
Hãy tưởng tượng dự án của bạn giống như một **"Bộ đồ nội thất IKEA"**:
- **Maven truyền thống** là việc bạn phải tự có sẵn bộ tua-vít và cờ-lê ở nhà để lắp ráp. Nếu bộ đồ nghề của bạn không khớp với ốc vít của IKEA, bạn chịu thua.
- **Maven Wrapper** là việc IKEA bỏ sẵn một chiếc **cờ-lê nhỏ** ngay bên trong hộp sản phẩm. Bất kể bạn là ai, có đồ nghề hay không, bạn chỉ cần lấy chiếc cờ-lê đó ra là có thể lắp ráp xong bộ đồ nội thất đúng chuẩn.

## 4. Where it fits
Developer -> `./mvnw` -> **mvnw script** -> `.mvn/wrapper/` (Check/Download Maven) -> **Apache Maven Binary** -> Build Logic.

## 5. When to use
- Luôn luôn nên có trong mọi dự án Maven hiện đại (Spring Initializr mặc định tạo sẵn).
- Khi làm việc trong team nhiều người với các hệ điều hành khác nhau.
- Khi cấu hình các pipeline CI/CD (GitHub Actions, Jenkins) để tránh việc phải cài đặt Maven trên runner.

## 6. When NOT to use
- Hầu như không có lý do gì để không dùng. Tuy nhiên, nếu bạn đang phát triển một thư viện cực kỳ nhỏ và muốn giảm thiểu kích thước repository (dù wrapper rất nhẹ), bạn có thể bỏ qua.

## 7. Trade-offs
| Pros | Cons |
|------|------|
| Đảm bảo phiên bản Maven nhất quán 100%. | Làm tăng thêm một vài file nhỏ trong repository (`.mvn/` folder). |
| Không cần cài đặt Maven thủ công. | Lần đầu chạy sẽ tốn thời gian tải Maven về (chỉ 1 lần duy nhất). |
| Dễ dàng nâng cấp Maven cho toàn bộ team bằng cách sửa 1 file config. | N/A |

## 8. Alternatives
| Option | So sánh |
|--------|---------|
| Local Maven (`mvn`) | Phụ thuộc vào cài đặt cá nhân, dễ gây xung đột phiên bản. |
| SDKMAN! | Công cụ quản lý nhiều phiên bản Maven trên một máy, nhưng vẫn yêu cầu cài đặt thủ công. |
| Gradle Wrapper (`gradlew`) | Tương đương nhưng dành cho các dự án sử dụng build tool Gradle. |

## 9. How
Các lệnh phổ biến với Maven Wrapper:

### Chạy build project (thay cho `mvn`)
```bash
# Trên Linux/macOS
./mvnw clean install

# Trên Windows
mvnw.cmd clean install
```

### Cấu hình phiên bản Maven
Bạn có thể thay đổi phiên bản Maven tại file: `.mvn/wrapper/maven-wrapper.properties`
```properties
distributionUrl=https://repo.maven.apache.org/maven2/org/apache/maven/apache-maven/3.9.5/apache-maven-3.9.5-bin.zip
```

### Thêm Wrapper vào dự án cũ (chưa có)
```bash
mvn wrapper:wrapper -Dmaven=3.9.5
```

## 10. Production concerns
### Proxy/Network
Trong môi trường doanh nghiệp có Firewall khắt khe, `mvnw` có thể không tải được Maven binary. Cần cấu hình proxy trong `MAVEN_OPTS` hoặc tải sẵn binary vào một repo nội bộ.

### Storage
Maven binary được tải về sẽ lưu trong thư mục `~/.m2/wrapper/dists`. Cần lưu ý dọn dẹp nếu ổ cứng bị đầy do dùng quá nhiều phiên bản khác nhau.

## 11. Common mistakes
- Mistake: Quên cấp quyền thực thi cho file `mvnw` trên Linux/macOS.
  Fix: Chạy lệnh `chmod +x mvnw`.

- Mistake: Không commit thư mục `.mvn/` lên Git.
  Fix: Thư mục `.mvn/` (chứa file `.jar` và `.properties` của wrapper) **phải** được commit để người khác có thể sử dụng.

## 12. Sample project
Tạo một dự án Spring Boot từ `start.spring.io`. Sau khi tải về, hãy xóa sạch Maven trên máy bạn, sau đó thử chạy `./mvnw spring-boot:run` để thấy sức mạnh của wrapper.

## 13. Interview
### Core Q&A
1. Q: Tại sao Spring Boot luôn tạo sẵn file `mvnw`?
   A: Để đảm bảo trải nghiệm "Out of the box". Người dùng chỉ cần tải code về là có thể chạy ngay mà không gặp lỗi do sai lệch phiên bản Maven.

2. Q: File `.mvn/wrapper/maven-wrapper.jar` đóng vai trò gì?
   A: Nó chứa logic chính để kiểm tra xem Maven đã có sẵn chưa, nếu chưa sẽ thực hiện tải về dựa trên URL trong file `.properties`.

### Scenario
"Build server của công ty không có quyền truy cập Internet để tải Maven. Làm sao để dùng `mvnw`?"
-> Trả lời: Tôi sẽ sửa file `maven-wrapper.properties`, thay đổi `distributionUrl` trỏ về một máy chủ chứa file binary Maven nội bộ (như Nexus hoặc Artifactory) mà build server có quyền truy cập.

## 14. References
- Maven Wrapper Plugin: https://maven.apache.org/wrapper/
- GitHub Source: https://github.com/takari/maven-wrapper

## 15. Real-world Code
Nghiên cứu cấu trúc thư mục của một dự án Spring Boot thực tế:
```text
my-app/
├── .mvn/
│   └── wrapper/
│       ├── maven-wrapper.jar
│       └── maven-wrapper.properties
├── mvnw
└── mvnw.cmd
```

## 16. Community
- Stack Overflow: Tag [maven-wrapper].
- Blog: "Why you should use the Maven Wrapper" - Baeldung.
