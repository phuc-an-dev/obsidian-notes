---
created: 2026-05-07
tags:
  - "#type/concept"
  - "#status/draft"
  - "#lang/others"
  - "#topic/devops"
related:
  - "[[Docker Compose.md]]"
  - "[[Environment Variables.md]]"
---

## 1. What
YAML (viết tắt của "YAML Ain't Markup Language") là một ngôn ngữ định dạng dữ liệu có thể đọc được bởi con người (human-readable). Nó thường được sử dụng cho các tệp cấu hình (configuration files) và trong các ứng dụng nơi dữ liệu đang được lưu trữ hoặc truyền đi. Tệp YAML thường có đuôi là `.yml` hoặc `.yaml`.

## 2. Why
Trước khi có YAML, các định dạng như XML và JSON là phổ biến nhất. Tuy nhiên, XML quá rườm rà với các thẻ đóng/mở, còn JSON đôi khi khó đọc với các dấu ngoặc nhọn `{}` và ngoặc vuông `[]` dày đặc khi cấu hình phức tạp. YAML ra đời để:
- Tối ưu hóa cho khả năng đọc của con người (Readability).
- Giảm bớt các ký tự thừa thãi, sử dụng khoảng trắng (indentation) để phân cấp.
- Hỗ trợ tốt cho việc biểu diễn các cấu trúc dữ liệu phức tạp một cách trực quan.

## 3. Mental Model
Hãy tưởng tượng YAML giống như một **bản danh sách liệt kê (Outline)** được trình bày phân cấp bằng các đầu dòng.
- Mỗi cấp độ lùi vào (indent) đại diện cho một cấp độ con của mục phía trên.
- Dấu gạch ngang `-` đại diện cho một mục trong danh sách (List).
- Dấu hai chấm `:` đại diện cho một cặp thuộc tính và giá trị (Key-Value).
Nó giống như cách bạn ghi chú nhanh các ý chính trong một cuốn sổ tay, nơi vị trí của dòng chữ xác định nó thuộc về mục nào.

## 4. Where it fits
User/Developer -> **YAML File (.yml)** -> Parser (Docker, Kubernetes, Spring Boot) -> Application Logic.

## 5. When to use
- Viết tệp cấu hình cho Docker Compose (`docker-compose.yml`).
- Định nghĩa tài nguyên trong Kubernetes (Pod, Service, Deployment).
- Cấu hình ứng dụng Spring Boot (`application.yml`).
- Viết workflow cho GitHub Actions.
- Cấu hình các công cụ CI/CD như GitLab CI, CircleCI.
- Cấu hình Ansible Playbooks.

## 6. When NOT to use
- Khi cần truyền tải dữ liệu qua mạng với hiệu năng cao và kích thước nhỏ nhất (nên dùng JSON hoặc Protocol Buffers).
- Khi dữ liệu không có tính phân cấp rõ ràng hoặc quá đơn giản (nên dùng tệp `.properties` hoặc `.env`).
- Trong các môi trường mà parser YAML không có sẵn hoặc gây tốn tài nguyên quá mức.

## 7. Trade-offs
| Pros | Cons |
|------|------|
| Cực kỳ dễ đọc và viết đối với con người. | Rất nhạy cảm với khoảng trắng (Indentation errors). |
| Hỗ trợ ghi chú (Comments) trực tiếp trong file. | Cú pháp có thể gây nhầm lẫn (ví dụ: `yes`, `no` có thể bị hiểu là boolean). |
| Cấu trúc dữ liệu phân cấp trực quan. | Parser thường chậm hơn và phức tạp hơn so với JSON. |

## 8. Alternatives
| Option | So sánh |
|--------|---------|
| JSON | Chuẩn hóa hơn cho web API, không nhạy cảm với khoảng trắng, nhưng không hỗ trợ comment. |
| XML | Mạnh mẽ về validation (XSD), nhưng cực kỳ rườm rà và khó đọc. |
| TOML | Tốt cho tệp cấu hình đơn giản, dễ đọc hơn YAML ở một số khía cạnh, nhưng ít phổ biến hơn trong DevOps. |
| .properties | Định dạng Key-Value đơn giản của Java, không hỗ trợ phân cấp tốt. |

## 9. How
Ví dụ một file cấu hình YAML cơ bản:

```yaml
# Thông tin dự án
project:
  name: "My Awesome App"
  version: 1.0.0
  description: |
    Đây là một ví dụ về chuỗi
    nhiều dòng trong YAML.

# Danh sách các môi trường
environments:
  - development
  - staging
  - production

# Cấu hình database
database:
  host: localhost
  port: 5432
  active: true # Boolean
```

## 10. Production concerns
### Security
Cẩn thận với tính năng "Object injection" của một số thư viện parser YAML. Luôn sử dụng Safe Load (ví dụ: `yaml.safe_load()` trong Python) để tránh thực thi mã độc từ file cấu hình.

### Large Files
Tệp YAML quá lớn (hàng ngàn dòng) có thể trở nên cực kỳ khó quản lý và dễ sai sót về indentation. Nên chia nhỏ cấu hình nếu công cụ hỗ trợ.

### Validation
Sử dụng các công cụ như `yamllint` để kiểm tra cú pháp trước khi deploy lên production.

## 11. Common mistakes
- Mistake: Sử dụng phím `Tab` để thụt lề thay vì phím `Space`.
  Fix: Hầu hết các parser YAML chỉ chấp nhận phím cách (Spaces). Luôn cấu hình IDE để tự động chuyển Tab thành Spaces.

- Mistake: Sai lệch thụt lề dẫn đến giá trị bị hiểu sai cấp độ.
  Fix: Sử dụng các extension hỗ trợ YAML trên VS Code hoặc IntelliJ để hiển thị đường kẻ (indent guides).

## 12. Sample project
Viết một file GitHub Actions workflow (`.github/workflows/main.yml`) để tự động chạy unit test mỗi khi có code mới push lên nhánh `main`. Phân tích cấu hình các bước (steps), biến môi trường (env) và các trigger trong file đó.

## 13. Interview
### Core Q&A
1. Q: Sự khác biệt lớn nhất giữa YAML và JSON là gì?
   A: YAML hỗ trợ ghi chú (comments), sử dụng khoảng trắng thay cho dấu ngoặc để phân cấp, và hỗ trợ các kiểu dữ liệu phức tạp hơn như chuỗi nhiều dòng. JSON thì chuẩn hóa hơn và không bị lỗi do khoảng trắng.

2. Q: "Anchor" và "Alias" trong YAML dùng để làm gì?
   A: Dùng để tái sử dụng dữ liệu. Bạn dùng `&` để đặt tên cho một khối dữ liệu (Anchor) và dùng `*` để tham chiếu lại khối đó (Alias) ở chỗ khác, giúp file ngắn gọn hơn.

### Scenario
"Bạn deploy một file cấu hình K8s và nhận được lỗi 'did not find expected key'. Bạn sẽ làm gì?"
-> Trả lời: Lỗi này 99% liên quan đến thụt lề (indentation). Tôi sẽ kiểm tra lại dòng báo lỗi và các dòng phía trên xem có dùng Tab thay cho Space không, hoặc có mục nào bị thụt lề sai vị trí so với cha của nó không. Tôi cũng sẽ dùng một công cụ online YAML Validator để kiểm tra nhanh.

## 14. References
- Official Website: https://yaml.org/
- YAML Lint: http://www.yamllint.com/
- Documentation: https://yaml.org/spec/1.2.2/

## 15. Real-world Code
Nghiên cứu file `docker-compose.yml` của các dự án nổi tiếng như **Elasticsearch** hoặc **PostgreSQL** để học cách họ tổ chức các cấu hình phức tạp và biến môi trường.

## 16. Community
- Reddit: r/devops (nơi thảo luận nhiều nhất về YAML).
- Stack Overflow: Tag [yaml].
- GitHub: Các repository chứa schema validation cho YAML như `schemastore.org`.
