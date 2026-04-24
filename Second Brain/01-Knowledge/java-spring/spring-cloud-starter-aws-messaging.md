---
created: 2026-04-20
tags:
  - "#type/library"
  - "#status/draft"
  - "#lang/spring"
  - "#topic/async"
related:
  - "[[spring-cloud-starter-aws]]"
  - "[[EventListener in Spring Boot]]"
---

# spring-cloud-starter-aws-messaging

## 1. What

`spring-cloud-starter-aws-messaging` (Spring Cloud AWS Messaging) là module chuyên biệt trong hệ sinh thái Spring Cloud AWS, cung cấp tích hợp giữa Spring Boot và hai dịch vụ messaging của AWS: **Amazon SQS** (Simple Queue Service) và **Amazon SNS** (Simple Notification Service). Module này expose annotation `@SqsListener` để consume SQS message theo phong cách Spring MVC `@MessageMapping`, cùng với `SqsTemplate` và `SnsTemplate` để gửi message. Kể từ Spring Cloud AWS 3.x, messaging được tách thành module riêng `spring-cloud-aws-starter-sqs` và `spring-cloud-aws-starter-sns`.

---

## 2. Why

Khi xây dựng hệ thống phân tán trên AWS, các service cần giao tiếp bất đồng bộ (asynchronous) để tách coupling và tăng resilience. Dùng AWS SDK v2 thuần để consume SQS đòi hỏi:

- Tự viết polling loop (`ReceiveMessageRequest`, `DeleteMessageRequest`).
- Tự parse JSON payload thành object.
- Tự quản lý Visibility Timeout, acknowledgement, và error handling.
- Tự implement concurrency model (bao nhiêu thread poll song song).
- Tự cấu hình Dead Letter Queue handling.

Với SNS, việc fan-out message đến nhiều endpoint cũng cần boilerplate để build `PublishRequest` và handle response. Spring Cloud AWS Messaging giải quyết bằng cách:

- Bọc toàn bộ SQS polling loop sau annotation `@SqsListener` — framework tự poll, deserialize, acknowledge, và retry.
- Expose `SqsTemplate` và `SnsTemplate` với API gọn để publish message mà không cần build request thủ công.
- Tích hợp Jackson để tự động serialize/deserialize payload sang POJO.
- Cho phép cấu hình concurrency, batch size, và visibility timeout qua `application.properties`.

---

## 3. Mental Model

Hãy tưởng tượng một bưu điện AWS với hai phòng ban:

**SQS — Phòng hàng đợi thư tín:** Người gửi bỏ thư vào hòm thư (queue). Hòm thư giữ thư cho đến khi người nhận đến lấy. Thư không bị mất nếu người nhận đang bận — nó nằm yên trong hòm, chờ đến lượt. Nếu người nhận lấy thư nhưng xử lý thất bại, thư được bỏ lại hòm để thử lại (Visibility Timeout).

**SNS — Phòng loa phóng thanh:** Người gửi nói một lần vào mic (publish). Hệ thống broadcast ngay lập tức đến tất cả người đăng ký (subscribers) — có thể là nhiều SQS queue, Lambda, email, HTTP endpoint cùng lúc.

`@SqsListener` giống như thuê một **nhân viên bưu điện chuyên nghiệp**: bạn chỉ cần nói "khi có thư ở hòm này thì đưa cho tôi đã được mở sẵn và dịch sang tiếng Việt (deserialize)". Nhân viên tự biết lấy thư, ký nhận, và báo lại nếu có vấn đề.

---

## 4. Where it fits

```
[Service A - Producer]
        |
        |-- SqsTemplate.send("order-queue", orderEvent)
        |-- SnsTemplate.publish("order-topic", orderEvent)
        |
        v
[Amazon SQS / Amazon SNS]
        |
        |-- SQS: message nằm trong queue, chờ consumer poll
        |-- SNS: fan-out ngay đến tất cả subscribers
        |
        v
[Service B - Consumer]
        |
        |-- @SqsListener("order-queue")
        |   Spring Cloud AWS tự poll + deserialize + ack
        |
        v
[Business Logic Handler]
```

SQS phù hợp cho point-to-point (một producer, một consumer group). SNS phù hợp cho pub/sub (một publisher, nhiều subscriber). Hai pattern thường kết hợp: SNS fan-out đến nhiều SQS queue (SNS -> SQS fan-out pattern).

---

## 5. When to use

- Cần giao tiếp bất đồng bộ giữa các microservice trên AWS mà không muốn quản lý message broker riêng (Kafka, RabbitMQ).
- Background job processing: upload ảnh xong thì gửi message vào queue, worker consume và resize ảnh.
- Event-driven architecture: service A publish sự kiện, nhiều service B/C/D consume độc lập.
- Decoupling: tách service gửi đơn hàng với service gửi email xác nhận — hai service không cần biết nhau.
- Hệ thống cần khả năng retry tự động khi consumer bị lỗi tạm thời, với DLQ cho message xử lý thất bại nhiều lần.

---

## 6. When NOT to use

- Cần message delivery đảm bảo thứ tự tuyệt đối (strict ordering): SQS Standard không đảm bảo thứ tự. Dùng SQS FIFO Queue, nhưng throughput bị giới hạn 300 msg/s (3000 msg/s với batching).
- Cần message streaming với retention dài ngày và replay: dùng Apache Kafka hoặc Amazon Kinesis thay vì SQS.
- Latency cực thấp (sub-millisecond): SQS có độ trễ vài ms đến vài chục ms do HTTP polling. Dùng Redis Pub/Sub hoặc gRPC streaming cho use case realtime.
- Consumer cần pull message theo tốc độ riêng và SQS message lớn (>256KB): SQS giới hạn 256KB/message — cần S3 Extended Client Pattern.
- Ứng dụng không deploy trên AWS: Spring Cloud AWS Messaging chỉ hoạt động với AWS services.

---

## 7. Trade-offs

| Ưu điểm | Nhược điểm |
|---------|-----------|
| Không cần quản lý broker infrastructure — AWS lo hoàn toàn | Vendor lock-in với AWS; migration sang RabbitMQ/Kafka tốn effort |
| `@SqsListener` ẩn hoàn toàn polling loop, retry, acknowledgement | Debugging khó hơn khi message không được xử lý — cần CloudWatch |
| Tích hợp Jackson tự động serialize/deserialize POJO | SQS Standard không đảm bảo ordering và có thể duplicate |
| Auto-scaling consumer: thêm instance là thêm consumer tự động | Message size giới hạn 256KB — cần workaround cho payload lớn |
| SQS practically free ở scale vừa (~1M request/tháng miễn phí) | SNS + SQS subscription cost tăng theo số lượng topic và subscriber |
| DLQ handling tích hợp sẵn, không cần code thêm | Cold start overhead nếu dùng trong Lambda |

---

## 8. Alternatives

| Giải pháp | Ưu điểm | Nhược điểm | Khi nào chọn |
|-----------|---------|-----------|-------------|
| Apache Kafka | High throughput, message replay, strict ordering | Phải tự quản lý cluster hoặc dùng Confluent Cloud (đắt hơn) | Event streaming, audit log, analytics pipeline |
| RabbitMQ (Spring AMQP) | Flexible routing, lower latency, không vendor lock-in | Phải tự quản lý broker hoặc dùng Amazon MQ | Multi-cloud, complex routing patterns |
| Amazon Kinesis | Ordered within shard, message replay 7 ngày | Phức tạp hơn SQS, shard management | Real-time analytics, log aggregation |
| Redis Pub/Sub | Sub-millisecond latency | At-most-once delivery, không persist message | Realtime notification, gaming leaderboard |
| AWS EventBridge | Event routing phức tạp, SaaS integration | Learning curve cao, cold start | Complex event-driven với nhiều rule/target |

---

## 9. How

**Dependency (Spring Cloud AWS 3.x):**

```xml
<dependency>
  <groupId>io.awspring.cloud</groupId>
  <artifactId>spring-cloud-aws-starter-sqs</artifactId>
</dependency>
<dependency>
  <groupId>io.awspring.cloud</groupId>
  <artifactId>spring-cloud-aws-starter-sns</artifactId>
</dependency>
```

**application.properties:**

```properties
spring.cloud.aws.region.static=ap-southeast-1
# Cấu hình SQS listener
spring.cloud.aws.sqs.listener.max-concurrent-messages=10
spring.cloud.aws.sqs.listener.max-messages-per-poll=10
spring.cloud.aws.sqs.listener.poll-timeout=20s
```

**POJO event:**

```java
public record OrderCreatedEvent(
    String orderId,
    String customerId,
    BigDecimal totalAmount,
    Instant createdAt
) {}
```

**Producer — gửi message qua SQS:**

```java
@Service
@RequiredArgsConstructor
public class OrderEventPublisher {

    private final SqsTemplate sqsTemplate;

    public void publishOrderCreated(OrderCreatedEvent event) {
        sqsTemplate.send(to -> to
            .queue("order-created-queue")
            .payload(event)
            .messageGroupId(event.customerId())  // chỉ dùng với FIFO queue
        );
    }
}
```

**Producer — fan-out qua SNS:**

```java
@Service
@RequiredArgsConstructor
public class OrderSnsPublisher {

    private final SnsTemplate snsTemplate;

    public void publishOrderCreated(OrderCreatedEvent event) {
        snsTemplate.sendNotification(
            "arn:aws:sns:ap-southeast-1:123456789:order-events",
            event,
            "OrderCreated"   // Subject
        );
    }
}
```

**Consumer — nhận message từ SQS:**

```java
@Component
public class OrderCreatedListener {

    @SqsListener("order-created-queue")
    public void handleOrderCreated(OrderCreatedEvent event) {
        // Spring Cloud AWS tự:
        // 1. Poll SQS
        // 2. Deserialize JSON -> OrderCreatedEvent
        // 3. Gọi method này
        // 4. Nếu method return bình thường -> xóa message khỏi queue (ack)
        // 5. Nếu method throw exception -> message trở lại queue sau Visibility Timeout
        log.info("Processing order: {}", event.orderId());
        orderService.process(event);
    }
}
```

**Consumer với message attributes:**

```java
@Component
public class OrderCreatedListener {

    @SqsListener("order-created-queue")
    public void handleOrderCreated(
            OrderCreatedEvent event,
            @Header("eventType") String eventType,
            @Header(SqsHeaders.SQS_RECEIPT_HANDLE_HEADER) String receiptHandle
    ) {
        log.info("Event type: {}, receipt: {}", eventType, receiptHandle);
        orderService.process(event);
    }
}
```

---

## 10. Production concerns

**Idempotency:**
SQS at-least-once delivery có thể deliver message nhiều lần (duplicate). Handler phải idempotent. Pattern phổ biến: lưu `messageId` vào Redis hoặc DB với TTL bằng message retention period (4 ngày mặc định). Kiểm tra trước khi xử lý.

```java
@SqsListener("order-created-queue")
public void handleOrderCreated(
        OrderCreatedEvent event,
        @Header(SqsHeaders.SQS_MESSAGE_ID_HEADER) String messageId
) {
    if (idempotencyStore.exists(messageId)) {
        return;  // đã xử lý rồi, bỏ qua
    }
    orderService.process(event);
    idempotencyStore.mark(messageId);
}
```

**Visibility Timeout:**
Phải lớn hơn thời gian xử lý dự kiến + buffer. Nếu handler mất 30s, set Visibility Timeout = 120s. Nếu không, SQS put message trở lại queue khi handler chưa xong, gây duplicate.

**Dead Letter Queue (DLQ):**
Luôn cấu hình DLQ cho mọi SQS queue production với `maxReceiveCount` = 3–5. Gắn CloudWatch alarm cho `ApproximateNumberOfMessagesVisible` trên DLQ.

**Concurrency:**
`max-concurrent-messages` quyết định bao nhiêu message được xử lý song song trên một instance. Mặc định 10. Tăng lên nếu handler nhanh và I/O-bound. Giữ thấp nếu handler gọi downstream service có rate limit.

**Message size:**
SQS giới hạn 256KB. Với payload lớn hơn, dùng S3 Extended Client Pattern: lưu payload vào S3, gửi S3 reference qua SQS.

**Monitoring:**
- `ApproximateNumberOfMessagesNotVisible` — số message đang được xử lý.
- `ApproximateAgeOfOldestMessage` — báo hiệu consumer lag.
- `NumberOfMessagesSent` / `NumberOfMessagesDeleted` — throughput.
- DLQ `ApproximateNumberOfMessagesVisible` — alert khi > 0.

**SNS + SQS fan-out:**
Khi dùng SNS -> SQS, bật `Raw Message Delivery` trên SNS subscription để tránh SNS wrapper JSON bao ngoài payload. Nếu không, Jackson phải deserialize hai lần.

---

## 11. Common mistakes

**Lỗi 1: Không cấu hình DLQ dẫn đến message retry vô hạn**

Handler throw exception liên tục (ví dụ do bug) khiến message retry không ngừng, chiếm toàn bộ concurrency của consumer, block các message khác.

Fix: Luôn tạo DLQ và gắn với queue chính qua Redrive Policy (`maxReceiveCount = 3`). Khi message bị reject 3 lần, nó tự động chuyển sang DLQ. Alert ngay khi DLQ có message.

**Lỗi 2: Visibility Timeout ngắn hơn thời gian xử lý thực tế**

Message xuất hiện lại trong queue trong khi handler đang xử lý, gây duplicate. Consumer xử lý cùng một order hai lần — tạo ra đơn hàng trùng.

Fix: Đặt Visibility Timeout = 3 x thời gian xử lý trung bình. Implement idempotency check bằng `messageId` để an toàn kép.

**Lỗi 3: Không bật Raw Message Delivery khi dùng SNS -> SQS**

SNS mặc định bọc payload trong một JSON envelope:
```json
{
  "Type": "Notification",
  "Message": "{\"orderId\":\"123\"}",
  ...
}
```
Jackson không thể deserialize trực tiếp sang `OrderCreatedEvent`, gây `JsonParseException`.

Fix: Bật `Raw Message Delivery` trên SNS subscription. Khi đó SQS nhận đúng payload gốc, Spring Cloud AWS deserialize thẳng sang POJO.

**Lỗi 4: Dùng SQS Standard Queue khi nghiệp vụ cần xử lý theo thứ tự**

SQS Standard delivery best-effort ordering — message có thể đến không theo thứ tự. Ví dụ: event `ORDER_CANCELLED` đến trước `ORDER_CREATED` gây lỗi logic.

Fix: Dùng SQS FIFO Queue với `MessageGroupId` = entity ID (ví dụ `orderId`). Message trong cùng group được đảm bảo ordering. Lưu ý: FIFO Queue throughput tối đa 3000 msg/s, không phù hợp cho volume rất cao.

---

## 12. Sample project

**Bài tập: Order Processing Pipeline với SQS + DLQ**

Xây dựng hai Spring Boot service:

1. **order-service** (Producer): `POST /orders` tạo đơn hàng, lưu DB, gửi `OrderCreatedEvent` vào SQS queue `order-created-queue`.
2. **notification-service** (Consumer): `@SqsListener("order-created-queue")` nhận event, giả lập gửi email (log ra console), implement idempotency check bằng in-memory set.

Hard constraints:
- Phải cấu hình DLQ `order-created-dlq` với `maxReceiveCount = 3`.
- Consumer phải idempotent (xử lý cùng message 2 lần không gây side effect).
- Chạy local bằng LocalStack với `spring.cloud.aws.endpoint=http://localhost:4566`.
- Viết integration test dùng `@LocalStackContainer` (Testcontainers) verify message được gửi và nhận đúng.

---

## 13. Interview

**Core Q&A:**

Q: SQS at-least-once delivery là gì và tại sao quan trọng?
A: AWS đảm bảo mỗi message được deliver ít nhất một lần, nhưng trong một số trường hợp (network retry, node failure) cùng một message có thể được deliver nhiều lần. Handler phải viết idempotent — xử lý cùng message nhiều lần cho kết quả như nhau. Kỹ thuật thường dùng: lưu messageId vào DB/Redis trước khi xử lý, kiểm tra trùng lặp trước khi thực hiện business logic.

Q: Visibility Timeout trong SQS là gì?
A: Khi consumer nhận một message, SQS ẩn message đó khỏi queue trong một khoảng thời gian (Visibility Timeout). Nếu consumer xử lý xong và delete message trước khi timeout, message biến mất. Nếu consumer crash hoặc timeout hết hạn, message xuất hiện lại trong queue để consumer khác nhận. Đây là cơ chế đảm bảo at-least-once delivery.

Q: SQS Standard khác SQS FIFO như thế nào?
A: Standard Queue: throughput gần như unlimited, best-effort ordering (không đảm bảo), at-least-once delivery, rẻ hơn. FIFO Queue: đảm bảo ordering trong cùng MessageGroupId, exactly-once processing, throughput giới hạn 3000 msg/s, đắt hơn. Chọn FIFO khi thứ tự xử lý ảnh hưởng đến business logic.

Q: SNS và SQS khác nhau như thế nào? Khi nào dùng SNS + SQS fan-out pattern?
A: SQS là point-to-point queue — một message chỉ được một consumer nhận. SNS là pub/sub — một message được broadcast đến tất cả subscribers cùng lúc. Fan-out pattern kết hợp cả hai: SNS topic fan-out đến nhiều SQS queue, mỗi queue có một consumer group riêng xử lý độc lập. Ví dụ: event `ORDER_CREATED` fan-out đến queue của email-service, inventory-service, và analytics-service.

Q: `@SqsListener` handle acknowledgement thế nào trong Spring Cloud AWS?
A: Mặc định dùng `ON_SUCCESS` acknowledgement mode — Spring Cloud AWS tự delete message khỏi SQS sau khi method return bình thường (không throw exception). Nếu method throw exception, message không bị delete, trở lại queue sau Visibility Timeout. Cũng có thể dùng `MANUAL` mode để control khi nào ack.

**Scenarios:**

Q: Consumer của bạn xử lý rất chậm và DLQ bắt đầu nhận message. Các bước debug?
A: (1) Kiểm tra CloudWatch metric `ApproximateAgeOfOldestMessage` — nếu tăng, consumer đang lag. (2) Xem log consumer tìm exception hoặc slow path. (3) So sánh thời gian xử lý với Visibility Timeout — nếu xử lý dài hơn timeout, message bị redeliver. (4) Kiểm tra downstream dependency (DB, external API) có bottleneck không. (5) Xem DLQ message content để tìm pattern lỗi.

Q: Bạn cần đảm bảo một order chỉ được xử lý đúng một lần dù SQS deliver nhiều lần. Thiết kế thế nào?
A: Dùng idempotency key là `messageId` (header `SqsHeaders.SQS_MESSAGE_ID_HEADER`). Trước khi xử lý, kiểm tra DB/Redis xem `messageId` đã tồn tại chưa. Nếu rồi thì skip. Nếu chưa, xử lý và lưu `messageId` vào store với TTL bằng message retention period của queue. Dùng distributed lock hoặc DB unique constraint để tránh race condition.

Q: Service cần notify đồng thời 5 service khác khi một order được tạo. Thiết kế messaging thế nào?
A: Dùng SNS + SQS fan-out: tạo một SNS topic `order-events`. Mỗi trong 5 service subscribe bằng một SQS queue riêng của mình vào topic đó. Khi order-service publish event lên SNS topic, SNS tự fan-out đến 5 SQS queue. Mỗi service consume queue riêng độc lập, không ảnh hưởng lẫn nhau. Bật Raw Message Delivery để tránh SNS envelope.

---

## 14. References

- Tài liệu chính thức Spring Cloud AWS SQS: https://docs.awspring.io/spring-cloud-aws/docs/3.2.1/reference/html/index.html#sqs-support
- Tài liệu chính thức Spring Cloud AWS SNS: https://docs.awspring.io/spring-cloud-aws/docs/3.2.1/reference/html/index.html#sns-support
- GitHub repository: https://github.com/awspring/spring-cloud-aws
- AWS SQS Developer Guide: https://docs.aws.amazon.com/AWSSimpleQueueService/latest/SQSDeveloperGuide/welcome.html
- AWS SNS Developer Guide: https://docs.aws.amazon.com/sns/latest/dg/welcome.html
- SQS FIFO Queue best practices: https://docs.aws.amazon.com/AWSSimpleQueueService/latest/SQSDeveloperGuide/FIFO-queues-understanding-logic.html

---

## 15. Real-world Code

- Spring Cloud AWS integration tests cho SQS (cách test thực tế với LocalStack): https://github.com/awspring/spring-cloud-aws/tree/main/spring-cloud-aws-integration-tests/src/test/java/io/awspring/cloud/it/sqs
- `maciejwalkowiak/spring-sqs-listener-example` — ví dụ SqsListener với error handling và DLQ: tìm trên GitHub của maciejwalkowiak
- AWS Labs sample: SNS + SQS fan-out pattern với Spring Boot: tìm trong `aws-samples` GitHub organization với keyword `spring-sqs-sns`

---

## 16. Community

- Reddit: r/SpringBoot, r/aws — tìm "SQS Spring" hoặc "spring cloud aws messaging"
- Stack Overflow tags: `spring-cloud-aws` + `amazon-sqs`: https://stackoverflow.com/questions/tagged/spring-cloud-aws+amazon-sqs
- Blog: Maciej Walkowiak — https://maciejwalkowiak.com (maintainer Spring Cloud AWS, nhiều bài về SQS patterns)
- Blog: Baeldung — tìm "Spring Cloud AWS SQS" tại https://www.baeldung.com
- Talk: "Messaging with SQS and SNS in Spring Boot" — tìm trên YouTube AWS channel và Spring I/O playlist
- Discord: Spring Community — channel `#spring-cloud`
