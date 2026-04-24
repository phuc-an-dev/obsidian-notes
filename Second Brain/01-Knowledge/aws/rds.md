---
created: 2026-04-24
tags:
  - "#type/concept"
  - "#status/draft"
  - "#lang/devops"
  - "#topic/database"
related:
  - "[[ec2]]"
  - "[[s3]]"
---

## 1. What
AWS RDS (Relational Database Service) là một dịch vụ cơ sở dữ liệu quan hệ được quản lý (managed service) giúp bạn dễ dàng thiết lập, vận hành và mở rộng các cơ sở dữ liệu phổ biến như MySQL, PostgreSQL, MariaDB, Oracle và SQL Server trên đám mây.

## 2. Why
Trước khi có RDS, quản trị viên database (DBA) phải tự cài đặt OS, cài đặt Database engine, tự cấu hình backup, tự quản lý việc vá lỗi (patching) và thiết lập High Availability thủ công. RDS ra đời để tự động hóa các tác vụ quản trị lặp đi lặp lại này, cho phép lập trình viên tập trung vào ứng dụng và dữ liệu.

## 3. Mental Model
Hãy tưởng tượng RDS như việc bạn thuê một đầu bếp riêng cho gia đình. Thay vì bạn phải tự đi chợ, sơ chế, nấu nướng và dọn dẹp (tự quản lý DB trên EC2), bạn chỉ cần order món bạn muốn (Database Engine, RAM, Storage). Đầu bếp sẽ lo liệu việc nấu nướng, đảm bảo vệ sinh (vá lỗi bảo mật) và luôn có món ăn sẵn sàng ngay cả khi một nguyên liệu nào đó bị thiếu (High Availability).

## 4. Where it fits
Application Server (EC2/Lambda) -> JDBC/ODBC Connection -> **AWS RDS Instance** -> EBS Volume (Storage) -> S3 (Automated Backups).

## 5. When to use
- Khi ứng dụng cần một cơ sở dữ liệu quan hệ tiêu chuẩn (SQL).
- Khi muốn giảm bớt gánh nặng quản trị hạ tầng (Backups, Patching).
- Khi cần tính năng High Availability (Multi-AZ) và Read Replicas một cách dễ dàng.
- Các ứng dụng doanh nghiệp yêu cầu sự ổn định và tuân thủ bảo mật.

## 6. When NOT to use
- Khi ứng dụng cần toàn quyền truy cập vào hệ điều hành của máy chủ database (nên dùng EC2).
- Khi dữ liệu không có cấu trúc và yêu cầu khả năng mở rộng ngang cực lớn (nên dùng DynamoDB).
- Khi ứng dụng chỉ cần một database siêu nhỏ cho mục đích test (có thể dùng SQLite hoặc Docker).

## 7. Trade-offs
| Pros | Cons |
|------|------|
| Tự động hóa Backup và Vá lỗi | Chi phí cao hơn so với tự cài trên EC2 |
| Dễ dàng triển khai Multi-AZ | Giới hạn quyền truy cập vào OS/File system |
| Hỗ trợ Read Replicas để tăng performance đọc | Khó tùy chỉnh sâu các file config hệ thống |

## 8. Alternatives
| Option | So sánh |
|--------|---------|
| Amazon Aurora | Database được AWS tối ưu riêng, nhanh hơn và tin cậy hơn RDS thường. |
| Self-managed on EC2 | Rẻ hơn, linh hoạt nhất nhưng tốn công quản trị cực lớn. |
| PlanetScale / Supabase | Các giải pháp Serverless DB hiện đại hơn, dễ scale hơn cho Web. |

## 9. How
Kết nối đến RDS PostgreSQL instance từ ứng dụng Node.js:
```javascript
const { Client } = require('pg');

const client = new Client({
  host: 'my-rds-db.cxyz123.us-east-1.rds.amazonaws.com',
  user: 'admin',
  password: 'my-secure-password',
  database: 'mydb',
  port: 5432,
  ssl: { rejectUnauthorized: false } // Khuyến nghị dùng SSL cho Production
});

await client.connect();
```

## 10. Production concerns
### Scaling
Có hai cách: **Vertical Scaling** (tăng RAM/CPU của instance) và **Horizontal Scaling** (thêm Read Replicas để chia sẻ tải lượng đọc).

### Failure
Sử dụng **Multi-AZ Deployment**. AWS sẽ tự động tạo một bản sao dự phòng (Standby) ở một Availability Zone khác. Nếu bản chính (Primary) chết, AWS sẽ tự động chuyển hướng (failover) sang bản dự phòng chỉ trong vài giây.

### Monitoring
Theo dõi CPU, Memory, Free Storage Space và **Database Connections** thông qua Amazon CloudWatch. Sử dụng Performance Insights để tìm các câu query chậm.

## 11. Common mistakes
- Mistake: Để RDS instance ở public subnet và mở port 3306/5432 cho toàn bộ internet.
  Fix: Luôn đặt RDS trong Private Subnet và chỉ cho phép Security Group của App Server truy cập.

- Mistake: Không kiểm tra dung lượng lưu trữ định kỳ dẫn đến DB bị treo do đầy ổ cứng.
  Fix: Bật tính năng "Storage Autoscaling" để RDS tự động tăng dung lượng khi cần.

## 12. Sample project
Xây dựng một hệ thống Blog: Web server chạy trên EC2 kết nối với RDS MySQL. Cấu hình Multi-AZ để đảm bảo blog không bị sập và dùng Read Replica để phục vụ hàng triệu lượt đọc bài viết mỗi ngày.

## 13. Interview
### Core Q&A
1. Q: Read Replica và Multi-AZ khác nhau như thế nào?
   A: Multi-AZ dùng cho **High Availability** (dự phòng lỗi, bản standby không thể đọc/ghi). Read Replica dùng cho **Scalability** (tăng hiệu năng đọc, có thể truy cập để query dữ liệu).

### Scenario
1. Q: Bạn làm gì nếu CPU của RDS liên tục ở mức 90%?
   A: Đầu tiên tôi sẽ dùng Performance Insights để tìm các câu query chưa được đánh index (Long running queries). Nếu query đã tối ưu mà vẫn cao, tôi sẽ cân nhắc nâng cấp Instance Class hoặc thêm Read Replicas nếu tải chủ yếu là đọc.

## 14. References
- Official Docs: https://aws.amazon.com/rds/
- Multi-AZ Explanations: https://aws.amazon.com/rds/features/multi-az/
- Security Best Practices: https://docs.aws.amazon.com/AmazonRDS/latest/UserGuide/CHAP_BestPractices.Security.html

## 15. Real-world Code
Sử dụng Terraform để định nghĩa RDS instance kèm theo Security Group và Subnet Group để quản lý hạ tầng một cách chuyên nghiệp.

## 16. Community
- YouTube: "AWS RDS Masterclass" - AWS Training.
- Stack Overflow: [amazon-rds] tag.
- Blog: AWS Database Blog.
