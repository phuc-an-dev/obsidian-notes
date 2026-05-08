---
created: 2026-05-07
tags:
  - "#type/concept"
  - "#status/draft"
  - "#lang/database"
  - "#topic/performance"
related:
  - "[[ec2]]"
  - "[[rds]]"
  - "[[aws-rds-public-accessible]]"
  - "[[MySQL JDBC Driver Deprecation.md]]"
---

## 1. What
Đây là sự so sánh giữa hai phương pháp triển khai cơ sở dữ liệu MySQL trên AWS: **Self-managed MySQL on EC2** (tự cài đặt và quản lý trên máy ảo) và **AWS RDS for MySQL** (dịch vụ cơ sở dữ liệu có quản lý). Việc chọn MySQL trên EC2 thường hướng tới mục tiêu tối ưu hóa chi phí và quyền kiểm soát tối đa.

## 2. Why
AWS RDS rất tiện lợi nhưng đi kèm với một khoản phí "quản lý" đáng kể (thường đắt hơn 30-50% so với EC2 có cùng cấu hình). Đối với các startup nhỏ, các dự án cá nhân hoặc môi trường development, việc tự cài đặt MySQL trên EC2 giúp:
- Tiết kiệm chi phí vận hành hàng tháng.
- Tận dụng được các instance nhỏ (như t3.nano/micro) mà RDS đôi khi không hỗ trợ linh hoạt.
- Có toàn quyền cấu hình sâu vào hệ điều hành và file system.

## 3. Mental Model
Hãy tưởng tượng việc sở hữu một chiếc xe:
- **MySQL trên EC2** giống như **tự mua xe và tự bảo trì**: Bạn phải tự thay dầu, tự sửa phanh, tự lo chỗ đỗ. Chi phí ban đầu và hàng tháng rẻ hơn, nhưng bạn tốn công sức và thời gian (vận hành).
- **AWS RDS** giống như **thuê xe có lái (Grab/Uber)**: Bạn chỉ cần ngồi lên và đi. Tài xế lo mọi thứ từ xăng xe, bảo trì đến an toàn. Tiện lợi vô cùng nhưng mỗi chuyến đi đều đắt hơn nhiều.

## 4. Where it fits
- **MySQL on EC2**: Infrastructure as a Service (IaaS).
- **AWS RDS**: Platform as a Service (PaaS).

## 5. When to use
### MySQL on EC2:
- Khi ngân sách cực kỳ hạn hẹp và bạn có kỹ năng quản trị Linux/DB.
- Cần cấu hình MySQL đặc thù mà RDS không cho phép (ví dụ: cài thêm plugin ngoài, sửa tham số OS).
- Dữ liệu không quá quan trọng hoặc đã có quy trình backup tự động riêng.

### AWS RDS:
- Khi ứng dụng là Production quan trọng, yêu cầu độ sẵn sàng cao (Multi-AZ).
- Team thiếu nhân sự quản trị hệ thống (Database Administrator).
- Cần các tính năng như tự động backup, point-in-time recovery mà không muốn tự viết script.

## 6. When NOT to use
- Đừng dùng MySQL trên EC2 cho Production nếu bạn không biết cách cấu hình backup và giám sát (Monitoring). Rủi ro mất dữ liệu là rất cao.
- Đừng dùng RDS cho các dự án "chạy xong bỏ" hoặc lab cá nhân nếu không muốn nhận hóa đơn bất ngờ vào cuối tháng.

## 7. Trade-offs
| Tiêu chí | MySQL on EC2 | AWS RDS |
|------|------|------|
| **Chi phí** | Rẻ nhất (chỉ trả tiền EC2 + EBS) | Đắt (bao gồm phí quản lý) |
| **Quản trị** | Vất vả (Tự cài, tự patch, tự backup) | Nhàn (AWS lo hết) |
| **Scaling** | Khó (Downtime khi đổi instance) | Dễ (Vài click chuột) |
| **Bảo mật** | Bạn tự chịu trách nhiệm | AWS hỗ trợ tận răng (KMS, IAM Auth) |

## 8. Alternatives
| Option | So sánh |
|--------|---------|
| AWS Aurora | Cao cấp hơn RDS, hiệu năng gấp 5 lần MySQL thường nhưng chi phí cao hơn. |
| Lightsail Database | Đơn giản hơn RDS, giá cố định hàng tháng, phù hợp dự án nhỏ. |
| Docker on EC2 | Dễ triển khai hơn cài trực tiếp trên OS, dễ di chuyển. |

## 9. How
### Cách tiết kiệm nhất với MySQL trên EC2:
1. Sử dụng **Reserved Instances** hoặc **Savings Plans** để giảm tới 72% chi phí EC2.
2. Sử dụng **Spot Instances** cho môi trường dev/test (rẻ hơn 90%).
3. Sử dụng **EBS Cold HDD (sc1)** hoặc **Throughput Optimized (st1)** cho dữ liệu ít truy cập (mặc dù SSD gp3 vẫn là khuyến nghị tốt nhất cho DB).
4. Thiết lập Script Backup vào S3 (S3 có chi phí lưu trữ cực rẻ so với EBS Snapshots).

Lệnh cài đặt cơ bản trên Ubuntu:
```bash
sudo apt update
sudo apt install mysql-server
# Sau đó chạy script bảo mật
sudo mysql_secure_installation
```

## 10. Production concerns
### Backup
Trên EC2, bạn phải tự cài `cron` để chạy `mysqldump` và đẩy file lên S3 định kỳ.
### High Availability
RDS có Multi-AZ tự động. Trên EC2, bạn phải tự cấu hình Replication (Master-Slave) và dùng công cụ như Orchestrator hoặc Keepalived.

## 11. Common mistakes
- Mistake: Để MySQL trên EC2 mở port 3306 cho `0.0.0.0/0`.
  Fix: Chỉ cho phép IP của ứng dụng hoặc dùng Security Group Referencing.

- Mistake: Không giới hạn dung lượng Log file, dẫn đến đầy ổ cứng EC2.
  Fix: Cấu hình log rotation cho MySQL logs.

## 12. Sample project
Thiết lập một hệ thống MySQL trên EC2 t3.micro:
1. Cài đặt MySQL 8.0.
2. Viết shell script thực hiện `mysqldump`.
3. Dùng AWS CLI đẩy bản dump lên S3 Bucket hàng ngày.
4. Thiết lập Lifecycle Policy trên S3 để xóa bản backup cũ sau 30 ngày.

## 13. Interview
### Core Q&A
1. Q: Tại sao RDS lại đắt hơn EC2?
   A: Vì bạn trả tiền cho sự tự động hóa: Backup, Patching, Monitoring, High Availability, và sự hỗ trợ từ đội ngũ chuyên gia của AWS.

2. Q: Làm thế nào để giảm downtime khi nâng cấp RAM/CPU cho MySQL trên EC2?
   A: Rất khó để tránh downtime hoàn toàn. Bạn phải tắt instance, đổi type, và bật lại. Với RDS, quá trình này diễn ra mượt mà hơn nhiều.

### Scenario
"Công ty đang dùng RDS và tốn 500$/tháng. Sếp muốn cắt giảm chi phí này xuống còn 100$. Bạn đề xuất gì?"
-> Trả lời: 
1. Kiểm tra xem có thể dùng Reserved Instance cho RDS không.
2. Nếu vẫn đắt, cân nhắc chuyển sang MySQL trên EC2 dùng t3 instances và Savings Plans.
3. Chấp nhận đánh đổi về việc team phải tự vận hành backup và không còn tính năng tự động recovery của RDS.

## 14. References
- AWS Blog: [Choosing between RDS and EC2](https://aws.amazon.com/blogs/database/choosing-between-amazon-rds-and-amazon-ec2-for-database-workloads/)
- MySQL Docs: [Installing MySQL on Linux](https://dev.mysql.com/doc/refman/8.0/en/linux-installation.html)

## 15. Real-world Code
Mẫu Script Backup MySQL lên S3 đơn giản:
```bash
#!/bin/bash
DATE=$(date +%Y%m%d%H%M)
FILENAME="backup-$DATE.sql.gz"
mysqldump -u root -p'password' --all-databases | gzip > /tmp/$FILENAME
aws s3 cp /tmp/$FILENAME s3://my-backup-bucket/mysql/
rm /tmp/$FILENAME
```

## 16. Community
- Reddit: r/aws - thảo luận "RDS vs EC2 for small startups".
- Stack Overflow: "Cost optimization for MySQL on AWS".
