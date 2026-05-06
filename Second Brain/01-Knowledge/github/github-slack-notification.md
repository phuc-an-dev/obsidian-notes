---
created: 2026-05-05
tags:
  - "#type/pattern"
  - "#status/draft"
  - "#lang/others"
  - "#topic/devops"
related:
  - "[[github-secrets]]"
  - "[[github-secret-injection]]"
  - "[[github-manual-deployment]]"
---

## 1. What
GitHub Slack Notification là việc tích hợp GitHub Actions với Slack thông qua Incoming Webhooks để tự động gửi thông báo về trạng thái của các quy trình CI/CD (Thành công, Thất bại, Đang chạy) vào một channel cụ thể. Slack Webhook URL được lưu trữ an toàn trong GitHub Secrets.

## 2. Why
Việc bắn thông báo về Slack giúp:
- **Tăng tính hiển thị (Visibility)**: Toàn bộ team biết được khi nào hệ thống được deploy mà không cần vào tab "Actions" của GitHub.
- **Phản ứng nhanh (MTTR)**: Phát hiện và sửa lỗi ngay lập tức nếu workflow bị fail trên Production.
- **Phối hợp team**: Tránh việc nhiều người cùng deploy chồng chéo hoặc biết được ai là người đã kích hoạt lệnh deploy.

## 3. Mental Model
Hãy tưởng tượng GitHub Actions là một **"Công trường xây dựng"**.
- Slack Webhook giống như một **"Chiếc bộ đàm"**.
- Khi một hạng mục hoàn thành hoặc gặp sự cố, người giám sát (Action) sẽ dùng bộ đàm để báo cáo về trung tâm điều hành (Slack Channel) cho mọi người cùng biết.

## 4. Where it fits
`GitHub Action Job -> Finish/Failure -> Slack API Call (via Webhook) -> Slack Channel`

## 5. When to use
- Deploy lên môi trường Production/Staging.
- Các job chạy định kỳ (Cron jobs) quan trọng như Backup DB.
- Khi cần thông báo cho các bên liên quan (Product Manager, QA) về phiên bản mới.

## 6. When NOT to use
- Các workflow chạy quá thường xuyên (ví dụ: mỗi lần save code) gây "loãng" thông tin (Chat fatigue).
- Các dự án cá nhân không cần sự phối hợp.

## 7. Trade-offs
| Pros | Cons |
|------|------|
| Cập nhật trạng thái thời gian thực. | Có thể gây nhiễu nếu cấu hình quá nhiều thông báo. |
| Dễ dàng setup (chỉ cần URL). | Nếu Webhook URL bị lộ, kẻ xấu có thể spam channel của bạn. |
| Hỗ trợ định dạng tin nhắn đẹp (Rich text, Buttons). | Phụ thuộc vào tính ổn định của Slack API. |

## 8. Alternatives
| Option | So sánh |
|--------|---------|
| GitHub Slack App | Chính chủ từ GitHub, nhiều tính năng hơn (slash commands) nhưng setup phức tạp hơn. |
| Email Notifications | Cổ điển, ít gây nhiễu hơn nhưng không mang tính thời gian thực. |
| Discord Webhooks | Tương tự Slack, thường dùng cho cộng đồng mã nguồn mở. |

## 9. How
### Step 1: Lấy Slack Webhook URL
1. Vào Slack Workspace -> `Apps` -> `Incoming Webhooks`.
2. Chọn channel và click `Add Incoming Webhooks integration`.
3. Copy đoạn URL (ví dụ: `https://hooks.slack.com/services/T.../B.../X...`).

### Step 2: Lưu vào GitHub Secrets
1. Repo -> `Settings` -> `Secrets and variables` -> `Actions`.
2. Click `New repository secret`.
3. Name: `SLACK_WEBHOOK_URL`.
4. Value: Dán đoạn URL vừa copy.

### Step 3: Cấu hình Workflow YAML
Sử dụng action phổ biến `rtCamp/action-slack-notify`:
```yaml
jobs:
  deploy:
    runs-on: ubuntu-latest
    steps:
      - name: Deploy Step
        run: echo "Deploying to Production..."
      
      - name: Slack Notification
        if: always() # Chạy ngay cả khi các bước trước fail
        uses: rtCamp/action-slack-notify@v2
        env:
          SLACK_WEBHOOK: ${{ secrets.SLACK_WEBHOOK_URL }}
          SLACK_CHANNEL: general
          SLACK_COLOR: ${{ job.status == 'success' && 'good' || 'danger' }}
          SLACK_MESSAGE: "Deploy status: ${{ job.status }}"
          SLACK_TITLE: "Deployment Report"
```

## 10. Production concerns
### Scaling
Sử dụng **Organization Secrets** để chia sẻ chung một Webhook URL cho tất cả các repo trong công ty nếu muốn thông báo dồn về một channel quản lý chung.

### Failure
Nếu Slack API bị down, workflow của bạn không nên bị dừng lại. Action gửi notification nên là bước cuối cùng và không ảnh hưởng đến logic deploy chính.

### Monitoring
Theo dõi số lượng tin nhắn gửi đi để tránh bị Slack giới hạn rate limit (thường rất cao, nhưng cần lưu ý nếu loop workflow).

## 11. Common mistakes
- **Mistake**: Hardcode trực tiếp Webhook URL vào file YAML.
  **Fix**: Luôn dùng GitHub Secrets.
- **Mistake**: Chỉ gửi thông báo khi thành công.
  **Fix**: Quan trọng nhất là phải gửi thông báo khi **Thất bại** (`if: failure()`) để xử lý kịp thời.

## 12. Sample project
Một hệ thống thông báo "xịn" hơn:
- Gửi tin nhắn kèm link tới commit, link tới log của Action và tên người thực hiện (`github.actor`).

## 13. Interview
### Core Q&A
1. Q: Làm thế nào để phân biệt tin nhắn từ Staging và Production trên Slack?
   A: Có thể dùng biến `env` để đổi màu (Color) hoặc thêm tiền tố (Prefix) vào tiêu đề tin nhắn dựa trên nhánh đang chạy.

2. Q: Tại sao nên dùng `rtCamp/action-slack-notify` thay vì tự viết `curl`?
   A: Action này hỗ trợ sẵn việc định dạng tin nhắn đẹp (màu sắc, icon, thông tin context) mà không cần phải xử lý chuỗi JSON phức tạp trong shell script.

### Scenario
**Tình huống**: Bạn muốn chỉ những đợt deploy Production mới bắn tin nhắn vào channel `#alerts`, còn các đợt khác bắn vào `#dev`.
**Giải quyết**: Sử dụng logic `if` trong workflow hoặc tạo 2 secrets khác nhau và gán dựa trên môi trường (Environment Secrets).

## 14. References
- Slack API: [Sending messages using Incoming Webhooks](https://api.slack.com/messaging/webhooks)
- GitHub Marketplace: [rtCamp/action-slack-notify](https://github.com/marketplace/actions/slack-notify)

## 15. Real-world Code
N/A

## 16. Community
- Slack Community Forum
- GitHub Actions Community
