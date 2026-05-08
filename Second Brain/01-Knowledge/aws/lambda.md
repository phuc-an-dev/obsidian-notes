---
created: 2026-04-24
tags:
  - "#type/concept"
  - "#status/draft"
  - "#lang/devops"
  - "#topic/async"
related:
  - "[[ec2]]"
  - "[[s3]]"
  - "[[Image Cropping with AWS Lambda and S3]]"
---

## 1. What
AWS Lambda là một dịch vụ tính toán không máy chủ (Serverless compute service) cho phép bạn chạy mã mà không cần cấp phát hoặc quản lý máy chủ. Lambda tự động thực thi mã của bạn để phản hồi các sự kiện và tự động quản lý tài nguyên tính toán cho bạn.

## 2. Why
Trước khi có Lambda, để chạy một đoạn mã nhỏ, bạn vẫn phải duy trì một máy chủ 24/7 (tốn tiền và công quản lý). Lambda ra đời để bạn chỉ phải trả tiền khi mã thực sự chạy (pay-per-invocation), giúp tối ưu hóa chi phí và giảm bớt gánh nặng vận hành hạ tầng.

## 3. Mental Model
Hãy tưởng tượng Lambda như một "vòi nước tự động". Bạn không cần phải xây dựng và bảo trì một bể chứa nước (server). Khi có người đưa tay vào (Sự kiện/Event), vòi nước tự động chảy (Chạy code). Khi người đó rút tay ra, nước ngừng chảy và bạn không tốn thêm giọt nước nào. Bạn chỉ trả tiền cho lượng nước đã dùng.

## 4. Where it fits
Event Source (S3, API Gateway, DynamoDB) -> **AWS Lambda Function** -> Destination (Database, Logs, SNS).

## 5. When to use
- Xử lý tệp tin ngay khi upload (ví dụ: resize ảnh trên S3).
- Xây dựng Backend cho ứng dụng web/mobile (kết hợp với API Gateway).
- Các tác vụ tự động hóa hạ tầng (ví dụ: tự động tắt EC2 vào ban đêm).
- Xử lý luồng dữ liệu thời gian thực (Kinesis, DynamoDB Streams).
- Gửi email/thông báo dựa trên sự kiện người dùng.

## 6. When NOT to use
- Các tác vụ chạy quá lâu (giới hạn tối đa là 15 phút).
- Ứng dụng yêu cầu duy trì trạng thái (stateful) cực kỳ phức tạp (nên dùng máy chủ truyền thống hoặc database).
- Các hệ thống yêu cầu độ trễ cực thấp và ổn định tuyệt đối (vấn đề "Cold Start").
- Ứng dụng yêu cầu tùy chỉnh sâu vào hệ điều hành hoặc phần cứng.

## 7. Trade-offs
| Pros | Cons |
|------|------|
| Không cần quản lý máy chủ | Giới hạn thời gian chạy (15 phút) |
| Tự động mở rộng (Scaling) cực nhanh | Vấn đề Cold Start (trễ ở lần chạy đầu tiên) |
| Chi phí cực thấp cho traffic thấp/vừa | Khó debug cục bộ hơn so với server truyền thống |

## 8. Alternatives
| Option | So sánh |
|--------|---------|
| AWS Fargate | Serverless container, tốt cho task chạy lâu hơn và phức tạp hơn. |
| Google Cloud Functions | Giải pháp tương đương từ Google Cloud. |
| Azure Functions | Giải pháp serverless từ Microsoft Azure. |

## 9. How
Một ví dụ về Lambda function đơn giản bằng Node.js:
```javascript
// index.js
export const handler = async (event) => {
    const name = event.name || "World";
    const response = {
        statusCode: 200,
        body: JSON.stringify(`Hello, ${name} from Lambda!`),
    };
    return response;
};
```

## 10. Production concerns
### Scaling
Lambda tự động scale bằng cách chạy thêm các "concurrent executions". Cần lưu ý giới hạn mặc định (thường là 1000) và cấu hình "Reserved Concurrency" cho các hàm quan trọng.

### Failure
Cấu hình Dead Letter Queues (DLQ) bằng SQS hoặc SNS để lưu trữ các sự kiện bị lỗi để xử lý sau (reprocess).

### Monitoring
Sử dụng **AWS CloudWatch Logs** và **X-Ray** để theo dõi lỗi, độ trễ và vết thực thi của các hàm Lambda.

## 11. Common mistakes
- Mistake: Thực hiện các kết nối database đắt đỏ (như tạo connection pool mới) bên trong handler.
  Fix: Khai báo các biến kết nối bên ngoài handler để tận dụng lại ở các lần gọi sau (Execution Context Reuse).

- Mistake: Để hàm Lambda chạy quá 15 phút mà không có cơ chế timeout/retry.
  Fix: Chia nhỏ các task lớn thành nhiều Lambda nhỏ hoặc dùng AWS Step Functions.

## 12. Sample project
Xây dựng một hệ thống xử lý video: User upload video lên S3 -> Kích hoạt Lambda để kiểm tra định dạng -> Gửi thông tin vào một SQS queue để MediaConvert xử lý -> Lambda cuối cùng gửi email thông báo cho user khi hoàn thành.

## 13. Interview
### Core Q&A
1. Q: "Cold Start" là gì và làm thế nào để giảm thiểu nó?
   A: Cold Start là độ trễ khi AWS phải cấp phát tài nguyên mới để chạy code của bạn sau một thời gian không sử dụng. Có thể giảm thiểu bằng cách dùng "Provisioned Concurrency", tăng dung lượng RAM (giúp tăng CPU tỉ lệ thuận), hoặc giữ cho gói code (deployment package) nhỏ nhất có thể.

### Scenario
1. Q: Làm thế nào để Lambda truy cập vào một database nằm trong một private subnet của VPC?
   A: Tôi sẽ cấu hình Lambda để kết nối vào VPC đó, chỉ định các Subnets và Security Groups phù hợp. Lambda sẽ được cấp một Network Interface (ENI) để giao tiếp nội bộ.

## 14. References
- Official Docs: https://docs.aws.amazon.com/lambda/
- Lambda Limits: https://docs.aws.amazon.com/lambda/latest/dg/gettingstarted-limits.html
- Best Practices: https://docs.aws.amazon.com/lambda/latest/dg/best-practices.html

## 15. Real-world Code
Nghiên cứu sử dụng **AWS SAM (Serverless Application Model)** hoặc **Serverless Framework** để quản lý và triển khai các ứng dụng Lambda quy mô lớn.

## 16. Community
- Podcast: Serverless Chats.
- YouTube: "Lambda Performance Optimization" bởi AWS re:Invent sessions.
- Twitter: #Serverless #AWSLambda.
