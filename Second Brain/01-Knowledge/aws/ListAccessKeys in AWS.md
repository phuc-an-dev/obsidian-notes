---
created: 2026-05-06
tags:
  - "#type/library"
  - "#status/draft"
  - "#lang/devops"
  - "#topic/iam"
related:
  - "[[iam.md]]"
  - "[[aws-mfa.md]]"
  - "[[aws-backdoor-checklist.md]]"
---

## 1. What
`ListAccessKeys` là một API của dịch vụ AWS Identity and Access Management (IAM), cho phép liệt kê thông tin về các Access Key ID được liên kết với một IAM User cụ thể. API này trả về các siêu dữ liệu (metadata) của key như Access Key ID, trạng thái (Active/Inactive), và ngày tạo, nhưng không bao giờ trả về Secret Access Key.

## 2. Why
Trước khi có công cụ quản lý tập trung, việc theo dõi xem một user đang sở hữu bao nhiêu key và chúng đã tồn tại bao lâu là rất khó khăn. `ListAccessKeys` ra đời để phục vụ mục đích kiểm soát (auditing), giúp quản trị viên hoặc các script tự động phát hiện các key cũ, key không còn sử dụng hoặc đảm bảo tuân thủ chính sách "chỉ có tối đa 2 access keys cho mỗi user".

## 3. Mental Model
Hãy tưởng tượng IAM User là một **"Chủ thẻ ngân hàng"** và `ListAccessKeys` là lệnh **"Liệt kê danh sách thẻ"** trên ứng dụng mobile banking:
- Bạn có thể thấy số thẻ (Access Key ID) và ngày mở thẻ.
- Bạn biết thẻ nào đang hoạt động, thẻ nào bị khóa.
- Tuy nhiên, ứng dụng không bao giờ hiển thị mã PIN (Secret Access Key) của các thẻ cũ, bạn chỉ thấy mã PIN một lần duy nhất khi vừa làm thẻ xong.

## 4. Where it fits
Vị trí trong luồng quản lý bảo mật:
`Administrator/Script -> ListAccessKeys -> IAM Engine -> Metadata Table -> Response List`

## 5. When to use
- Khi cần thực hiện việc xoay vòng key (Key Rotation) định kỳ.
- Khi cần kiểm tra xem một user có đang vi phạm giới hạn số lượng access keys không.
- Khi xây dựng các công cụ báo cáo bảo mật nội bộ để quét toàn bộ tài khoản AWS.

## 6. When NOT to use
- Khi bạn cần lấy Secret Access Key (điều này là không thể sau khi key đã được tạo).
- Khi bạn muốn kiểm tra quyền hạn của user (trường hợp này dùng `ListAttachedUserPolicies` hoặc `SimulatePrincipalPolicy`).

## 7. Trade-offs
| Pros | Cons |
|------|------|
| Cung cấp cái nhìn tổng quan về trạng thái credential của user. | Không cho biết lần cuối key được sử dụng (phải dùng `GetAccessKeyLastUsed`). |
| Nhẹ, tốc độ phản hồi nhanh. | Kết quả bị phân trang (pagination) nếu user có quá nhiều key (dù thực tế giới hạn là 2). |
| Hỗ trợ tốt cho tự động hóa bảo mật. | |

## 8. Alternatives
| Option | So sánh |
|--------|---------|
| Credential Report | Xuất file CSV cho toàn bộ user trong account, chậm hơn nhưng bao quát hơn. |
| AWS Config | Theo dõi thay đổi của IAM keys theo thời gian thực nhưng cấu hình phức tạp hơn. |

## 9. How
Ví dụ sử dụng AWS CLI và SDK:

```bash
# Liệt kê key cho một user cụ thể
aws iam list-access-keys --user-name phuc-an-dev
```

Ví dụ dùng AWS SDK for Java:
```java
IamClient iam = IamClient.builder().region(Region.AWS_GLOBAL).build();
ListAccessKeysRequest request = ListAccessKeysRequest.builder()
    .userName("phuc-an-dev")
    .build();
ListAccessKeysResponse response = iam.listAccessKeys(request);

response.accessKeyMetadata().forEach(key -> {
    System.out.println("Access Key ID: " + key.accessKeyId());
    System.out.println("Status: " + key.status());
});
```

## 10. Production concerns
### Pagination
Mặc dù mỗi IAM user chỉ có tối đa 2 key, nhưng API này vẫn hỗ trợ `Marker` và `MaxItems`. Trong các script quét hàng loạt, hãy luôn xử lý trường hợp có nhiều hơn một trang kết quả để đảm bảo code bền vững.

### Rate Limiting
Gọi API này quá nhiều lần trong thời gian ngắn (ví dụ quét hàng ngàn user liên tục) có thể gây ra lỗi `Throttling`. Hãy sử dụng cơ chế exponential backoff khi gọi.

### Permissions
Để chạy lệnh này, IAM entity cần có quyền `iam:ListAccessKeys`. Tuy nhiên, nếu user tự liệt kê key của chính mình, họ vẫn cần quyền này trừ khi được cấp qua một policy cụ thể.

## 11. Common mistakes
- Mistake: Giả định rằng nếu key tồn tại là nó đang được sử dụng.
  Fix: Luôn kết hợp với `GetAccessKeyLastUsed` để biết key đó có thực sự "còn sống" hay không.

- Mistake: Không kiểm tra field `Status`.
  Fix: Một key có thể tồn tại nhưng ở trạng thái `Inactive`, đừng nhầm lẫn nó với key đang hoạt động.

## 12. Sample project
Viết một Lambda function (Python/Boto3) chạy hàng tuần:
1. Quét toàn bộ IAM users.
2. Với mỗi user, gọi `ListAccessKeys`.
3. Nếu phát hiện key nào có `CreateDate` cũ hơn 90 ngày, gửi cảnh báo qua Amazon SNS để yêu cầu user xoay vòng key.

## 13. Interview
### Core Q&A
1. Q: `ListAccessKeys` có trả về Secret Access Key không?
   A: Không. AWS không bao giờ lưu trữ Secret Access Key ở dạng có thể đọc lại sau khi khởi tạo. API này chỉ trả về metadata.

2. Q: Giới hạn số lượng Access Keys tối đa cho mỗi IAM User là bao nhiêu?
   A: Hiện tại là 2 keys (bao gồm cả Active và Inactive).

### Scenario
"Bạn được giao nhiệm vụ dọn dẹp các Access Keys cũ trong hệ thống. Quy trình của bạn là gì?"
-> Trả lời:
1. Dùng `ListAccessKeys` để lấy danh sách key và ngày tạo.
2. Dùng `GetAccessKeyLastUsed` để xem key có được dùng trong 30 ngày qua không.
3. Nếu key cũ và không dùng: Chuyển trạng thái sang `Inactive` bằng `UpdateAccessKey`.
4. Chờ 7 ngày: Nếu không có khiếu nại, thực hiện `DeleteAccessKey`.

## 14. References
- Official Docs: [IAM ListAccessKeys API Reference](https://docs.aws.amazon.com/IAM/latest/APIReference/API_ListAccessKeys.html)
- AWS CLI: [aws iam list-access-keys](https://awscli.amazonaws.com/v2/documentation/api/latest/reference/iam/list-access-keys.html)

## 15. Real-world Code
Nghiên cứu công cụ [CloudCustodian](https://cloudcustodian.io/) - một công cụ quản trị cloud mã nguồn mở, sử dụng API này rất nhiều để thực thi các chính sách bảo mật IAM.

## 16. Community
- Reddit: r/aws
- Stack Overflow: Tag [aws-iam]
- AWS Security Blog.
