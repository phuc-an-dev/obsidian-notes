---
created: 2026-05-05
tags:
  - "#type/concept"
  - "#status/draft"
  - "#lang/devops"
  - "#topic/compute"
related:
  - "[[aws-ec2-instance]]"
  - "[[aws-ec2-security-groups]]"
---

## 1. What
AWS EC2 Key Pairs là một phương thức xác thực bảo mật dựa trên mã hóa bất đối xứng (Asymmetric Cryptography). Nó bao gồm một Public Key được AWS lưu trữ trên Instance và một Private Key mà người dùng tải về để chứng thực quyền truy cập khi kết nối SSH (Linux) hoặc giải mã mật khẩu RDP (Windows).

## 2. Why
Sử dụng mật khẩu truyền thống cho các server công khai rất dễ bị tấn công Brute-force. Key Pairs loại bỏ rủi ro này bằng cách yêu cầu một file khóa duy nhất có độ phức tạp cao, đảm bảo chỉ những ai sở hữu file Private Key mới có thể truy cập được vào hệ thống.

## 3. Mental Model
Hãy tưởng tượng Public Key là một cái ổ khóa gắn chặt trên cửa nhà (EC2 Instance). AWS là người thợ rèn giúp bạn đúc ổ khóa này. Private Key là chiếc chìa khóa duy nhất mở được ổ khóa đó. Bạn phải tự giữ chìa khóa này trong túi. Nếu bạn làm mất chìa khóa, thợ rèn (AWS) cũng không có chìa dự phòng để mở cửa cho bạn.

## 4. Where it fits
User Terminal -> SSH Client (Private Key) -> Network -> EC2 Instance (Public Key in ~/.ssh/authorized_keys).

## 5. When to use
- Đăng nhập vào Linux instances thông qua giao thức SSH.
- Lấy mật khẩu Administrator mặc định cho các Windows instances.
- Sử dụng làm phương thức xác thực cho các tool tự động hóa như Ansible hoặc Terraform khi cần can thiệp trực tiếp vào OS.

## 6. When NOT to use
- Khi tổ chức có chính sách bảo mật khắt khe không cho phép lưu trữ key files cục bộ (nên dùng AWS Systems Manager Session Manager).
- Khi muốn quản lý quyền truy cập tập trung qua IAM (nên dùng EC2 Instance Connect).

## 7. Trade-offs
| Pros | Cons |
|------|------|
| Bảo mật cực cao so với mật khẩu. | Nếu mất Private Key, việc khôi phục quyền truy cập rất phức tạp. |
| Không cần nhớ mật khẩu phức tạp. | Khó quản lý khi số lượng server và nhân sự tăng lên (Key sprawl). |
| Chuẩn hóa quốc tế (SSH standard). | Rủi ro lộ file Private Key nếu vô tình commit lên Git. |

## 8. Alternatives
| Option | So sánh |
|--------|---------|
| AWS Systems Manager (SSM) | Không cần Key Pair, không cần mở port 22, log được toàn bộ command. |
| EC2 Instance Connect | Truy cập SSH trực tiếp từ trình duyệt bằng quyền IAM, key tạm thời. |
| IAM Roles | Dùng cho ứng dụng chạy trên EC2 truy cập dịch vụ AWS khác (không dùng để login). |

## 9. How
Tạo một Key Pair mới bằng AWS CLI:
```bash
aws ec2 create-key-pair \
    --key-name MyProjectKey \
    --query 'KeyMaterial' \
    --output text > MyProjectKey.pem

# Quan trọng: Phải phân quyền cho file key trên Linux/Mac
chmod 400 MyProjectKey.pem
```

## 10. Production concerns
### Scaling
Trong môi trường lớn, tránh dùng chung một Key Pair cho tất cả mọi người. Nên sử dụng cơ chế xoay vòng (Rotation) hoặc tích hợp với các giải pháp quản lý danh tính tập trung.

### Failure
Rủi ro lớn nhất là mất file Private Key. Chiến lược dự phòng: Luôn cài đặt SSM Agent trên instance để có đường truy cập "backdoor" qua Systems Manager khi mất key.

### Monitoring
Theo dõi CloudTrail để biết ai đã tạo, xóa hoặc thay đổi Key Pairs trong tài khoản AWS.

## 11. Common mistakes
- Mistake: Lưu trữ file Private Key (.pem) trong thư mục dự án và vô tình push lên GitHub.
  Fix: Luôn thêm `*.pem` vào `.gitignore` và sử dụng AWS Secrets Manager để lưu trữ key nếu cần dùng trong CI/CD.

- Mistake: Không set quyền `chmod 400` cho file key, dẫn đến lỗi "Permissions are too open" khi SSH.
  Fix: Chạy lệnh `chmod 400 <file_name>.pem` để giới hạn quyền chỉ cho chủ sở hữu được đọc.

## 12. Sample project
Tự động hóa việc cấu hình SSH: Tạo file `~/.ssh/config` để quản lý nhiều EC2 instance với các Key Pairs khác nhau, cho phép login chỉ bằng lệnh `ssh web-server` thay vì gõ đầy đủ các tham số dài dòng.

## 13. Interview
### Core Q&A
1. Q: Nếu bạn làm mất Private Key của một EC2 instance đang chạy, bạn phải làm gì để lấy lại quyền truy cập?
   A: Có vài cách: 1. Sử dụng SSM Session Manager nếu đã cài Agent. 2. Stop instance, tháo EBS volume gắn sang một instance khác để sửa file `authorized_keys`, sau đó gắn lại. 3. Sử dụng EC2 User Data script để nạp Public Key mới.
2. Q: Sự khác biệt giữa định dạng .pem và .ppk là gì?
   A: `.pem` (Privacy Enhanced Mail) là định dạng chuẩn dùng cho OpenSSH (Linux/Mac). `.ppk` (PuTTY Private Key) là định dạng riêng của tool PuTTY trên Windows. Có thể dùng PuTTYgen để chuyển đổi giữa hai loại này.

### Scenario
Một nhân viên vừa nghỉ việc và họ có giữ Private Key của các server quan trọng. Bạn sẽ xử lý tình huống này như thế nào để đảm bảo an toàn?
Trả lời: Cần thực hiện quy trình "Key Rotation". Tạo một Key Pair mới, nạp Public Key mới vào file `~/.ssh/authorized_keys` của toàn bộ các instance, kiểm tra kết nối bằng key mới thành công, sau đó xóa Public Key cũ khỏi server và xóa Key Pair cũ trên console AWS.

## 14. References
- Official Docs: https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/ec2-key-pairs.html
- GitHub Repo: N/A
- Spec / RFC: RFC 4253 (SSH Transport Layer Protocol)
- Changelog: N/A

## 15. Real-world Code
- Ansible module `authorized_key`: https://docs.ansible.com/ansible/latest/collections/ansible/posix/authorized_key_module.html

## 16. Community
- Reddit: r/aws - Discussions on "Lost EC2 key pair".
- Stack Overflow: How to recover lost AWS key pair.
- Blog: AWS Security Blog - "Best practices for managing SSH key pairs".
