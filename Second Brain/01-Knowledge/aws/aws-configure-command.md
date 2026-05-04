---
created: 2026-05-04
tags:
  - "#type/tutorial"
  - "#status/draft"
  - "#lang/devops"
  - "#topic/iam"
related:
  - "[[aws-cli-configuration]]"
  - "[[iam-policy]]"
---

## 1. What
`aws configure` là lệnh cơ bản và phổ biến nhất của AWS CLI dùng để thiết lập các thông số xác thực và cấu hình mặc định trên máy tính cá nhân. Khi chạy lệnh này, chương trình sẽ yêu cầu bạn nhập 4 thông tin: Access Key ID, Secret Access Key, Default Region Name, và Default Output Format.

## 2. Why
AWS CLI cần biết bạn là ai (Credentials) và bạn muốn thao tác ở đâu (Region) để có thể gửi các yêu cầu API hợp lệ tới AWS Cloud. Nếu không chạy lệnh này, bạn sẽ nhận được lỗi "Unable to locate credentials" mỗi khi thực hiện bất kỳ lệnh nào. Đây là bước "chào hỏi" đầu tiên giữa máy tính của bạn và AWS.

## 3. Mental Model
Hãy tưởng tượng `aws configure` như việc bạn đi **"Đăng ký sim điện thoại"**:
- Bạn đưa Chứng minh thư (Access Key ID) và Chữ ký (Secret Access Key) cho nhà mạng.
- Bạn chọn khu vực phủ sóng ưu tiên (Default Region).
- Bạn chọn ngôn ngữ nhận tin nhắn (Output Format).
Sau khi đăng ký xong, chiếc điện thoại (AWS CLI) của bạn mới chính thức hoạt động được.

## 4. Where it fits
Nằm ở bước thiết lập môi trường phát triển (Dev Environment Setup):
`Install AWS CLI -> Run aws configure -> Start Managing Resources`

## 5. When to use
- Khi bạn lần đầu cài đặt AWS CLI trên máy mới.
- Khi bạn được cấp một cặp Access Key mới từ quản trị viên IAM.
- Khi bạn muốn thay đổi vùng (Region) mặc định cho các thao tác CLI.

## 6. When NOT to use
- Khi chạy code trên các dịch vụ tính toán của AWS (như EC2, Lambda). Trong trường hợp này, hãy dùng **IAM Role** thay vì cấu hình key thủ công.
- Khi bạn muốn thiết lập nhiều tài khoản khác nhau (nên dùng `aws configure --profile`).

## 7. Trade-offs
| Pros | Cons |
|------|------|
| Cực kỳ đơn giản, giao diện tương tác từng bước. | Lưu key dưới dạng văn bản thuần túy trong tệp tin (rủi ro bảo mật). |
| Thiết lập nhanh chóng các giá trị mặc định. | Khó quản lý nếu có quá nhiều cặp key khác nhau. |
| Tự động tạo thư mục và tệp cấu hình cần thiết. | |

## 8. Alternatives
| Option | So sánh |
|--------|---------|
| **Biến môi trường** | Thiết lập `AWS_ACCESS_KEY_ID`, nhanh nhưng chỉ có tác dụng trong phiên làm việc đó. |
| **Chỉnh sửa file thủ công** | Mở tệp `~/.aws/credentials` và gõ nội dung vào, nhanh đối với người dùng thâm niên. |
| **AWS SSO** | Đăng nhập qua trình duyệt, an toàn hơn vì không lưu permanent key vào máy. |

## 9. How
```bash
# Chạy lệnh
aws configure

# Quy trình tương tác:
# AWS Access Key ID [None]: AKIAIOSFODNN7EXAMPLE
# AWS Secret Access Key [None]: wJalrXUtnFEMI/K7MDENG/bPxRfiCYEXAMPLEKEY
# Default region name [None]: ap-southeast-1
# Default output format [None]: json

# Kiểm tra xem cấu hình đã lưu chưa
aws sts get-caller-identity
```

## 10. Production concerns
### Scaling
Đối với các team lớn, việc bắt mọi người chạy `aws configure` và lưu key cá nhân là một gánh nặng quản lý. Hãy chuyển sang dùng **IAM Identity Center (SSO)** để nhân viên đăng nhập bằng tài khoản công ty.

### Failure
Nếu nhập sai Secret Key, các lệnh CLI sẽ báo lỗi `SignatureDoesNotMatch`. Bạn chỉ cần chạy lại lệnh `aws configure` để ghi đè giá trị đúng.

### Monitoring
Các tệp cấu hình được lưu tại `~/.aws/` (Linux/macOS) hoặc `%USERPROFILE%\.aws\` (Windows). Cần đảm bảo quyền truy cập tệp này chỉ dành cho User hiện tại.

## 11. Common mistakes
- **Mistake**: Nhập sai định dạng Region (ví dụ nhập "Singapore" thay vì "ap-southeast-1").
  **Fix**: Luôn dùng đúng mã Code của Region theo tài liệu AWS.

- **Mistake**: Coi `aws configure` là bảo mật tuyệt đối.
  **Fix**: Key được lưu trong tệp `~/.aws/credentials`. Nếu máy tính bị mất hoặc bị hacker xâm nhập, key này sẽ bị lộ.

## 12. Sample project
Tạo một IAM User dành riêng cho việc học tập (chỉ có quyền đọc S3), lấy key của user đó và dùng `aws configure` để thiết lập trên máy cá nhân, sau đó thử liệt kê các bucket.

## 13. Interview
### Core Q&A
1. **Q**: Lệnh `aws configure` lưu thông tin vào những tệp nào?
   **A**: Lưu key vào `~/.aws/credentials` và lưu region/format vào `~/.aws/config`.
2. **Q**: Có cách nào để bảo mật key tốt hơn khi dùng CLI không?
   **A**: Dùng các công cụ như `aws-vault` để mã hóa và lưu key vào hệ thống quản lý mật khẩu của OS (như Keychain trên Mac).
3. **Q**: Làm sao để cấu hình cho một Region cụ thể mà không làm thay đổi giá trị mặc định?
   **A**: Bạn không cần chạy configure lại, chỉ cần thêm tham số `--region us-east-1` vào lệnh bạn đang gõ.

### Scenario
**Tình huống**: Bạn đã cấu hình `aws configure` nhưng khi chạy lệnh `aws s3 ls` vẫn báo "Unable to locate credentials". Tại sao?
**Trả lời**: Có thể tệp credentials nằm sai vị trí hoặc bạn đang có một biến môi trường `AWS_ACCESS_KEY_ID` rỗng đang ghi đè lên cấu hình trong file.

## 14. References
- AWS CLI User Guide: [Quick configuration](https://docs.aws.amazon.com/cli/latest/userguide/cli-chap-configure.html)

## 15. Real-world Code
- Một ví dụ về file `~/.aws/config` sau khi chạy lệnh:
  ```ini
  [default]
  region = ap-southeast-1
  output = json
  ```

## 16. Community
- Reddit: r/aws
- Stack Overflow: Tag [aws-cli]
