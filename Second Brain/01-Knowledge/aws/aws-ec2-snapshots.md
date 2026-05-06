---
created: 2026-05-05
tags:
  - "#type/concept"
  - "#status/draft"
  - "#lang/devops"
  - "#topic/performance"
related:
  - "[[ec2]]"
  - "[[s3]]"
---

## 1. What
Amazon EBS Snapshots là bản sao lưu tại một thời điểm (point-in-time) của các ổ đĩa Amazon EBS. Các bản sao lưu này được lưu trữ tăng dần (incremental) trên Amazon S3, giúp bảo vệ dữ liệu và hỗ trợ khôi phục sau thảm họa.

## 2. Why
Trước khi có snapshots, việc sao lưu dữ liệu yêu cầu dừng hệ thống hoặc copy thủ công, dễ gây mất dữ liệu hoặc không đồng nhất. Snapshots ra đời để cung cấp cơ chế sao lưu nhanh chóng, tự động và tiết kiệm dung lượng nhờ cơ chế lưu trữ thay đổi (block-level changes).

## 3. Mental Model
Hãy tưởng tượng EBS Snapshot giống như việc chụp một bức ảnh cho cuốn sách đang viết. Lần đầu tiên bạn chụp toàn bộ các trang đã viết. Những lần sau, bạn chỉ chụp những trang có sự thay đổi hoặc viết thêm. Khi cần khôi phục, bạn ghép bức ảnh gốc với các bức ảnh thay đổi để có cuốn sách hoàn chỉnh tại thời điểm đó.

## 4. Where it fits
EBS Volume -> Snapshot (Stored in S3) -> New EBS Volume / AMI.
Cơ chế: EBS -> EBS Snapshot Service -> S3 Managed Storage.

## 5. When to use
- Sao lưu định kỳ dữ liệu quan trọng của hệ thống.
- Tạo bản sao của hệ thống (AMI) để mở rộng (scaling).
- Di chuyển dữ liệu giữa các Availability Zones (AZ) hoặc Regions.
- Thử nghiệm các thay đổi lớn trên hệ thống mà không rủi ro bằng cách tạo snapshot trước khi thực hiện.

## 6. When NOT to use
- Không dùng làm giải pháp lưu trữ dữ liệu truy cập thường xuyên (S3 hoặc EBS là lựa chọn tốt hơn).
- Không dùng cho dữ liệu yêu cầu tính nhất quán cực cao giữa nhiều volumes mà không có cơ chế tạm dừng IO (quiesce).
- Tránh lạm dụng snapshot quá dày đặc cho các volume có tốc độ thay đổi dữ liệu (churn rate) cao vì sẽ làm tăng chi phí S3 nhanh chóng.

## 7. Trade-offs
| Pros | Cons |
|------|------|
| Lưu trữ tăng dần giúp tiết kiệm chi phí | Tốc độ khôi phục (rehydration) lần đầu có thể chậm |
| Dễ dàng tự động hóa với Lifecycle Manager | Rủi ro rò rỉ dữ liệu nếu thiết lập Public Snapshot |
| Hỗ trợ mã hóa dữ liệu (Encryption) | Chi phí tăng cao nếu không dọn dẹp các snapshot cũ |

## 8. Alternatives
| Option | So sánh |
|--------|---------|
| AWS Backup | Quản lý tập trung nhiều resource, hỗ trợ lịch trình phức tạp hơn |
| EBS Multi-Attach | Cho phép nhiều EC2 cùng ghi vào 1 volume thay vì snapshot qua lại |
| Instance Store | Lưu trữ tạm thời tốc độ cao nhưng mất dữ liệu khi stop instance |

## 9. How
```bash
# Tạo snapshot cho một volume cụ thể
aws ec2 create-snapshot \
    --volume-id vol-1234567890abcdef0 \
    --description "Production backup 2026-05-05"

# Kiểm tra trạng thái snapshot
aws ec2 describe-snapshots --snapshot-ids snap-1234567890abcdef0

# Chia sẻ snapshot với account khác một cách an toàn
aws ec2 modify-snapshot-attribute \
    --snapshot-id snap-1234567890abcdef0 \
    --attribute shareResults \
    --operation-type add \
    --user-ids 123456789012
```

## 10. Production concerns
### Scaling
Sử dụng Amazon Data Lifecycle Manager (DLM) để tự động hóa việc tạo và xóa snapshot theo chính sách. Điều này giúp hệ thống tự duy trì mà không cần can thiệp thủ công khi số lượng instance tăng lên.

### Failure
Khi khôi phục từ snapshot, volume mới sẽ có trạng thái "lazy loading" từ S3. Điều này có thể gây độ trễ cho ứng dụng trong những lần truy cập đầu tiên. Cần sử dụng Fast Snapshot Restore (FSR) cho các ứng dụng nhạy cảm về hiệu năng.

### Monitoring
Theo dõi các metrics thông qua CloudWatch. Thiết lập cảnh báo nếu việc tạo snapshot thất bại hoặc nếu dung lượng snapshot tăng đột biến bất thường.

## 11. Common mistakes
- Mistake: Để snapshot ở chế độ Public khiến toàn bộ dữ liệu bí mật (DB, Code, Keys) bị lộ cho bất kỳ ai có tài khoản AWS.
  Fix: Luôn kiểm tra quyền truy cập của snapshot, sử dụng IAM Policy để chặn quyền `ModifySnapshotAttribute` cho Public.

- Mistake: Không mã hóa (encrypt) snapshot chứa dữ liệu nhạy cảm.
  Fix: Kích hoạt mặc định mã hóa EBS tại level account hoặc luôn chọn mã hóa khi tạo snapshot/AMI.

## 12. Sample project
Thiết lập một hệ thống sao lưu tự động cho ứng dụng WordPress chạy trên EC2. Yêu cầu: Mỗi ngày chụp 1 snapshot lúc 2h sáng, chỉ giữ lại bản sao lưu của 7 ngày gần nhất, và tự động copy snapshot sang một Region khác để phòng trường hợp Region chính gặp sự cố.

## 13. Interview
### Core Q&A
1. Q: EBS Snapshot được lưu trữ ở đâu và cơ chế tính phí thế nào?
   A: Snapshot được lưu trữ trên Amazon S3 nhưng bạn không thể thấy chúng trong S3 bucket của mình. Phí được tính dựa trên dung lượng dữ liệu thay đổi giữa các lần snapshot (incremental) chứ không phải toàn bộ dung lượng của volume gốc.
2. Q: Làm thế nào để đảm bảo tính nhất quán của dữ liệu khi chụp snapshot cho một database đang chạy?
   A: Tốt nhất nên tạm dừng các thao tác ghi (flush & lock) hoặc unmount volume trước khi chụp. Nếu không thể dừng, snapshot vẫn sẽ diễn ra (crash-consistent) nhưng có thể yêu cầu kiểm tra toàn vẹn khi khôi phục.

### Scenario
Hệ thống của bạn bị tấn công Ransomware và dữ liệu trên EBS đã bị mã hóa. Bạn có snapshot từ 4 tiếng trước. Hãy trình bày các bước khôi phục nhanh nhất để giảm thiểu Downtime?
Bước 1: Cô lập instance bị nhiễm. Bước 2: Tạo Volume mới từ snapshot gần nhất (ưu tiên dùng FSR nếu có). Bước 3: Đổi tên volume cũ và mount volume mới vào instance sạch. Bước 4: Kiểm tra dữ liệu và khởi động lại dịch vụ.

## 14. References
- Official Docs: https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/EBSSnapshots.html
- GitHub Repo: https://github.com/aws/aws-cli
- Spec / RFC: N/A
- Changelog: AWS DLM updates for cross-region copy.

## 15. Real-world Code
https://github.com/toniblyx/prowler (Công cụ audit bảo mật AWS, kiểm tra các public snapshots)

## 16. Community
- Reddit: r/aws
- Stack Overflow: Tag #amazon-ebs #amazon-ec2
- Blog: AWS Architecture Blog
- Talk: AWS re:Invent sessions on Backup & Recovery techniques.
