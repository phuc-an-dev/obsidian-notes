---
created: 2026-05-06
tags:
  - "#type/concept"
  - "#status/draft"
  - "#lang/devops"
  - "#topic/cloud"
related:
  - "[[aws-ec2-instance.md]]"
  - "[[iam.md]]"
---

## 1. What
Instance Metadata Service (IMDS) là một dịch vụ nội bộ chạy trên mọi thực thể Amazon EC2, cung cấp dữ liệu về chính thực thể đó (metadata). Người dùng và ứng dụng chạy trên EC2 có thể truy cập dịch vụ này thông qua một địa chỉ IP HTTP cục bộ không thể định tuyến (non-routable): `169.254.169.254`.

## 2. Why
Khi chạy ứng dụng trên Cloud, ứng dụng thường cần biết các thông tin về môi trường thực thi của nó mà không muốn hardcode. IMDS giải quyết các vấn đề:
- **Tự động cấu hình**: Lấy địa chỉ IP public/private, hostname, hoặc ID của instance để tự đăng ký vào hệ thống log hoặc monitoring.
- **Bảo mật**: Lấy các credentials tạm thời (Temporary Credentials) từ IAM Role gắn với instance (Instance Profile) để truy cập các dịch vụ AWS khác mà không cần lưu trữ Access Key trên đĩa.
- **Tùy biến**: Truy cập vào User Data (script chạy lúc khởi tạo) để hoàn tất cấu hình phần mềm.

## 3. Mental Model
Hãy tưởng tượng IMDS giống như một **"Gương soi thông minh"** được gắn sẵn trong mỗi phòng (EC2 Instance):
- Căn phòng không biết nó ở tầng mấy hay số phòng bao nhiêu.
- Nhưng khi nó nhìn vào gương (`169.254.169.254`), chiếc gương sẽ hiển thị mọi thông tin: "Bạn là phòng 101, thuộc tòa nhà A, đang có khóa bảo vệ cấp bởi lễ tân".
- Chỉ những người ở trong phòng mới soi được chiếc gương này, người ở ngoài tòa nhà không thể nhìn thấy.

## 4. Where it fits
Vị trí trong kiến trúc EC2:
`Application/User -> HTTP Get -> 169.254.169.254 (IMDS) -> EC2 Hypervisor -> Instance Metadata`

## 5. When to use
- Khi cần lấy IAM Role credentials để ứng dụng gọi API AWS (S3, SQS).
- Khi cần xác định Availability Zone (AZ) mà instance đang chạy để tối ưu hóa traffic.
- Khi cần đọc User Data script để thực hiện cấu hình động lúc runtime.
- Khi cần lấy Public IP của chính nó để thông báo cho một dịch vụ DNS bên ngoài.

## 6. When NOT to use
- Khi bạn đang chạy ứng dụng trong Docker container và muốn quản lý credentials qua ECS Task Role hoặc Kubernetes Service Account (mặc dù IMDS vẫn khả dụng nhưng có các phương thức chuyên biệt hơn).
- Khi bạn cần các dữ liệu không thuộc về metadata của instance (như dữ liệu nghiệp vụ của ứng dụng).

## 7. Trade-offs
| Pros | Cons |
|------|------|
| Không cần cấu hình endpoint, luôn là IP 169.254.169.254. | Truy cập qua HTTP không mã hóa (mặc dù chỉ giới hạn trong instance). |
| Cung cấp credentials tạm thời cực kỳ an toàn. | Có nguy cơ bị tấn công SSRF (Server-Side Request Forgery) nếu ứng dụng có lỗ hổng. |
| Miễn phí hoàn toàn và không giới hạn băng thông nội bộ. | IMDSv1 có bảo mật yếu hơn so với IMDSv2. |

## 8. Alternatives
| Option | So sánh |
|--------|---------|
| Environment Variables | Phải gán thủ công lúc khởi tạo, không tự động cập nhật credentials. |
| AWS CLI / SDK | Thực chất AWS CLI/SDK cũng gọi tới IMDS để lấy thông tin nếu không tìm thấy credentials local. |

## 9. How
Truy cập IMDS qua dòng lệnh (Ví dụ IMDSv2 - Bản bảo mật):

```bash
# 1. Lấy Token (Bắt buộc cho IMDSv2)
TOKEN=`curl -X PUT "http://169.254.169.254/latest/api/token" -H "X-aws-ec2-metadata-token-ttl-seconds: 21600"`

# 2. Lấy Instance ID
curl -H "X-aws-ec2-metadata-token: $TOKEN" http://169.254.169.254/latest/meta-data/instance-id

# 3. Lấy IAM Credentials
curl -H "X-aws-ec2-metadata-token: $TOKEN" http://169.254.169.254/latest/meta-data/iam/security-credentials/my-iam-role

# 4. Lấy User Data
curl -H "X-aws-ec2-metadata-token: $TOKEN" http://169.254.169.254/latest/user-data
```

## 10. Production concerns
### IMDSv2 (Session-oriented)
Luôn cấu hình instance để **chỉ cho phép IMDSv2**. Phiên bản này yêu cầu một Token trong Header, giúp ngăn chặn các cuộc tấn công SSRF phổ biến nơi kẻ tấn công chỉ có thể thực hiện GET request đơn giản mà không thể thêm Header.

### Hop Limit
Khi chạy Docker, Token có thể không đến được container do giới hạn số chặng (Hop Limit) của gói tin HTTP. Cần tăng `http-put-response-hop-limit` lên 2 hoặc cao hơn.

### Security
Vô hiệu hóa IMDS nếu instance không thực sự cần (ví dụ: một proxy tĩnh không gọi API AWS nào) để giảm thiểu bề mặt tấn công.

## 11. Common mistakes
- Mistake: Sử dụng IMDSv1 trên Production.
  Fix: Chuyển sang IMDSv2 bằng lệnh `aws ec2 modify-instance-metadata-options`.

- Mistake: Hardcode IP `169.254.169.254` trong code thay vì dùng thư viện chính thức của AWS.
  Fix: Luôn ưu tiên dùng AWS SDK, nó sẽ tự động handle việc lấy Token và gọi IMDS đúng cách.

## 12. Sample project
Viết một script Bash để tự động gắn thẻ (Tag) cho instance dựa trên thông tin lấy từ IMDS:
1. Lấy Instance ID từ IMDS.
2. Lấy Region từ AZ trong IMDS.
3. Gọi lệnh `aws ec2 create-tags` để gắn tag "Environment=Production".

## 13. Interview
### Core Q&A
1. Q: Địa chỉ IP của Instance Metadata Service là gì?
   A: `169.254.169.254`.

2. Q: Tại sao IMDSv2 lại an toàn hơn IMDSv1?
   A: IMDSv2 yêu cầu một quá trình "bắt tay" để lấy Token qua lệnh PUT và yêu cầu Token đó trong Header của mọi request sau đó. Điều này ngăn chặn hacker lợi dụng lỗi SSRF trên ứng dụng (thường chỉ thực hiện được GET) để đánh cắp credentials.

### Scenario
"Ứng dụng của bạn chạy trong Docker trên EC2 không lấy được IAM credentials từ IMDSv2. Nguyên nhân có thể do đâu?"
-> Trả lời: Khả năng cao nhất là do `http-put-response-hop-limit` đang để bằng 1 (mặc định). Khi gói tin đi từ EC2 host vào Docker container, nó bị tính là 1 hop và bị drop. Cần tăng giới hạn này lên 2.

## 14. References
- AWS Documentation: [Instance metadata and user data](https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/ec2-instance-metadata.html)
- AWS Security Blog: [Defense in depth with IMDSv2](https://aws.amazon.com/blogs/security/defense-in-depth-using-session-oriented-armoring-for-the-ec2-instance-metadata-service/)

## 15. Real-world Code
Hầu hết các công cụ như `Cloud-init` hoặc `AWS CLI` đều mặc định sử dụng dịch vụ này để tự động cấu hình môi trường khi instance vừa startup.

## 16. Community
- Reddit: r/aws.
- Stack Overflow: Tag [amazon-imds].
