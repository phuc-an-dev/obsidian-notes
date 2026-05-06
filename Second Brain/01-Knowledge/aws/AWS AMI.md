---
created: 2026-05-06
tags:
  - "#type/concept"
  - "#status/draft"
  - "#lang/devops"
  - "#topic/cloud"
related:
  - "[[aws-ec2-instance.md]]"
  - "[[Docker Image.md]]"
---

## 1. What
Amazon Machine Image (AMI) là một đơn vị thông tin được AWS sử dụng để khởi tạo một máy ảo (EC2 instance). Nó đóng vai trò là một gói chứa cấu hình hệ điều hành (OS), máy chủ ứng dụng (Application Server), và các ứng dụng cần thiết để chạy một hệ thống hoàn chỉnh.

## 2. Why
Trước khi có AMI, việc thiết lập một server mới yêu cầu cài đặt OS và cấu hình thủ công rất tốn thời gian và dễ xảy ra sai sót. AMI ra đời để:
- **Tự động hóa**: Khởi tạo hàng loạt server có cấu hình giống hệt nhau trong vài phút.
- **Tính nhất quán**: Đảm bảo môi trường chạy ứng dụng luôn đồng nhất trên mọi instance.
- **Sao lưu (Backup)**: Lưu trữ trạng thái của một server tại một thời điểm để có thể khôi phục khi gặp sự cố.

## 3. Mental Model
Hãy tưởng tượng AMI giống như một **"Khuôn đúc bánh"** hoặc một **"File Ghost/ISO"**:
- Bạn tạo ra một cái khuôn hoàn hảo (AMI) với đầy đủ hình dáng và nguyên liệu.
- Từ cái khuôn đó, bạn có thể đúc ra bao nhiêu chiếc bánh (EC2 Instance) tùy thích.
- Tất cả những chiếc bánh được đúc từ cùng một khuôn sẽ có hình dạng và tính chất y hệt nhau.

## 4. Where it fits
Vị trí trong luồng vận hành:
`AMI (Blueprint) -> Launch Instance Configuration -> EC2 Instance (Running Machine)`

## 5. When to use
- Khi cần triển khai cụm server Auto Scaling (tự động tăng giảm số lượng server theo tải).
- Khi muốn tạo bản sao lưu (Backup) cho server trước khi thực hiện thay đổi lớn.
- Khi muốn chia sẻ cấu hình server chuẩn cho các team khác hoặc cộng đồng (Community AMIs).
- Khi xây dựng môi trường thảm họa (Disaster Recovery).

## 6. When NOT to use
- Khi bạn chỉ cần thay đổi một vài file cấu hình nhỏ (nên dùng User Data hoặc Configuration Management tools như Ansible).
- Khi dữ liệu thay đổi liên tục hàng giây (Dữ liệu động nên lưu ở Database hoặc EBS Volume tách biệt, không nên nhét vào AMI).

## 7. Trade-offs
| Pros | Cons |
|------|------|
| Khởi động instance cực nhanh. | Tốn phí lưu trữ trên S3 (Snapshot). |
| Đảm bảo tính bất biến (Immutability) của server. | Phải tạo lại AMI mới mỗi khi có bản cập nhật ứng dụng/OS. |
| Dễ dàng di chuyển giữa các Region. | Khó chỉnh sửa nội dung bên trong sau khi đã tạo (phải launch ra rồi tạo lại). |

## 8. Alternatives
| Option | So sánh |
|--------|---------|
| Docker Image | Nhẹ hơn, khởi động nhanh hơn nhưng chạy trên lớp OS có sẵn, không chứa toàn bộ Kernel như AMI. |
| User Data Script | Linh hoạt hơn nhưng chậm hơn vì phải chạy script cài đặt mỗi khi server khởi động. |
| VM Import/Export | Chuyển đổi máy ảo từ môi trường On-premise (VMware, VirtualBox) sang AWS. |

## 9. How
Quy trình tạo một Custom AMI từ một EC2 đang chạy:

```bash
# Sử dụng AWS CLI để tạo AMI
aws ec2 create-image \
    --instance-id i-1234567890abcdef0 \
    --name "My Server v1.0" \
    --description "AMI cho Web Server Nginx" \
    --no-reboot
```

Lưu ý: Tham số `--no-reboot` giúp tạo image mà không làm ngắt quãng server, nhưng có rủi ro về tính toàn vẹn dữ liệu (Data integrity). Khuyên dùng reboot để đảm bảo file system ổn định.

## 10. Production concerns
### Region Scope
AMI có tính chất vùng (Region-specific). Để dùng một AMI ở Region khác, bạn phải thực hiện lệnh `Copy Image`.

### Lifecycle Management
Sử dụng Amazon Data Lifecycle Manager (DLM) hoặc AWS Backup để tự động hóa việc tạo và xóa các AMI cũ nhằm tối ưu chi phí.

### Sharing
Có thể chia sẻ AMI cho các AWS Account cụ thể hoặc công khai (Public). Khi chia sẻ, hãy cẩn thận không để lộ Secret Keys hoặc mật khẩu bên trong image.

## 11. Common mistakes
- Mistake: Để nguyên thông tin nhạy cảm (.ssh/authorized_keys, mật khẩu, API keys) trong AMI khi chia sẻ.
  Fix: Sử dụng các công cụ như `cloud-init` để dọn dẹp (cleanup) các thông tin cá nhân trước khi tạo image.

- Mistake: Không kiểm tra xem AMI có dùng EBS-backed hay Instance Store-backed.
  Fix: Luôn ưu tiên EBS-backed vì dữ liệu bền vững và hỗ trợ Stop/Start instance.

## 12. Sample project
Sử dụng **HashiCorp Packer** để xây dựng quy trình "Golden AMI":
1. Định nghĩa một file JSON/HCL mô tả OS (Ubuntu).
2. Tự động chạy script cài đặt Nginx và Java.
3. Packer sẽ tự động launch một server tạm, cài đặt, tạo AMI và xóa server tạm đó.

## 13. Interview
### Core Q&A
1. Q: Làm thế nào để cập nhật phần mềm trong một AMI hiện có?
   A: Bạn không thể sửa AMI trực tiếp. Bạn phải launch một instance từ AMI đó, cài đặt bản cập nhật, sau đó tạo một AMI mới từ instance vừa cập nhật.

2. Q: Sự khác biệt giữa Snapshot và AMI là gì?
   A: Snapshot là bản sao của một ổ đĩa (EBS Volume). AMI là một gói hoàn chỉnh bao gồm Snapshot của ổ đĩa gốc (Root Volume) cộng với các metadata (quyền hạn, block device mapping) để có thể khởi động thành một server.

### Scenario
"Hệ thống Auto Scaling của bạn đang launch các server mới nhưng ứng dụng không tự khởi động. Vấn đề có thể nằm ở đâu?"
-> Trả lời:
1. Kiểm tra xem script khởi động ứng dụng đã được cài đặt làm Systemd service bên trong AMI chưa.
2. Kiểm tra xem file cấu hình ứng dụng có phụ thuộc vào một IP cố định nào đó của server cũ khi tạo AMI không.
3. Kiểm tra logs của `cloud-init` để xem quá trình khởi động server gặp lỗi gì.

## 14. References
- AWS Documentation: [Amazon Machine Images (AMI)](https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/AMIs.html)
- AWS Marketplace: Nơi mua/bán các AMI đã được tối ưu sẵn từ các nhà cung cấp phần mềm.

## 15. Real-world Code
Sử dụng Terraform để tìm kiếm AMI mới nhất của Ubuntu:
```hcl
data "aws_ami" "ubuntu" {
  most_recent = true
  filter {
    name   = "name"
    values = ["ubuntu/images/hvm-ssd/ubuntu-focal-20.04-amd64-server-*"]
  }
  owners = ["099720109477"] # Canonical
}
```

## 16. Community
- AWS Community Builders.
- Reddit: r/aws.
- GitHub: Các repository chứa mã nguồn Packer để build Golden AMIs.
