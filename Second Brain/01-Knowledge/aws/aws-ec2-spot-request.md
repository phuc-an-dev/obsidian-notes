---
created: 2026-05-05
tags:
  - "#type/concept"
  - "#status/draft"
  - "#lang/devops"
  - "#topic/compute"
related:
  - "[[aws-ec2-instance]]"
---

## 1. What
Spot Request là một yêu cầu mua tài nguyên tính toán EC2 dư thừa của AWS với giá rẻ hơn rất nhiều (giảm tới 90%) so với giá On-Demand. Tuy nhiên, AWS có quyền thu hồi các instance này bất cứ lúc nào khi họ cần lại tài nguyên cho người dùng On-Demand.

## 2. Why
AWS luôn có một lượng lớn tài nguyên server nhàn rỗi. Để tối ưu hóa doanh thu, họ bán rẻ lượng tài nguyên này. Đối với người dùng, đây là giải pháp tối ưu chi phí cực mạnh cho các tác vụ không yêu cầu tính liên tục tuyệt đối hoặc có khả năng chịu lỗi cao.

## 3. Mental Model
Hãy tưởng tượng bạn đi mua vé máy bay giờ chót (standby ticket). Bạn mua được vé với giá cực rẻ, nhưng bạn chỉ được lên máy bay nếu còn chỗ trống. Nếu có hành khách mua vé hạng thương gia (On-Demand) đến, bạn có thể phải nhường chỗ và đợi chuyến sau.

## 4. Where it fits
Cost Optimization -> EC2 Purchasing Options. 
Spot Instances nằm cùng tầng với On-Demand và Reserved Instances nhưng khác biệt về cơ chế đấu giá và tính ổn định.

## 5. When to use
- Các tác vụ xử lý dữ liệu lớn (Big Data, Batch Processing).
- Môi trường CI/CD, chạy test tự động.
- Render video, xử lý ảnh.
- Các ứng dụng stateless, có khả năng tự phục hồi khi một node bị sập.

## 6. When NOT to use
- Database (trừ khi có cơ chế replication cực tốt).
- Các ứng dụng stateful yêu cầu duy trì kết nối liên tục.
- Các tác vụ quan trọng không thể bị gián đoạn (Critical workloads).
- Môi trường Production cho khách hàng cuối mà không có lớp On-Demand dự phòng.

## 7. Trade-offs
| Pros | Cons |
|------|------|
| Tiết kiệm chi phí lên đến 70-90%. | Có thể bị thu hồi bất ngờ (Interruption). |
| Quy mô lớn: Có thể chạy hàng nghìn instance giá rẻ. | Không đảm bảo thời gian sống của instance. |
| Giảm lãng phí tài nguyên của cloud. | Cần kiến trúc ứng dụng phức tạp hơn để xử lý gián đoạn. |

## 8. Alternatives
| Option | So sánh |
|--------|---------|
| On-Demand | Đắt nhất nhưng đảm bảo 100% không bị thu hồi bởi AWS. |
| Reserved Instances | Tiết kiệm 30-70% nếu cam kết dùng 1-3 năm, ổn định tuyệt đối. |
| Savings Plans | Linh hoạt hơn RI nhưng vẫn yêu cầu cam kết sử dụng lâu dài. |

## 9. How
```bash
# Tạo một Spot Instance Request đơn giản qua AWS CLI
aws ec2 request-spot-instances \
    --spot-price "0.03" \
    --instance-count 1 \
    --type "one-time" \
    --launch-specification '{
        "ImageId": "ami-0abcdef1234567890",
        "InstanceType": "t3.medium",
        "KeyName": "my-key-pair",
        "SecurityGroupIds": ["sg-12345678"]
    }'
```

## 10. Production concerns
### Scaling
Sử dụng Spot Fleet hoặc Auto Scaling Group với "Capacity Rebalancing" để tự động thay thế các Spot instance sắp bị thu hồi bằng các instance mới.

### Failure
Xử lý Spot Interruption Notice: AWS cung cấp cảnh báo trước 2 phút qua CloudWatch Events hoặc Instance Metadata Service. Ứng dụng cần tận dụng 2 phút này để lưu state hoặc drain traffic.

### Monitoring
Theo dõi "Spot Interruption Rate" trong AWS Console để chọn loại Instance Type có độ ổn định cao nhất trong vùng (AZ) đó.

## 11. Common mistakes
- Mistake: Chọn một loại Instance Type duy nhất cho Spot Fleet.
  Fix: Nên đa dạng hóa (Diversify) nhiều loại Instance Type và nhiều AZ để giảm tỷ lệ toàn bộ fleet bị sập cùng lúc.

- Mistake: Đặt giá Spot quá cao (cao hơn giá On-Demand).
  Fix: AWS sẽ tính giá Spot hiện tại chứ không tính giá bạn đặt, nhưng đặt quá cao không giúp bạn giữ instance lâu hơn nếu AWS thực sự hết tài nguyên.

## 12. Sample project
Thiết lập một cụm Kubernetes (EKS) sử dụng Spot Instances cho Worker Nodes, kết hợp với công cụ "Karpenter" để tự động quản lý và tối ưu hóa việc sử dụng Spot.

## 13. Interview
### Core Q&A
1. Q: Spot Interruption là gì và bạn xử lý nó như thế nào?
   A: Là việc AWS lấy lại instance. Xử lý bằng cách lắng nghe cảnh báo 2 phút, dùng checkpointing để lưu tiến trình công việc, và dùng ASG để tự động yêu cầu instance mới.

2. Q: Sự khác biệt giữa Spot Request "one-time" và "persistent" là gì?
   A: One-time sẽ biến mất sau khi instance bị tắt hoặc bị thu hồi. Persistent sẽ tự động mở lại một instance mới khi có giá phù hợp hoặc có tài nguyên trống.

3. Q: Làm sao để tối ưu khả năng sống sót của Spot Instance?
   A: Sử dụng chiến lược "capacity-optimized" thay vì "lowest-price" để chọn những pool instance có xác suất bị thu hồi thấp nhất.

### Scenario
Bạn cần chạy một job xử lý 10TB dữ liệu trong 5 tiếng. Ngân sách cực hạn hẹp. Bạn chọn phương án nào?
Trả lời: Sử dụng Spot Fleet với cơ chế đa dạng hóa Instance Type. Chia nhỏ dữ liệu thành các chunk và lưu kết quả trung gian (checkpoint) vào S3. Nếu một node bị thu hồi, node mới sẽ tiếp tục từ chunk chưa hoàn thành.

## 14. References
- Official Docs: https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/using-spot-instances.html
- GitHub Repo: N/A
- Spec / RFC: N/A
- Changelog: N/A

## 15. Real-world Code
Sử dụng Terraform `aws_spot_instance_request` để quản lý hạ tầng Spot theo code.

## 16. Community
- Reddit: r/aws - thảo luận về các chiến thuật dùng Spot hiệu quả.
- Stack Overflow: Cách handle termination signal trên Spot instance.
- Blog: Jeff Barr's blog về Spot Instance cải tiến.
- Talk: re:Invent sessions: "Deep dive into Amazon EC2 Spot Instances".
