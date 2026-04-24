---
created: 2026-04-24
tags:
  - "#type/concept"
  - "#status/draft"
  - "#lang/devops"
  - "#topic/async"
related:
  - "[[lambda]]"
  - "[[s3]]"
---

## 1. What
AWS SQS (Simple Queue Service) là một dịch vụ hàng đợi tin nhắn được quản lý hoàn toàn (fully managed message queuing service) cho phép bạn tách rời và mở rộng quy mô các microservices, hệ thống phân tán và ứng dụng serverless. SQS loại bỏ sự phức tạp và chi phí vận hành liên quan đến việc quản lý và vận hành phần mềm middleware hàng đợi tin nhắn.

## 2. Why
Trong kiến trúc truyền thống, nếu Component A gọi trực tiếp Component B (Synchronous), và Component B bị sập hoặc quá tải, Component A cũng sẽ thất bại. SQS ra đời để làm "vùng đệm" ở giữa, cho phép các thành phần giao tiếp bất đồng bộ, giúp hệ thống chịu lỗi tốt hơn và có thể xử lý các đỉnh traffic đột ngột mà không làm sập backend.

## 3. Mental Model
Hãy tưởng tượng SQS như một cái **Hòm thư** tại bưu điện.
- Người gửi (Producer) chỉ cần bỏ thư vào hòm và đi làm việc khác. Họ không cần biết khi nào thư được đọc.
- Người nhận (Consumer) sẽ ra hòm thư để lấy thư về xử lý khi họ rảnh.
- Nếu có quá nhiều thư cùng lúc, hòm thư sẽ giữ chúng lại an toàn cho đến khi người nhận xử lý hết. Thư không bao giờ bị mất nếu người nhận tạm thời đi vắng.

## 4. Where it fits
Producer (EC2/Lambda) -> SendMessage -> **AWS SQS Queue** -> ReceiveMessage -> Consumer (EC2/Lambda) -> DeleteMessage.

## 5. When to use
- Tách rời (Decoupling) các thành phần ứng dụng để chúng hoạt động độc lập.
- Xử lý các tác vụ nền (Background jobs) tốn thời gian như gửi email, resize ảnh, tạo báo cáo.
- Điều tiết tải (Load leveling/Buffering) để bảo vệ backend khỏi các đợt bùng nổ traffic.
- Triển khai mô hình Fan-out khi kết hợp với AWS SNS.

## 6. When NOT to use
- Khi yêu cầu phản hồi ngay lập tức (Real-time synchronous communication).
- Khi cần chia sẻ dữ liệu lớn (Message size giới hạn ở 256KB, nếu lớn hơn nên dùng S3 và gửi link qua SQS).
- Khi cần duy trì trạng thái phiên làm việc (Session state) giữa người dùng và server.

## 7. Trade-offs
| Pros | Cons |
|------|------|
| Khả năng mở rộng gần như vô hạn | Độ trễ (Latency) tăng do giao tiếp bất đồng bộ |
| Độ tin cậy cực cao (Dữ liệu được lưu trữ trên nhiều AZ) | Tin nhắn có thể bị nhận lặp lại (Standard Queue) |
| Không cần quản lý máy chủ hàng đợi | Giới hạn kích thước tin nhắn (256KB) |

## 8. Alternatives
| Option | So sánh |
|--------|---------|
| AWS Kinesis | Tốt hơn cho luồng dữ liệu lớn (Big Data), cho phép nhiều consumer đọc cùng một dữ liệu. |
| RabbitMQ | Mạnh mẽ hơn về routing, nhưng phải tự quản lý hoặc dùng bản Managed. |
| Apache Kafka | Hiệu năng cực cao, phù hợp cho log aggregation và stream processing quy mô lớn. |

## 9. How
Gửi tin nhắn vào SQS bằng Node.js (AWS SDK v3):
```javascript
import { SQSClient, SendMessageCommand } from "@aws-sdk/client-sqs";

const client = new SQSClient({ region: "us-east-1" });

const sendMessage = async () => {
  const command = new SendMessageCommand({
    QueueUrl: "https://sqs.us-east-1.amazonaws.com/123456789012/MyQueue",
    MessageBody: JSON.stringify({ orderId: 123, status: "pending" }),
    DelaySeconds: 10,
  });

  const response = await client.send(command);
  console.log("Message Sent, ID:", response.MessageId);
};
```

## 10. Production concerns
### Scaling
SQS Standard queues hỗ trợ số lượng tin nhắn gần như không giới hạn mỗi giây. SQS FIFO có giới hạn thấp hơn (300-3000 req/s tùy cấu hình) nhưng đảm bảo thứ tự.

### Failure
Sử dụng **Dead Letter Queues (DLQ)** để hứng các tin nhắn không thể xử lý sau nhiều lần thử (retry). Điều này giúp ngăn chặn "Poison messages" làm nghẽn hàng đợi chính.

### Monitoring
Theo dõi các chỉ số quan trọng trong CloudWatch: `ApproximateNumberOfMessagesVisible` (để biết hàng đợi có đang bị ứ đọng không) và `NumberOfMessagesSent/Received`.

## 11. Common mistakes
- Mistake: Không xóa tin nhắn sau khi xử lý xong (DeleteMessage).
  Fix: SQS không tự xóa tin nhắn sau khi nhận. Bạn phải xóa thủ công để tin nhắn không xuất hiện lại trong hàng đợi sau khi hết `Visibility Timeout`.

- Mistake: Để `Visibility Timeout` quá ngắn so với thời gian xử lý thực tế của Consumer.
  Fix: Đảm bảo `Visibility Timeout` đủ dài để Consumer hoàn thành công việc trước khi tin nhắn trở nên "visible" cho các consumer khác.

## 12. Sample project
Hệ thống đặt hàng: API nhận đơn hàng -> Gửi vào SQS -> Một Lambda function làm Consumer đọc từ SQS để trừ kho và gửi email. Nếu email server sập, tin nhắn vẫn nằm trong SQS và sẽ được thử lại sau.

## 13. Interview
### Core Q&A
1. Q: Sự khác biệt giữa Standard Queue và FIFO Queue là gì?
   A: Standard đảm bảo thông lượng tối đa nhưng không hứa về thứ tự và có thể có tin nhắn lặp lại (at-least-once). FIFO đảm bảo thứ tự chính xác (First-In-First-Out) và chỉ nhận một lần duy nhất (exactly-once) nhưng thông lượng thấp hơn.

### Scenario
1. Q: Làm thế nào để xử lý các tin nhắn lỗi mà không làm mất dữ liệu?
   A: Tôi sẽ cấu hình một **Dead Letter Queue (DLQ)**. Khi một tin nhắn thất bại vượt quá số lần retry cho phép (`maxReceiveCount`), SQS sẽ tự động chuyển nó sang DLQ. Sau đó tôi có thể kiểm tra nội dung và fix lỗi thủ công hoặc reprocess.

## 14. References
- Official Docs: https://docs.aws.amazon.com/sqs/
- Standard vs FIFO: https://docs.aws.amazon.com/AWSSimpleQueueService/latest/SQSDeveloperGuide/standard-and-fifo-queues.html

## 15. Real-world Code
Tìm hiểu cách sử dụng `Long Polling` (`WaitTimeSeconds` > 0) để giảm chi phí bằng cách giảm số lượng request trống lên SQS.

## 16. Community
- YouTube: "AWS SQS Deep Dive" - AWS re:Invent.
- Blog: "Decoupling Microservices with SQS" - AWS Blog.
- Twitter: #AWSSQS #Serverless.
