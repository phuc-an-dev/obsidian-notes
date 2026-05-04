---
created: 2026-05-04
tags:
  - "#type/tutorial"
  - "#status/draft"
  - "#lang/devops"
  - "#topic/iam"
related:
  - "[[iam-policy]]"
  - "[[macos-environment-variables]]"
---

## 1. What
**AWS CLI (Command Line Interface)** là công cụ dòng lệnh mạnh mẽ để quản lý các dịch vụ AWS. **Profile** là một tập hợp các thông số cấu hình (credentials, region, output format) được lưu trữ dưới một cái tên cụ thể, cho phép bạn nhanh chóng chuyển đổi giữa các tài khoản hoặc quyền hạn khác nhau.

## 2. Why
Việc truy cập AWS Console (giao diện web) rất tốn thời gian cho các tác vụ lặp đi lặp lại. AWS CLI giúp tự động hóa mọi thứ qua script. Cơ chế **Profile** cực kỳ quan trọng nếu bạn làm việc với nhiều tài khoản (ví dụ: Dev, Staging, Production) hoặc nhiều khách hàng khác nhau trên cùng một máy tính.

## 3. Mental Model
Hãy tưởng tượng AWS CLI như một chiếc **"Điều khiển từ xa đa năng (Universal Remote)"**. Mỗi **Profile** giống như một **"Nút bấm"** được lập trình sẵn cho một thiết bị: Nút 1 cho TV (Tài khoản Dev), Nút 2 cho Điều hòa (Tài khoản Prod). Bạn không cần phải tháo pin và cài đặt lại mỗi khi muốn điều khiển thiết bị khác, chỉ cần nhấn đúng nút (gọi đúng profile).

## 4. Where it fits
Nó nằm ở máy cục bộ (Local Machine) của lập trình viên hoặc server CI/CD:
`Developer Command -> AWS CLI (Profile Selection) -> ~/.aws/credentials -> AWS API`

## 5. When to use
- Khi cần upload file lên S3, khởi chạy EC2 từ dòng lệnh.
- Khi quản lý nhiều môi trường (Dev/Prod) một cách biệt lập.
- Khi xây dựng các script tự động hóa (Bash/Python) tương tác với AWS.

## 6. When NOT to use
- Khi chạy code bên trong EC2/Lambda (nên dùng **IAM Role** để an toàn hơn).
- Khi bạn chỉ cần xem nhanh một thông số trên console mà không cần tự động hóa.

## 7. Trade-offs
| Pros | Cons |
|------|------|
| Tốc độ xử lý cực nhanh, hỗ trợ script. | Lưu trữ credentials (key) trực tiếp trên ổ cứng (cần bảo mật máy cá nhân). |
| Quản lý được hàng trăm tài khoản qua Profile. | Phải nhớ các lệnh và tham số phức tạp. |
| Hỗ trợ nhiều định dạng đầu ra (JSON, Table, Text). | |

## 8. Alternatives
| Option | So sánh |
|--------|---------|
| **AWS Management Console** | Giao diện trực quan nhưng chậm, không thể tự động hóa. |
| **AWS SDKs** | Dùng trong mã nguồn (Java, Python), mạnh mẽ hơn cho logic phức tạp. |
| **AWS CloudShell** | CLI chạy trực tiếp trên trình duyệt, không cần cài đặt local. |

## 9. How
Cấu hình Profile mặc định:
```bash
aws configure
# Nhập Access Key, Secret Key, Region (ví dụ: ap-southeast-1), Format (json)
```

Cấu hình Profile có tên (Named Profile):
```bash
aws configure --profile dev-account
```

Sử dụng Profile:
```bash
# Cách 1: Thêm tham số --profile vào lệnh
aws s3 ls --profile dev-account

# Cách 2: Thiết lập biến môi trường cho phiên làm việc hiện tại
export AWS_PROFILE=dev-account
aws s3 ls
```

## 10. Production concerns
### Scaling
Khi quản lý hàng trăm profile, tệp `~/.aws/credentials` sẽ rất khó kiểm soát. Hãy sử dụng các công cụ như **AWS SSO (IAM Identity Center)** để quản lý đăng nhập tập trung.

### Failure
Nếu key hết hạn hoặc sai, bạn sẽ nhận lỗi `InvalidClientTokenId` hoặc `SignatureDoesNotMatch`. Kiểm tra lại tệp `~/.aws/credentials`.

### Monitoring
Luôn đặt `output = json` trong cấu hình để dễ dàng parse dữ liệu bằng các công cụ như `jq`.

## 11. Common mistakes
- **Mistake**: Lưu key của tài khoản Root (Root User) vào AWS CLI.
  **Fix**: Luôn dùng IAM User với quyền hạn hạn chế.

- **Mistake**: Commit tệp `~/.aws/credentials` lên GitHub.
  **Fix**: Đảm bảo tệp này luôn nằm trong `.gitignore` nếu bạn lỡ tay copy vào thư mục dự án.

## 12. Sample project
Tạo một script bash để kiểm tra dung lượng của tất cả S3 Buckets trong 2 tài khoản khác nhau (Dev và Prod) bằng cách luân chuyển giữa các profile `dev` và `prod`.

## 13. Interview
### Core Q&A
1. **Q**: Tệp cấu hình của AWS CLI nằm ở đâu trên máy tính?
   **A**: Thường nằm ở thư mục home: `~/.aws/credentials` (lưu key) và `~/.aws/config` (lưu region/format).
2. **Q**: Làm thế nào để dùng CLI mà không cần lưu key vào máy?
   **A**: Dùng **AWS SSO** (`aws sso login`) hoặc dùng các công cụ bọc như `aws-vault` để lưu key vào Keychain của hệ điều hành.
3. **Q**: Độ ưu tiên của credentials trong AWS CLI như thế nào?
   **A**: Biến môi trường (`AWS_ACCESS_KEY_ID`) có ưu tiên cao nhất -> sau đó đến tham số dòng lệnh -> cuối cùng mới đến Profile trong file config.

### Scenario
**Tình huống**: Bạn chạy lệnh `aws s3 ls` nhưng nó lại liệt kê bucket của tài khoản cá nhân thay vì tài khoản công ty. Bạn xử lý thế nào?
**Trả lời**: Do tôi đang dùng profile mặc định (`default`). Tôi cần thêm `--profile công-ty` vào lệnh hoặc chạy `export AWS_PROFILE=công-ty` trước khi thực hiện các lệnh tiếp theo.

## 14. References
- Official Docs: [Named Profiles for AWS CLI](https://docs.aws.amazon.com/cli/latest/userguide/cli-configure-profiles.html)

## 15. Real-world Code
- Tệp `~/.aws/credentials` mẫu:
  ```ini
  [default]
  aws_access_key_id=AKIA...
  aws_secret_access_key=wJalr...

  [dev]
  aws_access_key_id=AKIA_DEV...
  aws_secret_access_key=qWerty...
  ```

## 16. Community
- Reddit: r/aws
- Stack Overflow: Tag [aws-cli]
