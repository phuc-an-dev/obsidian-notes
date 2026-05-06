---
created: 2026-05-06
tags:
  - "#type/library"
  - "#status/draft"
  - "#lang/devops"
  - "#topic/organizations"
related:
  - "[[iam.md]]"
---

## 1. What
`AcceptHandshake` là một API thuộc dịch vụ AWS Organizations, được sử dụng để chấp nhận một yêu cầu "bắt tay" (handshake) được gửi từ một tài khoản khác. Handshake là cơ chế bảo mật của AWS để thiết lập mối quan hệ giữa các tài khoản, chẳng hạn như mời một tài khoản tham gia vào một tổ chức (Organization).

## 2. Why
Trong quản lý đa tài khoản (Multi-account management), việc tự ý thêm một tài khoản vào tổ chức mà không có sự đồng ý của chủ sở hữu tài khoản đó là một rủi ro bảo mật. `AcceptHandshake` ra đời để tạo ra một quy trình xác nhận hai bước (Two-step verification): Một bên gửi lời mời (Invite) và bên kia phải chủ động chấp nhận (Accept), đảm bảo tính minh bạch và quyền kiểm soát của mỗi chủ tài khoản.

## 3. Mental Model
Hãy tưởng tượng `AcceptHandshake` giống như việc **"Chấp nhận lời mời kết bạn"** trên Facebook:
- Tài khoản quản trị (Management Account) gửi một lời mời kết bạn (Handshake).
- Bạn (Member Account) nhận được thông báo nhưng chưa trở thành bạn bè ngay lập tức.
- Bạn phải nhấn nút "Chấp nhận" (`AcceptHandshake`) thì mối quan hệ mới chính thức được thiết lập và bạn mới bắt đầu chia sẻ các quyền lợi/nghĩa vụ trong tổ chức.

## 4. Where it fits
Vị trí trong luồng quy trình:
`Management Account -> InviteAccount -> Handshake Created (PENDING) -> Member Account -> AcceptHandshake -> Handshake (ACCEPTED) -> Account joined Organization`

## 5. When to use
- Khi bạn nhận được lời mời gia nhập một AWS Organization từ đối tác hoặc phòng ban khác.
- Khi tổ chức muốn nâng cấp từ "Consolidated Billing" sang "All Features" (cần các tài khoản thành viên chấp nhận handshake `ENABLE_ALL_FEATURES`).
- Khi có sự chuyển giao trách nhiệm thanh toán hoặc quản lý giữa các tổ chức.

## 6. When NOT to use
- Khi bạn muốn từ chối lời mời (trường hợp này dùng `DeclineHandshake`).
- Khi bạn là người gửi lời mời (trường hợp này dùng `InviteAccount`).
- Khi tài khoản đã nằm sẵn trong tổ chức và không có yêu cầu thay đổi nào đang chờ xử lý.

## 7. Trade-offs
| Pros | Cons |
|------|------|
| Đảm bảo tính đồng thuận giữa các chủ tài khoản. | Quy trình thủ công có thể làm chậm việc mở rộng tổ chức nếu không tự động hóa. |
| Cơ chế bảo mật mạnh mẽ, tránh việc bị "ép buộc" gia nhập tổ chức lạ. | Handshake sẽ hết hạn sau một khoảng thời gian nếu không được chấp nhận. |
| Cho phép kiểm tra kỹ các điều khoản trước khi đồng ý. | |

## 8. Alternatives
| Option | So sánh |
|--------|---------|
| CreateAccount | Tạo tài khoản mới trực tiếp bên trong Organization, không cần qua bước Handshake (vì Organization sở hữu tài khoản đó ngay từ đầu). |
| DeclineHandshake | Dùng để từ chối nếu lời mời không mong muốn. |

## 9. How
Sử dụng AWS CLI để chấp nhận một handshake:

```bash
# 1. Liệt kê các handshake đang chờ (PENDING)
aws organizations list-handshakes-for-account

# 2. Chấp nhận handshake bằng ID
aws organizations accept-handshake --handshake-id h-examplehandshakeid111
```

Ví dụ cấu hình IAM policy để cho phép hành động này:
```json
{
    "Version": "2012-10-17",
    "Statement": [
        {
            "Effect": "Allow",
            "Action": "organizations:AcceptHandshake",
            "Resource": "*"
        }
    ]
}
```

## 10. Production concerns
### Handshake Expiration
Các handshake không tồn tại vĩnh viễn. Nếu không được chấp nhận trong vòng vài ngày, chúng sẽ tự động hết hạn và bên gửi phải gửi lại lời mời mới.

### Permissions
Chỉ người dùng hoặc role có quyền `organizations:AcceptHandshake` mới có thể thực hiện lệnh này. Thường thì quyền này được cấp cho admin của tài khoản thành viên.

### Visibility
Sau khi đã được chấp nhận, thông tin về handshake vẫn có thể truy cập được thông qua API trong khoảng 30 ngày trước khi bị hệ thống xóa bỏ hoàn toàn.

## 11. Common mistakes
- Mistake: Thử chấp nhận một handshake đã hết hạn (EXPIRED) hoặc đã bị hủy (CANCELED).
  Fix: Luôn chạy `list-handshakes-for-account` để kiểm tra trạng thái hiện tại trước khi gọi lệnh accept.

- Mistake: Quên rằng việc gia nhập Organization có thể thay đổi phương thức thanh toán của tài khoản.
  Fix: Luôn kiểm tra loại handshake (INVITE hay ENABLE_ALL_FEATURES) để hiểu rõ hệ quả về tài chính.

## 12. Sample project
Tự động hóa việc gia nhập Organization:
Viết một script chạy định kỳ trên các tài khoản vệ tinh, nếu thấy có handshake `INVITE` từ ID của tài khoản quản trị công ty, script sẽ tự động gọi `AcceptHandshake` để hoàn tất việc onboarding.

## 13. Interview
### Core Q&A
1. Q: Trạng thái của Handshake sau khi gọi `AcceptHandshake` thành công là gì?
   A: Trạng thái sẽ chuyển từ `PENDING` sang `ACCEPTED`.

2. Q: Có thể chấp nhận handshake của tài khoản khác không?
   A: Không. Bạn chỉ có thể chấp nhận handshake được gửi đích danh đến tài khoản mà bạn đang có quyền truy cập.

### Scenario
"Bạn nhận được một handshake mời gia nhập Organization lạ, bạn sẽ làm gì?"
-> Trả lời: 
1. Không chấp nhận ngay lập tức. 
2. Sử dụng `DescribeHandshake` để xem thông tin chi tiết về bên gửi (Management Account ID) và các điều khoản đính kèm.
3. Nếu không xác minh được nguồn gốc, tôi sẽ dùng `DeclineHandshake` để đảm bảo an toàn cho tài khoản.

## 14. References
- AWS API Reference: [AcceptHandshake](https://docs.aws.amazon.com/organizations/latest/APIReference/API_AcceptHandshake.html)
- AWS CLI Command: [accept-handshake](https://awscli.amazonaws.com/v2/documentation/api/latest/reference/organizations/accept-handshake.html)

## 15. Real-world Code
Nghiên cứu cách các công cụ Landing Zone (như AWS Control Tower) quản lý handshakes khi thực hiện quy trình "Enroll Account".

## 16. Community
- Reddit: r/aws
- AWS Organizations Forum.
- Stack Overflow: Tag [aws-organizations].
