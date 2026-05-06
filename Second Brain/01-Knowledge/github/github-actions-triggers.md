---
created: 2026-05-06
tags:
  - "#type/concept"
  - "#status/draft"
  - "#lang/devops"
  - "#topic/ci-cd"
related:
  - "[[github-actions-ci.md]]"
  - "[[github-actions-cd.md]]"
---

## 1. What
GitHub Actions Workflow Triggers (sử dụng từ khóa `on` trong file YAML) là các sự kiện cụ thể dùng để kích hoạt một quy trình công việc (workflow). Các trigger này xác định khi nào và dưới điều kiện nào thì các jobs trong workflow sẽ bắt đầu thực thi.

## 2. Why
Một hệ thống CI/CD không thể tự chạy nếu không biết khi nào cần chạy. Trước khi có các trigger linh hoạt, lập trình viên phải kích hoạt thủ công hoặc dùng các webhook phức tạp. GitHub Triggers cung cấp một cơ chế tự động hóa mạnh mẽ, giúp tiết kiệm tài nguyên bằng cách chỉ chạy workflow khi thực sự cần thiết (ví dụ: chỉ khi code được merge vào nhánh chính).

## 3. Mental Model
Hãy tưởng tượng Workflow Triggers giống như các **"Cảm biến thông minh"** trong một ngôi nhà:
- **Push/PR**: Cảm biến ở cửa, mỗi khi có người bước vào (code mới đến), hệ thống sẽ bật đèn (chạy CI).
- **Schedule**: Đồng hồ báo thức, cứ đúng 7h sáng hệ thống sẽ tự động tưới cây (chạy Daily Build).
- **Manual**: Nút bấm vật lý trên tường, bạn chỉ nhấn khi thực sự muốn bật máy pha cà phê (chạy Deploy).

## 4. Where it fits
Vị trí trong file cấu hình:
`on: [Sự kiện] -> filter (nhánh, tag, đường dẫn) -> jobs -> steps`

## 5. When to use
- Khi muốn kiểm tra code ngay khi có Pull Request (`pull_request`).
- Khi muốn triển khai ứng dụng ngay khi code được merge vào `main` (`push`).
- Khi cần dọn dẹp database hoặc chạy quét bảo mật định kỳ (`schedule`).
- Khi cần chạy lại một bước triển khai cụ thể mà không cần thay đổi code (`workflow_dispatch`).

## 6. When NOT to use
- Tránh sử dụng các trigger quá rộng (ví dụ: chạy toàn bộ test cho mọi lần push lên bất kỳ nhánh nào) vì sẽ gây lãng phí phút build (Build minutes).
- Không dùng `schedule` cho các tác vụ yêu cầu phản hồi tức thời.

## 7. Trade-offs
| Pros | Cons |
|------|------|
| Tự động hóa hoàn toàn, không cần con người can thiệp. | Dễ gây lãng phí tài nguyên nếu cấu hình filter không kỹ. |
| Hỗ trợ nhiều kịch bản phức tạp (Matrix, Dependencies). | Khó debug nếu sự kiện trigger từ bên ngoài (Webhook). |
| Tích hợp sâu với mọi hoạt động trên GitHub. | Giới hạn về tần suất chạy (ví dụ schedule tối thiểu 5 phút). |

## 8. Alternatives
| Option | So sánh |
|--------|---------|
| Webhooks | Linh hoạt hơn nhưng yêu cầu bạn tự dựng server để lắng nghe và xử lý. |
| External CI Tools | (Jenkins, CircleCI) Có trigger riêng nhưng không mượt mà bằng "hàng chính chủ" GitHub. |

## 9. How
Các loại trigger phổ biến nhất:

### Sự kiện Git (Webhook events)
```yaml
on:
  push:
    branches: [ main ]
    paths:
      - 'src/**'
  pull_request:
    types: [opened, synchronize, reopened]
```

### Lịch trình (Scheduled events)
```yaml
on:
  schedule:
    - cron: '0 0 * * *' # Chạy vào 00:00 mỗi ngày
```

### Thủ công (Manual events)
```yaml
on:
  workflow_dispatch:
    inputs:
      environment:
        description: 'Môi trường triển khai'
        required: true
        default: 'staging'
```

## 10. Production concerns
### Rate Limiting
Sử dụng `concurrency` để ngăn chặn việc nhiều workflow chạy đè lên nhau khi có nhiều push liên tục:
```yaml
concurrency:
  group: ${{ github.workflow }}-${{ github.ref }}
  cancel-in-progress: true
```

### Filters
Luôn sử dụng `paths` hoặc `paths-ignore` để tránh chạy CI khi chỉ thay đổi file `README.md` hoặc tài liệu.

## 11. Common mistakes
- Mistake: Sử dụng `on: push` mà không giới hạn nhánh, dẫn đến việc chạy CI cho cả các nhánh rác.
  Fix: Luôn chỉ định `branches: [ main, develop ]`.

- Mistake: Viết sai cú pháp Cron (GitHub dùng chuẩn 5 trường nhưng có một số giới hạn về múi giờ UTC).
  Fix: Luôn kiểm tra cron tại [crontab.guru](https://crontab.guru/) và nhớ rằng nó chạy theo giờ UTC.

## 12. Sample project
Thiết lập một workflow "Security Scan":
1. Chạy tự động vào 2h sáng mỗi Chủ Nhật (`schedule`).
2. Có thể kích hoạt thủ công bất cứ lúc nào từ giao diện GitHub (`workflow_dispatch`).
3. Chỉ quét các file trong thư mục `src/`.

## 13. Interview
### Core Q&A
1. Q: Làm thế nào để chạy một workflow chỉ khi một workflow khác đã hoàn thành thành công?
   A: Sử dụng trigger `workflow_run`.

2. Q: `repository_dispatch` khác gì với `workflow_dispatch`?
   A: `workflow_dispatch` kích hoạt thủ công từ giao diện GitHub. `repository_dispatch` kích hoạt thông qua một API call từ bên ngoài GitHub.

### Scenario
"Bạn muốn workflow CD chỉ chạy khi có một thẻ (tag) mới được push lên theo định dạng `v1.0.0`. Bạn cấu hình thế nào?"
-> Trả lời: 
```yaml
on:
  push:
    tags:
      - 'v*'
```

## 14. References
- Official Docs: [Events that trigger workflows](https://docs.github.com/en/actions/using-workflows/events-that-trigger-workflows)
- Webhook events: [Webhook events and payloads](https://docs.github.com/en/webhooks/webhook-events-and-payloads)

## 15. Real-world Code
Nghiên cứu file `.github/workflows` của các repo lớn để thấy cách họ dùng `workflow_dispatch` để cung cấp các tùy chọn debug cho cộng đồng.

## 16. Community
- Reddit: r/GitHubActions
- GitHub Community Forum.
- Stack Overflow: Tag [github-actions].
