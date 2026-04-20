---
created: 2026-04-20
tags:
  - "#type/library"
  - "#status/draft"
  - "#lang/spring"
  - "#topic/cloud"
related:
  - "[[Feign Client in Spring Boot]]"
  - "[[WebClient in Spring Boot]]"
---

# spring-cloud-starter-aws

## 1. What

`spring-cloud-starter-aws` (Spring Cloud AWS) là một module thuộc hệ sinh thái Spring Cloud, cung cấp tích hợp tự nhiên giữa ứng dụng Spring Boot và các dịch vụ AWS (Amazon Web Services). Thay vì làm việc trực tiếp với AWS SDK thuần, Spring Cloud AWS bọc các dịch vụ như S3, SQS, SNS, RDS, ElastiCache, Secrets Manager thành các abstraction quen thuộc của Spring (auto-configuration, Spring beans, `@Autowired`). Kể từ phiên bản 3.x, dự án được tách thành các module độc lập (`spring-cloud-aws-s3`, `spring-cloud-aws-sqs`, v.v.) thay vì một starter monolithic.

---

## 2. Why

AWS SDK for Java v2 rất mạnh nhưng verbose: bạn phải tự khởi tạo client, quản lý credentials, configure region, serialize/deserialize message, và handle retry logic. Trong một ứng dụng Spring Boot, điều này tạo ra boilerplate lặp lại ở khắp nơi:

- Mỗi service cần S3 phải tự tạo `S3Client` bean và inject region/credentials.
- Gửi message SQS phải tự build `SendMessageRequest`, handle batch, và parse `ReceiveMessageResponse`.
- Đọc secret từ AWS Secrets Manager phải tự gọi SDK rồi inject thủ công vào `@Value`.
- Config properties AWS không integrate với Spring `Environment` abstraction.

Spring Cloud AWS giải quyết bằng cách:
- Auto-configure AWS clients dựa trên `application.properties` — không cần `@Bean` thủ công.
- Expose `S3Template`, `SqsTemplate`, `SnsTemplate` theo kiểu Spring — API gọn và consistent.
- Integrate AWS Secrets Manager và Parameter Store vào Spring `Environment` — inject trực tiếp bằng `@Value`.
- Dùng IAM role tự động trên EC2/ECS/EKS — không cần hardcode credentials.

---

## 3. Mental Model

Hãy tưởng tượng AWS giống như một tòa nhà văn phòng khổng lồ với hàng chục phòng ban khác nhau (S3, SQS, SNS, RDS, v.v.). Mỗi phòng ban có quy trình làm việc riêng, form mẫu riêng, và nhân viên lễ tân riêng.

AWS SDK thuần: Bạn phải tự đến từng phòng ban, hỏi đường (configure endpoint), xuất trình thẻ nhân viên (credentials), điền form theo quy trình riêng của từng phòng, rồi tự mang kết quả về.

Spring Cloud AWS: Bạn có một **trợ lý tòa nhà (Spring Cloud AWS)**. Bạn nói "gửi file này lên S3" hay "đặt message này vào hàng đợi SQS" — trợ lý biết đường đến phòng nào, quen mặt với lễ tân (credentials đã configured), điền form hộ bạn, và trả về kết quả theo ngôn ngữ Spring mà bạn đã quen.

---

## 4. Where it fits

```
[Spring Boot Application]
        |
        |-- @Value("${secret}")  <-- AWS Secrets Manager / Parameter Store
        |-- S3Template           <-- Amazon S3
        |-- SqsTemplate          <-- Amazon SQS
        |-- SnsTemplate          <-- Amazon SNS
        |-- DataSource (auto)    <-- Amazon RDS (via Spring JDBC)
        |
        v
[Spring Cloud AWS Auto-Configuration]
        |
        v
[AWS SDK v2 Clients (S3Client, SqsClient, SnsClient, ...)]
        |
        v
[AWS Services (us-east-1, ap-southeast-1, ...)]
```

Spring Cloud AWS nằm ở tầng giữa, dịch Spring idioms sang AWS SDK calls. Ứng dụng không biết và không cần biết chi tiết của AWS SDK.

---

## 5. When to use

- Ứng dụng Spring Boot triển khai trên AWS (EC2, ECS, EKS, Lambda).
- Cần giao tiếp với S3, SQS, SNS, Secrets Manager, Parameter Store trong Spring Boot.
- Muốn inject AWS secrets trực tiếp vào `@Value` mà không cần code boilerplate.
- Dùng SQS như một message queue thay thế cho RabbitMQ/Kafka trong hệ thống nhỏ/vừa trên AWS.
- Team quen với Spring và muốn giảm thiểu tiếp xúc trực tiếp với AWS SDK thuần.

---

## 6. When NOT to use

- Ứng dụng chạy trên môi trường không phải AWS (GCP, Azure, on-premise) — Spring Cloud AWS chỉ hỗ trợ AWS, không có abstraction multi-cloud.
- Cần control chi tiết AWS SDK (custom retry policy, presigned URL phức tạp, multipart upload lớn) — dùng AWS SDK v2 trực tiếp, tránh bị giới hạn bởi abstraction của Spring Cloud AWS.
- Lambda function cần cold start nhanh — Spring Cloud AWS auto-configuration thêm overhead khởi động đáng kể; cân nhắc dùng Quarkus/Micronaut hoặc AWS SDK thuần.
- Team chưa có kinh nghiệm AWS — Spring Cloud AWS ẩn đi nhiều detail, gây khó debug khi có vấn đề permission hoặc network.

---

## 7. Trade-offs

| Ưu điểm | Nhược điểm |
|---------|-----------|
| Giảm boilerplate đáng kể so với AWS SDK thuần | Thêm dependency, tăng kích thước artifact |
| Auto-configuration tự phát hiện credentials (IAM role, env vars, profiles) | Abstraction che khuất AWS SDK detail, khó debug IAM/network error |
| Inject secrets vào `@Value` như config thông thường | Phiên bản 3.x breaking changes lớn so với 2.x — migration tốn effort |
| Template API (`S3Template`, `SqsTemplate`) nhất quán và dễ test (mock) | Một số tính năng AWS SDK v2 chưa được expose qua template |
| Tích hợp tốt với Spring Boot test (`@SpringBootTest`, `@LocalStackContainer`) | LocalStack setup thêm phức tạp cho integration test |
| Hỗ trợ Virtual Threading (Spring Boot 3.2+) | Tài liệu chính thức còn thiếu ví dụ thực tế so với AWS SDK docs |

---

## 8. Alternatives

| Giải pháp | Ưu điểm | Nhược điểm | Khi nào chọn |
|-----------|---------|-----------|-------------|
| AWS SDK v2 thuần | Full control, không overhead, AWS docs chính thức đầy đủ | Verbose, phải tự configure bean, không Spring-idiomatic | Khi cần tính năng AWS không có trong Spring Cloud AWS |
| Quarkus AWS extensions | Cold start thấp hơn, native image tốt hơn | Không phải Spring, team cần học Quarkus | Lambda function cần performance cực cao |
| Micronaut AWS | Tương tự Quarkus, AOT compilation | Không phải Spring | Lambda / serverless ưu tiên startup time |
| Localstack + manual SDK | Control hoàn toàn cho testing | Setup phức tạp hơn | CI/CD environment cần full AWS emulation |

---

## 9. How

**Thêm dependency (Maven, Spring Cloud AWS 3.x):**

```xml
<dependencyManagement>
  <dependencies>
    <dependency>
      <groupId>io.awspring.cloud</groupId>
      <artifactId>spring-cloud-aws-dependencies</artifactId>
      <version>3.2.1</version>
      <type>pom</type>
      <scope>import</scope>
    </dependency>
  </dependencies>
</dependencyManagement>

<dependencies>
  <!-- S3 -->
  <dependency>
    <groupId>io.awspring.cloud</groupId>
    <artifactId>spring-cloud-aws-starter-s3</artifactId>
  </dependency>
  <!-- SQS -->
  <dependency>
    <groupId>io.awspring.cloud</groupId>
    <artifactId>spring-cloud-aws-starter-sqs</artifactId>
  </dependency>
  <!-- Secrets Manager -->
  <dependency>
    <groupId>io.awspring.cloud</groupId>
    <artifactId>spring-cloud-aws-starter-secrets-manager</artifactId>
  </dependency>
</dependencies>
```

**application.properties:**

```properties
spring.cloud.aws.region.static=ap-southeast-1
spring.cloud.aws.credentials.access-key=${AWS_ACCESS_KEY_ID}
spring.cloud.aws.credentials.secret-key=${AWS_SECRET_ACCESS_KEY}

# Inject từ Secrets Manager (key /myapp/db-password trong AWS)
spring.cloud.aws.secretsmanager.import[0]=/myapp/db-password
```

**Upload file lên S3:**

```java
@Service
@RequiredArgsConstructor
public class FileStorageService {

    private final S3Template s3Template;

    public String upload(String bucket, String key, MultipartFile file) throws IOException {
        s3Template.upload(bucket, key, file.getInputStream(),
            ObjectMetadata.builder().contentType(file.getContentType()).build());
        return s3Template.createSignedGetURL(bucket, key, Duration.ofMinutes(60)).toString();
    }

    public Resource download(String bucket, String key) {
        return s3Template.download(bucket, key);
    }
}
```

**Gửi và nhận SQS message:**

```java
@Service
@RequiredArgsConstructor
public class OrderEventService {

    private final SqsTemplate sqsTemplate;

    public void sendOrderCreated(OrderEvent event) {
        sqsTemplate.send("order-events-queue", event);  // tự serialize sang JSON
    }
}

@Component
public class OrderEventListener {

    @SqsListener("order-events-queue")
    public void handleOrderCreated(OrderEvent event) {
        // Spring tự deserialize JSON -> OrderEvent
        System.out.println("Received order: " + event.orderId());
    }
}
```

**Inject secret từ Secrets Manager:**

```java
@Value("${/myapp/db-password}")
private String dbPassword;
```

---

## 10. Production concerns

**Credentials:**
- Trên EC2/ECS/EKS, không bao giờ hardcode `access-key` và `secret-key`. Dùng IAM Instance Profile / Task Role — Spring Cloud AWS tự phát hiện.
- Dùng `spring.cloud.aws.credentials.instance-profile=true` để bật rõ ràng trên EC2.

**S3:**
- Multipart upload cho file lớn (>100MB): dùng AWS SDK `S3AsyncClient` trực tiếp, `S3Template` chưa hỗ trợ native multipart.
- Presigned URL hết hạn ngắn (15–60 phút) cho download — không expose bucket public.
- Bật S3 Versioning nếu cần rollback file.

**SQS:**
- Visibility timeout phải lớn hơn thời gian xử lý message để tránh duplicate processing.
- Bật Dead Letter Queue (DLQ) cho mọi SQS queue production — set `maxReceiveCount` 3–5.
- `@SqsListener` mặc định dùng Long Polling (20 giây) — tiết kiệm chi phí.
- Idempotency: SQS `at-least-once` delivery, message có thể duplicate — handler phải idempotent.

**Secrets Manager:**
- Bật caching để tránh gọi Secrets Manager mỗi request (Spring Cloud AWS 3.x cache mặc định).
- IAM policy chỉ cấp `secretsmanager:GetSecretValue` cho secret cụ thể, không wildcard `*`.

**Monitoring:**
- Bật CloudWatch metrics cho SQS (`ApproximateNumberOfMessagesNotVisible`, `ApproximateAgeOfOldestMessage`).
- Alert khi DLQ có message — thường báo hiệu bug trong handler.

---

## 11. Common mistakes

**Lỗi 1: Hardcode credentials trong application.properties**

```properties
# SAI - credentials lộ vào source control
spring.cloud.aws.credentials.access-key=AKIAIOSFODNN7EXAMPLE
spring.cloud.aws.credentials.secret-key=wJalrXUtnFEMI/K7MDENG/bPxRfiCYEXAMPLEKEY
```

Fix: Dùng IAM role trên AWS hoặc inject qua environment variable, không bao giờ commit credentials vào repo.

```properties
# ĐÚNG - inject từ environment variable
spring.cloud.aws.credentials.access-key=${AWS_ACCESS_KEY_ID:}
spring.cloud.aws.credentials.secret-key=${AWS_SECRET_ACCESS_KEY:}
```

**Lỗi 2: Không set Visibility Timeout đủ lớn cho SQS listener**

Nếu xử lý một message mất 30 giây nhưng Visibility Timeout chỉ là 30 giây (mặc định), SQS sẽ put message trở lại queue trước khi handler hoàn thành, gây duplicate processing.

Fix: Set Visibility Timeout trên SQS queue lớn hơn thời gian xử lý dự kiến cộng thêm buffer an toàn, ví dụ 120 giây cho handler chạy tối đa 60 giây.

**Lỗi 3: Quên cấu hình DLQ cho SQS queue production**

Message lỗi sẽ retry vô hạn, block queue, và gây CPU spike.

Fix: Luôn gắn DLQ với `maxReceiveCount` 3–5. Monitor DLQ bằng CloudWatch alarm.

**Lỗi 4: Dùng Spring Cloud AWS 2.x dependency khi project dùng Spring Boot 3.x**

`io.awspring.cloud:spring-cloud-aws-autoconfigure:2.4.x` không tương thích với Spring Boot 3.x (do Jakarta EE namespace migration). Gây `ClassNotFoundException` hoặc auto-configuration không load.

Fix: Dùng `io.awspring.cloud:spring-cloud-aws-dependencies:3.x` BOM cho Spring Boot 3.x.

---

## 12. Sample project

**Bài tập: File Upload Service với S3 + SQS notification**

Xây dựng một REST API nhỏ với hai chức năng:
1. `POST /files` — nhận `MultipartFile`, upload lên S3, gửi một SQS message chứa `{fileKey, uploadedAt, userId}`.
2. `GET /files/{key}/url` — trả về presigned URL có hạn 30 phút.
3. Một `@SqsListener` log message nhận được ra console.

Hard constraint: Tất cả credentials phải đọc từ environment variable, không hardcode trong code hay config file. Chạy được local với LocalStack (dùng `spring.cloud.aws.endpoint`).

---

## 13. Interview

**Core Q&A:**

Q: Spring Cloud AWS khác gì so với dùng AWS SDK v2 trực tiếp?
A: Spring Cloud AWS bọc AWS SDK v2 thành Spring abstractions — auto-configuration, `S3Template`/`SqsTemplate`, inject credentials tự động, và tích hợp secrets vào Spring `Environment`. AWS SDK v2 thuần cho phép control chi tiết hơn nhưng verbose hơn và không Spring-idiomatic.

Q: Spring Cloud AWS phát hiện credentials theo thứ tự nào?
A: Theo AWS Default Credentials Provider Chain: environment variables (`AWS_ACCESS_KEY_ID`) -> Java system properties -> AWS profile (`~/.aws/credentials`) -> ECS task role -> EC2 instance profile. Trên production AWS, nên dùng IAM role (ECS task role hoặc EC2 instance profile), không bao giờ hardcode.

Q: SQS `at-least-once` nghĩa là gì và ảnh hưởng đến code như thế nào?
A: SQS đảm bảo mỗi message được deliver ít nhất một lần, nhưng có thể deliver nhiều hơn một lần (duplicate). Handler phải viết theo kiểu idempotent — xử lý cùng một message nhiều lần phải cho kết quả giống nhau. Thường dùng unique message ID hoặc database upsert để đảm bảo idempotency.

Q: Tại sao không nên dùng Spring Cloud AWS cho Lambda cold start nhạy cảm?
A: Spring Boot context initialization + Spring Cloud AWS auto-configuration add thêm 500ms–2s vào cold start. Với Lambda cần respond trong 100ms, overhead này không chấp nhận được. Giải pháp thay thế: AWS SDK v2 thuần, Quarkus, hoặc Micronaut với native image.

Q: Secrets Manager integration trong Spring Cloud AWS 3.x hoạt động thế nào?
A: Khi khai báo `spring.cloud.aws.secretsmanager.import[0]=/myapp/db-password`, Spring Cloud AWS fetch secret từ AWS Secrets Manager trong quá trình bootstrap context và thêm vào `Environment`. Từ đó `@Value("${/myapp/db-password}")` hoạt động bình thường như config thông thường.

**Scenarios:**

Q: Team muốn test integration với S3 và SQS mà không kết nối AWS thật. Bạn sẽ làm thế nào?
A: Dùng LocalStack — một Docker image emulate các AWS service locally. Trong Spring Boot test, dùng `@LocalStackContainer` (Testcontainers) hoặc Docker Compose. Override endpoint bằng `spring.cloud.aws.endpoint=http://localhost:4566`. Spring Cloud AWS sẽ route tất cả calls tới LocalStack thay vì AWS.

Q: SQS listener của bạn đột ngột xử lý rất chậm, DLQ bắt đầu có message. Debug như thế nào?
A: Kiểm tra theo thứ tự: (1) CloudWatch metric `ApproximateAgeOfOldestMessage` để xem message tồn đọng bao lâu. (2) Application logs xem handler đang throw exception hay không. (3) Kiểm tra Visibility Timeout có đủ lớn không. (4) Xem DLQ message content để tìm pattern lỗi. (5) Kiểm tra downstream dependency (DB, external API) có chậm hoặc down không.

Q: Bạn cần cho phép user download file từ S3 private bucket nhưng không muốn expose bucket public. Giải pháp?
A: Dùng presigned URL — `s3Template.createSignedGetURL(bucket, key, Duration.ofMinutes(30))`. URL này chứa signature AWS, cho phép download trong thời gian giới hạn mà không cần credentials. Bucket vẫn private hoàn toàn.

---

## 14. References

- Tài liệu chính thức Spring Cloud AWS 3.x: https://docs.awspring.io/spring-cloud-aws/docs/3.2.1/reference/html/
- GitHub repository: https://github.com/awspring/spring-cloud-aws
- AWS SDK for Java v2 (tầng bên dưới): https://docs.aws.amazon.com/sdk-for-java/latest/developer-guide/home.html
- Changelog Spring Cloud AWS: https://github.com/awspring/spring-cloud-aws/releases
- Migration guide 2.x -> 3.x: https://docs.awspring.io/spring-cloud-aws/docs/3.2.1/reference/html/index.html#migration-guide

---

## 15. Real-world Code

- `awspring/spring-cloud-aws` (source chính thức, đọc integration tests để hiểu cách dùng): https://github.com/awspring/spring-cloud-aws/tree/main/spring-cloud-aws-integration-tests
- `maciejwalkowiak/spring-boot-s3-example` — ví dụ minimal S3 upload/download với Spring Boot 3: https://github.com/maciejwalkowiak/spring-boot-s3-example
- Ứng dụng microservice mẫu dùng SQS listener + DLQ pattern: tìm trong Spring Cloud AWS samples folder trên GitHub.

---

## 16. Community

- Reddit: r/SpringBoot — tìm "spring cloud aws" để xem discussion thực tế
- Stack Overflow tag: `spring-cloud-aws` — https://stackoverflow.com/questions/tagged/spring-cloud-aws
- Blog: Maciej Walkowiak (maintainer chính của Spring Cloud AWS) — https://maciejwalkowiak.com
- Talk: "Spring Cloud AWS 3.0" tại Spring I/O — tìm trên YouTube với keyword "Spring Cloud AWS 3.0 Spring IO"
- Discord: Spring Community Discord — channel `#spring-cloud`
