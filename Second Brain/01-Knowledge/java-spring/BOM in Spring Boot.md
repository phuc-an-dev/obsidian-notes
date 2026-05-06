---
created: 2026-05-06
tags:
  - "#type/concept"
  - "#status/draft"
  - "#lang/spring"
  - "#topic/performance"
related:
  - "[[Force Dependency Version in Spring Boot.md]]"
  - "[[Transitive Dependency in Spring Boot.md]]"
---

## 1. What
BOM (Bill of Materials) là một file POM đặc biệt trong Maven dùng để quản lý đồng bộ phiên bản của một nhóm các thư viện liên quan. Thay vì khai báo phiên bản cho từng thư viện, người dùng chỉ cần "import" một BOM duy nhất, và BOM này sẽ định nghĩa sẵn các phiên bản tối ưu và tương thích nhất cho tất cả các thư viện con.

## 2. Why
Trong các hệ sinh thái lớn như Spring Cloud, Google Cloud SDK, hoặc Azure SDK, số lượng thư viện con lên tới hàng trăm. Nếu không có BOM:
- Lập trình viên phải tự nhớ hàng chục số phiên bản khác nhau.
- Rất dễ xảy ra tình trạng không tương thích (vd: Spring Cloud bản A không chạy được với Spring Boot bản B).
BOM giúp đơn giản hóa việc nâng cấp: Chỉ cần đổi 1 số phiên bản của BOM, toàn bộ các thư viện liên quan sẽ được nâng cấp đồng bộ.

## 3. Mental Model
Hãy tưởng tượng BOM giống như một **"Thực đơn combo" (Combo Menu)** ở cửa hàng thức ăn nhanh:
- Thay vì bạn phải chọn lẻ tẻ: Gà rán v1.2, Khoai tây v3.4, Nước ngọt v0.9 (rất dễ bị đau bụng nếu chọn sai bản).
- Bạn chọn "Combo Gia đình" (BOM). Nhà hàng đã đảm bảo các món trong combo này luôn "hợp rơ" và ngon nhất khi đi cùng nhau. Bạn không cần quan tâm chi tiết từng món bản bao nhiêu nữa.

## 4. Where it fits
Vị trí trong file POM:
Nó thường nằm trong block `<dependencyManagement>` với `<scope>import</scope>` và `<type>pom</type>`.

## 5. When to use
- Khi sử dụng Spring Cloud (bắt buộc dùng BOM để đồng bộ các service như Eureka, Config, Gateway).
- Khi dự án của bạn có quá nhiều module và muốn dùng chung một bộ phiên bản thư viện nội bộ.
- Khi dùng các bộ SDK lớn từ các nhà cung cấp Cloud (AWS, GCP, Azure).

## 6. When NOT to use
- Khi dự án chỉ dùng 1-2 thư viện độc lập, không nằm trong một hệ sinh thái lớn.
- Khi bạn muốn kiểm soát hoàn toàn phiên bản của từng thư viện một cách thủ công (rất hiếm khi cần).

## 7. Trade-offs
| Pros | Cons |
|------|------|
| Đảm bảo tính tương thích 100% giữa các thư viện. | Có thể mang vào các phiên bản mà bạn không thực sự muốn (nhưng hiếm khi sai). |
| File POM cực kỳ gọn gàng, không còn các dòng `<version>` rác. | Cần hiểu cơ chế `import scope` trong Maven. |
| Nâng cấp toàn bộ hệ sinh thái chỉ bằng 1 dòng code. | |

## 8. Alternatives
| Option | So sánh |
|--------|---------|
| Parent POM | Thừa kế toàn bộ cấu hình, nhưng mỗi dự án chỉ có 1 Parent. BOM linh hoạt hơn vì 1 dự án có thể import nhiều BOM. |
| Properties | Chỉ quản lý chuỗi ký tự, không quản lý được cấu trúc và sự tương thích như BOM. |

## 9. How
Cách sử dụng BOM trong Maven (Ví dụ Spring Cloud):

```xml
<dependencyManagement>
    <dependencies>
        <dependency>
            <groupId>org.springframework.cloud</groupId>
            <artifactId>spring-cloud-dependencies</artifactId>
            <version>2022.0.3</version>
            <type>pom</type>
            <scope>import</scope>
        </dependency>
    </dependencies>
</dependencyManagement>

<dependencies>
    <dependency>
        <groupId>org.springframework.cloud</groupId>
        <artifactId>spring-cloud-starter-config</artifactId>
        <!-- KHÔNG cần khai báo <version> ở đây -->
    </dependency>
</dependencies>
```

## 10. Production concerns
### Version Consistency
Sử dụng BOM đảm bảo rằng môi trường Production chạy đúng những gì đã được kiểm thử (Tested integrations), giảm thiểu lỗi "MethodNotFound" do lệch version giữa các microservices.

### Upgrade Path
Khi nâng cấp BOM, hãy đọc kỹ "Release Notes" vì nó có thể thay đổi hàng chục thư viện con cùng lúc.

## 11. Common mistakes
- Mistake: Khai báo BOM trong phần `<dependencies>` thay vì `<dependencyManagement>`.
  Fix: BOM phải được import trong `dependencyManagement`.

- Mistake: Quên `<type>pom</type>` và `<scope>import</scope>`. Đây là cú pháp bắt buộc để Maven hiểu đây là một BOM.

## 12. Sample project
Thiết lập một project Spring Boot và import BOM của AWS SDK for Java 2.x. Sau đó thêm các dependency như S3, SQS mà không ghi version.

## 13. Interview
### Core Q&A
1. Q: Sự khác biệt giữa việc dùng `<parent>` và `import BOM` là gì?
   A: `<parent>` cho phép thừa kế cả plugin, properties và dependencies, nhưng chỉ được có 1 parent. `import BOM` chỉ thừa kế phần `dependencyManagement` nhưng bạn có thể import bao nhiêu BOM tùy thích.

2. Q: `scope: import` có ý nghĩa gì?
   A: Nó chỉ được dùng với `<type>pom</type>` trong `dependencyManagement`. Nó bảo Maven hãy thay thế dependency này bằng danh sách các dependencies có trong file POM được trỏ tới.

### Scenario
"Dự án của bạn đang dùng Spring Boot 2.7 và muốn thêm Spring Cloud. Làm sao để chọn đúng phiên bản Spring Cloud?"
-> Trả lời: Tôi sẽ tìm bảng tra cứu tương thích (Release Train) trên trang chủ Spring. Sau đó tôi sẽ import đúng phiên bản BOM của Spring Cloud (vd: `2021.0.x`) vào `dependencyManagement`. Cách này đảm bảo mọi starter của Spring Cloud sẽ tự động chọn version khớp với Spring Boot 2.7.

## 14. References
- Maven: [Introduction to the Bill of Materials](https://maven.apache.org/guides/introduction/introduction-to-dependency-mechanism.html#importing-dependencies)
- Spring Cloud: [Spring Cloud Release Trains](https://spring.io/projects/spring-cloud)

## 15. Real-world Code
Kiểm tra file POM của bất kỳ dự án Spring Cloud nào trên GitHub để thấy cách họ sử dụng `spring-cloud-dependencies` BOM.

## 16. Community
- Baeldung: [Spring SDK BOM](https://www.baeldung.com/spring-maven-bom)
- Stack Overflow: Tag [maven-bom].
