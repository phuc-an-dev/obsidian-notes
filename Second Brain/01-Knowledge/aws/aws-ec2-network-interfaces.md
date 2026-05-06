---
created: 2026-05-05
tags:
  - "#type/concept"
  - "#status/draft"
  - "#lang/devops"
  - "#topic/aws"
related:
  - "[[ec2]]"
---

## 1. What
Elastic Network Interface (ENI) là một thành phần mạng ảo (virtual network component) trong AWS đại diện cho một card mạng ảo (Virtual Network Interface Card - vNIC). Mỗi EC2 instance mặc định có một ENI chính (primary) và có thể gắn thêm các ENI phụ (secondary).

## 2. Why
Trước khi có ENI linh hoạt, cấu hình mạng thường dính chặt với phần cứng hoặc instance. ENI ra đời để tách biệt định danh mạng (IP, MAC address) khỏi sức mạnh tính toán (EC2). Điều này cho phép "di chuyển" danh tính mạng từ instance này sang instance khác, hỗ trợ quản lý traffic phức tạp và tăng khả năng sẵn sàng (high availability).

## 3. Mental Model
Hãy tưởng tượng EC2 Instance là một chiếc máy tính xách tay và ENI là một chiếc USB-to-LAN adapter (card mạng gắn ngoài). Bạn có thể rút card mạng này ra khỏi máy này và cắm vào máy khác. Tất cả các thiết lập như địa chỉ IP, quyền truy cập (Security Group) đều nằm trên chiếc card mạng đó, nên khi bạn cắm nó vào máy mới, máy đó lập tức có "danh tính mạng" của chiếc card đó.

## 4. Where it fits
ENI nằm ở lớp Networking trong AWS. Nó là điểm kết nối giữa EC2 Instance và VPC Subnet.
`EC2 Instance <-> ENI <-> Subnet <-> VPC Router`.

## 5. When to use
- Tạo Dual-homed instances: Một instance kết nối với hai subnet khác nhau (ví dụ: Management subnet và Public subnet).
- Xây dựng Network Appliances: Như Firewalls, Load Balancers, NAT instances cần nhiều card mạng để xử lý traffic đi qua.
- High Availability (Low-budget): Di chuyển IP nhanh chóng từ instance bị lỗi sang instance dự phòng (Warm standby).
- Licensing: Một số phần mềm bản quyền yêu cầu MAC address cố định.

## 6. When NOT to use
- Các ứng dụng web đơn giản chỉ cần một địa chỉ IP công cộng và truy cập Internet cơ bản.
- Khi có thể dùng Elastic IP (EIP) để trỏ vào instance, vì EIP linh hoạt hơn và không bị giới hạn bởi instance type nhiều như ENI.
- Khi yêu cầu băng thông cực lớn (>100Gbps), lúc đó nên cân nhắc Elastic Fabric Adapter (EFA).

## 7. Trade-offs
| Pros | Cons |
|------|------|
| Linh hoạt trong việc quản lý IP và MAC address. | Giới hạn số lượng ENI tùy thuộc vào Instance Type. |
| Có thể gán nhiều Security Group khác nhau cho từng ENI. | Phức tạp trong việc cấu hình Routing Table bên trong hệ điều hành (OS). |
| Cho phép tách biệt traffic (Management vs Data). | Phí phát sinh nếu dùng quá nhiều Public IP trên các ENI. |

## 8. Alternatives
| Option | So sánh |
|--------|---------|
| ENA (Enhanced Networking) | Thế hệ card mạng mới cho hiệu năng cao hơn, độ trễ thấp hơn. |
| EFA (Elastic Fabric Adapter) | Dành cho HPC (High Performance Computing), hỗ trợ bypass OS kernel. |
| Elastic IP (EIP) | Chỉ là địa chỉ IP tĩnh, không bao gồm MAC hay cấu hình card mạng vật lý. |

## 9. How
Tạo và gắn ENI bằng AWS CLI:
```bash
# 1. Tạo ENI trong một subnet cụ thể
aws ec2 create-network-interface --subnet-id subnet-0123456789abcdef0 --description "Secondary ENI"

# 2. Gắn ENI vào instance (index 1 là secondary interface)
aws ec2 attach-network-interface --network-interface-id eni-0123456789abcdef0 --instance-id i-0123456789abcdef0 --device-index 1

# 3. Kiểm tra trạng thái ENI
aws ec2 describe-network-interfaces --network-interface-ids eni-0123456789abcdef0
```

## 10. Production concerns
### Scaling
Mỗi loại EC2 Instance (t3.micro, m5.large...) có giới hạn số lượng ENI tối đa và số lượng IP trên mỗi ENI. Cần kiểm tra bảng giới hạn của AWS trước khi thiết kế hệ thống cần nhiều IP.

### Failure
Primary ENI (eth0) không bao giờ có thể tách rời khỏi instance. Nếu instance bị xóa, primary ENI cũng biến mất theo mặc định (có thể cấu hình lại). Secondary ENI thì có thể tồn tại độc lập.

### Monitoring
Sử dụng VPC Flow Logs để theo dõi lưu lượng mạng đi qua từng ENI cụ thể. CloudWatch cũng cung cấp các chỉ số như `NetworkIn` và `NetworkOut` ở cấp độ interface.

## 11. Common mistakes
- Mistake: Cố gắng rút Primary ENI (eth0) ra khỏi một instance đang chạy.
  Fix: Chỉ có thể di chuyển Secondary ENIs. Nếu muốn chuyển IP của eth0, hãy dùng Elastic IP.

- Mistake: Gắn 2 ENIs vào cùng một subnet mà không cấu hình routing bên trong OS, dẫn đến lỗi "Asymmetric Routing".
  Fix: Cấu hình Source Routing trong Linux (ip rule, ip route) để đảm bảo traffic quay về đúng interface nó đã đi vào.

## 12. Sample project
Thiết kế một hệ thống "Internal Firewall" sử dụng EC2. Instance này sẽ có 2 ENI:
- ENI 1 (Untrusted): Nằm trong Public Subnet để nhận traffic từ Internet.
- ENI 2 (Trusted): Nằm trong Private Subnet để gửi traffic đã lọc sạch vào các Server nội bộ.
- Yêu cầu: Cấu hình Security Group trên ENI 1 cực kỳ chặt chẽ (chỉ mở port 80/443).

## 13. Interview
### Core Q&A
1. Q: Làm thế nào để di chuyển một địa chỉ IP riêng (Private IP) từ instance này sang instance khác mà không gây gián đoạn lâu?
   A: Cách tốt nhất là gán Private IP đó vào một Secondary ENI. Khi cần chuyển, ta thực hiện Detach ENI đó khỏi instance cũ và Attach vào instance mới. Quá trình này giữ nguyên cả Private IP và Security Group gắn kèm.

2. Q: "Device Index" trong cấu hình ENI có ý nghĩa gì?
   A: Device Index xác định thứ tự của interface trong hệ điều hành. Index 0 luôn là primary interface (eth0). Các index tiếp theo (1, 2, 3...) là các secondary interfaces (eth1, eth2...).

### Scenario
Tình huống: "Bạn có một ứng dụng cũ chỉ chạy được nếu MAC address không thay đổi, nhưng bạn cần nâng cấp Instance Type từ t2 lên t3. Bạn sẽ làm thế nào?"
Giải pháp: Tôi sẽ tạo một Secondary ENI và gắn vào instance t2 hiện tại, sau đó cấu hình ứng dụng nhận MAC của ENI này. Khi nâng cấp, tôi sẽ stop instance t2, detach ENI đó ra. Sau đó tôi khởi tạo instance t3 mới và attach ENI cũ đó vào với đúng MAC address ban đầu.

## 14. References
- Official Docs: https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/using-eni.html
- AWS Blog: Best practices for configuring network interfaces.
- Spec / RFC: IEEE 802.3 (Ethernet) virtual standards.

## 15. Real-world Code
Terraform định nghĩa EC2 với nhiều ENI:
```hcl
resource "aws_network_interface" "multi-eni" {
  subnet_id       = aws_subnet.public.id
  private_ips     = ["10.0.1.50"]
  security_groups = [aws_security_group.web.id]
}

resource "aws_instance" "app" {
  ami           = "ami-0c55b159cbfafe1f0"
  instance_type = "t2.micro"

  network_interface {
    network_interface_id = aws_network_interface.multi-eni.id
    device_index         = 0
  }
}
```

## 16. Community
- Reddit: r/aws - thảo luận về VPC và Networking.
- Stack Overflow: tags [amazon-ec2] [eni].
- Blog: Jeff Barr's AWS Blog.
- Talk: AWS re:Invent: Networking Deep Dive.
