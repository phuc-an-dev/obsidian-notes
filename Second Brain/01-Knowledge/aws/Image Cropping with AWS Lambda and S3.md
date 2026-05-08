---
created: 2026-05-07
tags:
  - "#type/tutorial"
  - "#status/draft"
  - "#lang/devops"
  - "#topic/async"
related:
  - "[[lambda]]"
  - "[[s3]]"
---

## 1. What
Đây là một kiến trúc serverless sử dụng AWS Lambda để tự động xử lý (crop/resize) hình ảnh ngay khi chúng được upload lên một S3 Bucket (Source), sau đó lưu kết quả vào một S3 Bucket khác (Destination). Quá trình này được kích hoạt bởi S3 Event Notifications.

## 2. Why
Việc xử lý ảnh trực tiếp trên server truyền thống (EC2) gây tốn tài nguyên CPU và khó mở rộng khi có hàng ngàn user upload cùng lúc. Sử dụng Lambda giúp:
- Tự động hóa hoàn toàn (Event-driven).
- Khả năng mở rộng vô hạn (Scalability).
- Chỉ trả tiền cho thời gian thực thi thực tế (Cost-effective).

## 3. Mental Model
Hãy tưởng tượng S3 Bucket giống như một **hòm thư**.
- Khi có một bức thư mới (ảnh gốc) được bỏ vào hòm thư "Đầu vào".
- Một **người giúp việc** (Lambda) sẽ ngay lập tức nhận được thông báo, chạy đến lấy bức thư đó ra.
- Người giúp việc dùng kéo cắt tỉa bức thư (Crop ảnh) theo đúng yêu cầu.
- Cuối cùng, người giúp việc bỏ bức thư đã cắt vào hòm thư "Kết quả".

## 4. Where it fits
User -> S3 Upload (Source) -> **S3 Event Notification** -> **AWS Lambda (Image Processor)** -> S3 Put (Destination).

## 5. When to use
- Tạo ảnh thumbnail cho các website mạng xã hội hoặc e-commerce.
- Chuẩn hóa kích thước ảnh profile của người dùng.
- Xử lý ảnh hàng loạt (batch processing) mà không muốn quản lý server.

## 6. When NOT to use
- Khi ảnh có dung lượng cực lớn (> 500MB) vượt quá giới hạn bộ nhớ hoặc thời gian chạy của Lambda.
- Khi cần xử lý video nặng (nên dùng AWS Elemental MediaConvert).
- Khi logic xử lý yêu cầu các thư viện đặc thù không hỗ trợ tốt trên môi trường Linux của Lambda.

## 7. Trade-offs
| Pros | Cons |
|------|------|
| Zero server management. | Cold start có thể làm tăng latency cho bức ảnh đầu tiên. |
| Tự động scale theo lượng upload. | Phải quản lý thư viện xử lý ảnh (như Sharp) dưới dạng Lambda Layers. |
| Tích hợp sẵn với hệ sinh thái AWS. | Giới hạn thời gian chạy tối đa 15 phút. |

## 8. Alternatives
| Option | So sánh |
|--------|---------|
| AWS CloudFront Functions / Edge@Lambda | Xử lý ảnh tại vùng biên (Edge), tốt cho resize on-the-fly nhưng phức tạp hơn. |
| EC2 + Queue (SQS) | Kiểm soát tốt hơn nhưng tốn công quản lý hạ tầng. |

## 9. How
Ví dụ sử dụng Node.js với thư viện `sharp` (cần được package trong Lambda Layer).

```javascript
const { S3Client, GetObjectCommand, PutObjectCommand } = require("@aws-sdk/client-s3");
const sharp = require("sharp");

const s3 = new S3Client();

exports.handler = async (event) => {
    const bucket = event.Records[0].s3.bucket.name;
    const key = decodeURIComponent(event.Records[0].s3.object.key.replace(/\+/g, " "));
    const dstBucket = bucket + "-resized";
    const dstKey = "cropped-" + key;

    try {
        // 1. Get image from Source S3
        const response = await s3.send(new GetObjectCommand({ Bucket: bucket, Key: key }));
        const stream = response.Body;
        const buffer = await streamToBuffer(stream);

        // 2. Process image with Sharp
        const outputBuffer = await sharp(buffer)
            .resize(300, 300) // Resize hoặc crop tùy ý
            .toBuffer();

        // 3. Upload to Destination S3
        await s3.send(new PutObjectCommand({
            Bucket: dstBucket,
            Key: dstKey,
            Body: outputBuffer,
            ContentType: "image/jpeg"
        }));

        return { status: "Success" };
    } catch (error) {
        console.error(error);
        throw error;
    }
};

async function streamToBuffer(stream) {
    return new Promise((resolve, reject) => {
        const chunks = [];
        stream.on("data", (chunk) => chunks.push(chunk));
        stream.on("error", reject);
        stream.on("end", () => resolve(Buffer.concat(chunks)));
    });
}
```

## 10. Production concerns
### Scaling
Lambda có thể chạy hàng ngàn instance song song. Cần chú ý đến quota của tài khoản (Concurrent executions).

### Failure
Sử dụng **Dead Letter Queue (DLQ)** để lưu lại các sự kiện (events) bị lỗi để xử lý lại sau.

### Monitoring
Theo dõi `Duration`, `MemoryUsage`, và `Errors` trong CloudWatch Metrics. Sử dụng `console.log` để debug (logs sẽ vào CloudWatch Logs).

## 11. Common mistakes
- Mistake: Upload ảnh đã xử lý vào **cùng một bucket** với ảnh gốc mà không lọc prefix, dẫn đến vòng lặp vô tận (Infinite recursion).
  Fix: Luôn dùng 2 bucket khác nhau hoặc dùng tiền tố (Prefix/Folder) khác nhau và cấu hình trigger cẩn thận.

- Mistake: Quên cấp quyền `s3:GetObject` và `s3:PutObject` cho IAM Role của Lambda.
  Fix: Kiểm tra IAM Policy gắn với Lambda.

## 12. Sample project
Tạo một quy trình tự động: Người dùng upload ảnh thẻ, Lambda nhận diện khuôn mặt (dùng Rekognition) sau đó tự động crop lấy phần mặt và lưu vào bucket "profile-pictures".

## 13. Interview
### Core Q&A
1. Q: Làm thế nào để tránh vòng lặp vô hạn khi Lambda ghi file vào S3?
   A: Sử dụng hai bucket riêng biệt (Input vs Output) hoặc cấu hình S3 Trigger chỉ kích hoạt với một Prefix nhất định (ví dụ: `uploads/`) và ghi kết quả vào Prefix khác (ví dụ: `processed/`).

2. Q: Tại sao nên dùng Lambda Layer cho thư viện `sharp`?
   A: Vì `sharp` có các file binary phụ thuộc vào hệ điều hành (native dependencies). Dùng Layer giúp code Lambda gọn nhẹ hơn và dễ tái sử dụng thư viện cho nhiều function khác.

### Scenario
Hệ thống của bạn thỉnh thoảng bị lỗi khi xử lý các ảnh PNG dung lượng lớn. Bạn sẽ kiểm tra gì?
Trả lời: Kiểm tra cấu hình Memory của Lambda (mặc định 128MB có thể không đủ để load ảnh lớn vào buffer). Tăng Memory cũng đồng thời tăng sức mạnh CPU, giúp xử lý nhanh hơn.

## 14. References
- AWS Tutorial: https://docs.aws.amazon.com/lambda/latest/dg/with-s3-example.html
- Sharp Documentation: https://sharp.pixelplumbing.com/

## 15. Real-world Code
Mẫu Lambda xử lý ảnh chuyên nghiệp:
https://github.com/aws-samples/lambda-refarch-imagerecognition

## 16. Community
- YouTube: "S3 Event Notifications with AWS Lambda" - AWS Developers.
- Blog: "Serverless Image Resizing with AWS Lambda & S3" - Serverless Stack.
