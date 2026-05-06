---
created: 2026-05-05
tags:
  - "#type/concept"
  - "#status/draft"
  - "#lang/others"
  - "#topic/devops"
related:
  - "[[github-actions-workflow-protection]]"
  - "[[github-repository-roles]]"
---

## 1. What
Deploy manual trên GitHub là quá trình kích hoạt việc triển khai mã nguồn lên các môi trường (Staging, Production) thông qua sự can thiệp trực tiếp của con người thay vì tự động hoàn toàn. Hai cơ chế chính thường được dùng là `workflow_dispatch` (nút bấm tay) và `Environment Approvals` (phê duyệt thủ công).

## 2. Why
Trong môi trường doanh nghiệp hoặc dự án lớn, việc tự động deploy 100% (Continuous Deployment) đôi khi mang lại rủi ro:
- **Kiểm soát thời điểm**: Không muốn deploy vào giờ cao điểm hoặc ngày nghỉ (Friday Deploy).
- **Phê duyệt cuối cùng**: Cần Manager hoặc QA xác nhận lại kết quả test trước khi "Go Live".
- **Tính tuân thủ (Compliance)**: Các quy trình bảo mật yêu cầu phải có bằng chứng về việc phê duyệt thủ công.
- **Tiết kiệm tài nguyên**: Chỉ deploy các feature cụ thể khi cần thiết để test.

## 3. Mental Model
Hãy tưởng tượng quy trình này giống như **"Nút bấm phóng tên lửa"**. 
- Hệ thống CI (kiểm tra tên lửa, nhiên liệu) có thể chạy tự động.
- Nhưng lệnh "Phóng" (Deploy) phải do một người có thẩm quyền nhấn nút sau khi đã kiểm tra mọi thông số cuối cùng.

## 4. Where it fits
Vị trí trong luồng CI/CD:
`Code -> CI (Build/Test) -> Artifacts -> [Dừng lại chờ] -> Manual Trigger -> CD (Deploy)`

## 5. When to use
- Deploy lên môi trường Production.
- Các tác vụ bảo trì định kỳ (Dọn dẹp DB, Backup).
- Rollback về phiên bản cũ khi có sự cố.
- Chạy các script migration dữ liệu quan trọng.

## 6. When NOT to use
- Deploy lên môi trường Development/Sandbox: Nên tự động hoàn toàn để tăng tốc độ phát triển.
- Các project cá nhân nhỏ không đòi hỏi tính an toàn cao.
- Khi tần suất deploy quá lớn (ví dụ: 50 lần/ngày), manual sẽ trở thành nút thắt cổ chai.

## 7. Trade-offs
| Pros | Cons |
|------|------|
| Độ an toàn và tin cậy cao. | Làm chậm tốc độ đưa sản phẩm ra thị trường (Time to market). |
| Tránh được các sự cố tự động dây chuyền. | Tốn công sức con người (Manual effort). |
| Có cơ hội kiểm tra lần cuối (Sanity check). | Có thể quên không deploy dẫn đến lệch version giữa các môi trường. |

## 8. Alternatives
| Option | So sánh |
|--------|---------|
| Continuous Deployment | Tự động hoàn toàn. Nhanh nhưng rủi ro nếu bộ test không bao phủ 100%. |
| Git Tag Trigger | Deploy khi có tag mới. Vừa là auto vừa là manual (vì phải tạo tag tay). |
| ChatOps (Slack/Discord) | Deploy bằng câu lệnh trong chat. Tiện lợi nhưng cần setup bot phức tạp. |

## 9. How
### Cơ chế 1: `workflow_dispatch` (Manual Button)
Thêm vào file YAML để hiện nút "Run workflow" trên UI:
```yaml
name: Manual Deploy
on:
  workflow_dispatch:
    inputs:
      environment:
        description: 'Môi trường cần deploy'
        required: true
        default: 'staging'
        type: choice
        options:
          - staging
          - production
jobs:
  deploy:
    runs-on: ubuntu-latest
    steps:
      - name: Deploying to ${{ github.event.inputs.environment }}
        run: echo "Deploying..."
```

### Cơ chế 2: GitHub Environments (Approval Gate)
1. Vào `Settings -> Environments -> Create environment (ví dụ: Production)`.
2. Tích chọn `Required reviewers` và chọn người/team có quyền duyệt.
3. Trong file YAML, gán job vào environment đó:
```yaml
jobs:
  deploy:
    environment: production
    runs-on: ubuntu-latest
    steps:
      - run: echo "Waiting for approval..."
```

## 10. Production concerns
### Scaling
- Sử dụng **Deployment Protection Rules** (Enterprise) để tích hợp với các công cụ bên thứ ba (ví dụ: chỉ cho deploy nếu ticket Jira đã được duyệt).

### Failure
- Nếu người phê duyệt (Reviewer) vắng mặt: Cần thiết lập ít nhất 2-3 người có quyền duyệt để tránh đình trệ.

### Monitoring
- Theo dõi **Deployment Dashboard** trong tab "Actions" để thấy lịch sử ai đã nhấn nút deploy và thời điểm nào.

## 11. Common mistakes
- **Mistake**: Không giới hạn ai được quyền chạy `workflow_dispatch`. Mọi người có quyền `Write` đều có thể deploy Production.
  **Fix**: Kết hợp với **Environment Protection Rules** để khóa quyền thực thi.

- **Mistake**: Deploy manual từ một nhánh (branch) chưa được merge vào `main`.
  **Fix**: Cấu hình workflow chỉ cho phép chạy manual trên nhánh `main`.

## 12. Sample project
Một workflow hoàn chỉnh kết hợp CI tự động và CD thủ công:
- Khi push code: Tự động chạy Unit Test.
- Nếu test pass: Hiện nút bấm để deploy lên Production.

## 13. Interview
### Core Q&A
1. Q: Làm thế nào để truyền tham số (ví dụ: version) khi deploy manual?
   A: Sử dụng `inputs` trong `workflow_dispatch`. Người dùng sẽ nhập version vào một ô input trên UI trước khi chạy.

2. Q: Sự khác biệt giữa `workflow_dispatch` và `repository_dispatch`?
   A: `workflow_dispatch` kích hoạt từ UI hoặc API của GitHub. `repository_dispatch` thường được kích hoạt bởi các webhook từ bên ngoài GitHub.

### Scenario
**Tình huống**: Bạn muốn deploy lên Production nhưng hệ thống CI đang bị lỗi một test case không quan trọng. Bạn có nên dùng Manual Deploy để bypass không?
**Giải quyết**: Tuyệt đối không nên bypass test bằng cách manual nếu không thực sự khẩn cấp. Cách đúng là fix test hoặc đánh dấu test đó là `skipped` nếu nó thực sự không quan trọng, sau đó mới deploy theo quy trình chuẩn để có audit trail.

## 14. References
- Official Docs: [Events that trigger workflows](https://docs.github.com/en/actions/using-workflows/events-that-trigger-workflows#workflow_dispatch)
- Official Docs: [Using environments for deployment](https://docs.github.com/en/actions/deployment/targeting-different-environments/using-environments-for-deployment)

## 15. Real-world Code
Nhiều công ty sử dụng mẫu: `CI -> Build Image -> Update Manifest Repo (ArgoCD/Flux)` trong đó bước Update Manifest là Manual Trigger.

## 16. Community
- Stack Overflow: Tag `github-actions-dispatch`
- Blog: `Deployment best practices on GitHub`
