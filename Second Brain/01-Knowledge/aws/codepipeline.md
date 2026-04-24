---
created: 2026-04-24
tags:
  - "#type/concept"
  - "#status/draft"
  - "#lang/devops"
  - "#topic/performance"
related:
  - "[[iam]]"
  - "[[ec2]]"
  - "[[lambda]]"
---

## 1. What
AWS CodePipeline là một dịch vụ phân phối liên tục (Continuous Delivery) được quản lý hoàn toàn, giúp bạn tự động hóa các giai đoạn release phần mềm. Nó điều phối các bước từ khi có thay đổi mã nguồn (Source), qua quá trình xây dựng (Build), kiểm thử (Test) cho đến khi triển khai (Deploy) sản phẩm cuối cùng.

## 2. Why
Việc triển khai phần mềm thủ công thường chậm chạp, dễ gây sai sót và khó kiểm soát phiên bản. CodePipeline ra đời để chuẩn hóa quy trình phát hành, đảm bảo mọi thay đổi đều đi qua các bước kiểm tra nghiêm ngặt một cách tự động, giúp tăng tốc độ đưa tính năng mới đến người dùng (Velocity) và giảm thiểu rủi ro lỗi hệ thống.

## 3. Mental Model
Hãy tưởng tượng CodePipeline như một **Băng chuyền nhà máy tự động**.
- Đầu vào (Source) là các linh kiện (Mã nguồn).
- Băng chuyền đưa linh kiện qua các trạm robot: trạm lắp ráp (Build), trạm kiểm tra chất lượng (Test), và cuối cùng là trạm đóng gói/giao hàng (Deploy).
- Nếu bất kỳ trạm nào phát hiện lỗi, băng chuyền sẽ dừng lại ngay lập tức và báo động cho kỹ sư để khắc phục trước khi sản phẩm lỗi đến tay khách hàng.

## 4. Where it fits
Source (CodeCommit/GitHub/S3) -> **AWS CodePipeline (Orchestrator)** -> Build/Test (CodeBuild) -> Deploy (CodeDeploy/S3/ECS/Lambda).

## 5. When to use
- Tự động hóa hoàn toàn quy trình release từ dev đến production.
- Triển khai ứng dụng lên các dịch vụ AWS như EC2, Lambda, ECS, hoặc S3.
- Cần tích hợp các bước phê duyệt thủ công (Manual Approval) trước khi deploy lên môi trường nhạy cảm.
- Quản lý quy trình release phức tạp với nhiều môi trường (Staging, UAT, Production).

## 6. When NOT to use
- Dự án cực kỳ đơn giản, chỉ cần deploy một lần duy nhất (Manual deploy nhanh hơn).
- Khi bạn đã sử dụng các giải pháp CI/CD tích hợp sẵn của Git provider (như GitHub Actions, GitLab CI/CD) và không có nhu cầu chuyển dịch sang ecosystem của AWS.
- Khi quy trình triển khai yêu cầu các logic điều phối cực kỳ đặc thù mà các Action mặc định của AWS không hỗ trợ.

## 7. Trade-offs
| Pros | Cons |
|------|------|
| Tích hợp sâu và mượt mà với các dịch vụ AWS | Phụ thuộc chặt chẽ vào hệ sinh thái AWS (Vendor lock-in) |
| Trả tiền theo sử dụng (Pay-as-you-go) | UI đôi khi phức tạp và khó cấu hình hơn so với GitHub Actions |
| Hỗ trợ bảo mật mạnh mẽ qua IAM Roles | Một số Action bên thứ ba có thể khó tích hợp hơn |

## 8. Alternatives
| Option | So sánh |
|--------|---------|
| GitHub Actions | Hiện đại, dễ dùng, tích hợp ngay tại nơi lưu trữ code. |
| GitLab CI/CD | Mạnh mẽ, hỗ trợ tốt cho môi trường on-premise. |
| Jenkins | Linh hoạt nhất, cộng đồng plugin khổng lồ nhưng tốn công quản lý server. |

## 9. How
Cấu hình đơn giản qua AWS CLI để lấy trạng thái của một pipeline:
```bash
aws codepipeline get-pipeline-state --name MyWebAppPipeline
```
Hoặc ví dụ cấu hình một Stage trong file CloudFormation:
```yaml
Stages:
  - Name: Source
    Actions:
      - Name: SourceAction
        ActionTypeId:
          Category: Source
          Owner: AWS
          Provider: S3
          Version: '1'
        OutputArtifacts:
          - Name: SourceArtifact
        Configuration:
          S3Bucket: my-source-bucket
          S3ObjectKey: source.zip
```

## 10. Production concerns
### Scaling
CodePipeline tự động mở rộng để xử lý số lượng pipeline và lượt thực thi không giới hạn. Tuy nhiên, tốc độ của pipeline thường bị giới hạn bởi tốc độ của các dịch vụ nó gọi (như CodeBuild).

### Failure
Sử dụng tính năng **Manual Approval** cho các stage quan trọng. Nếu một stage thất bại, pipeline sẽ dừng lại. Bạn có thể cấu hình CloudWatch Events để gửi thông báo qua SNS khi pipeline fail.

### Monitoring
Sử dụng **AWS CloudTrail** để theo dõi ai đã thay đổi cấu hình pipeline và **CloudWatch Metrics** để theo dõi thời gian hoàn thành của mỗi stage.

## 11. Common mistakes
- Mistake: Không sử dụng Manual Approval trước khi deploy lên Production.
  Fix: Luôn thêm một Stage Approval để con người kiểm tra cuối cùng trước khi release.

- Mistake: Lưu trữ secrets (như API keys) trực tiếp trong file cấu hình pipeline.
  Fix: Sử dụng **AWS Secrets Manager** hoặc **Parameter Store** để quản lý thông tin nhạy cảm.

## 12. Sample project
Xây dựng pipeline tự động cho một ứng dụng React: GitHub (Source) -> CodeBuild (Build & Test) -> S3 (Deploy) -> CloudFront (Invalidation). Mỗi khi code được merge vào branch `main`, trang web sẽ tự động cập nhật.

## 13. Interview
### Core Q&A
1. Q: "Artifact" trong CodePipeline là gì?
   A: Artifact là các tệp tin (mã nguồn, file build, cấu hình) được truyền qua lại giữa các Stage của pipeline. Chúng thường được lưu trữ trong một S3 Bucket nội bộ.

### Scenario
1. Q: Làm thế nào để triển khai một thay đổi mã nguồn sang hai Region khác nhau cùng lúc?
   A: Tôi sẽ cấu hình một Stage có hai **Action Deploy** chạy song song, mỗi Action chỉ định một Region đích khác nhau trong cấu hình của nó.

## 14. References
- Official Docs: https://docs.aws.amazon.com/codepipeline/
- Pipeline Concepts: https://docs.aws.amazon.com/codepipeline/latest/userguide/concepts.html

## 15. Real-world Code
Sử dụng **AWS CDK** hoặc **Terraform** để định nghĩa toàn bộ "Pipeline as Code", giúp việc tái sử dụng và quản lý phiên bản quy trình CI/CD trở nên dễ dàng.

## 16. Community
- YouTube: "CI/CD on AWS" - AWS re:Invent sessions.
- Stack Overflow: [aws-codepipeline] tag.
- Blog: AWS DevOps Blog.
