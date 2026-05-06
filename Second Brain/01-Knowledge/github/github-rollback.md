---
created: 2026-05-06
tags:
  - "#type/concept"
  - "#status/draft"
  - "#lang/devops"
  - "#topic/ci-cd"
related:
  - "[[github-actions-cd.md]]"
  - "[[git-history.md]]"
---

## 1. What
GitHub Rollback là quá trình khôi phục lại trạng thái ổn định trước đó của mã nguồn hoặc môi trường triển khai sau khi phát hiện bản cập nhật mới nhất có lỗi. Có hai cấp độ rollback chính: Rollback Code (sử dụng Git) và Rollback Deployment (sử dụng GitHub Actions/Environments).

## 2. Why
Dù có quy trình test kỹ lưỡng đến đâu, lỗi vẫn có thể lọt lên Production. GitHub Rollback là "Phanh khẩn cấp" giúp:
- **Giảm thiểu thiệt hại (MTTR)**: Đưa hệ thống về trạng thái hoạt động bình thường nhanh nhất có thể.
- **An tâm triển khai**: Cho phép team tự tin deploy thường xuyên vì biết luôn có đường lui an toàn.
- **Giữ uy tín**: Tránh việc người dùng gặp lỗi trong thời gian dài.

## 3. Mental Model
Hãy tưởng tượng GitHub Rollback giống như tính năng **"System Restore"** trên Windows hoặc **"Nút Undo"** trong một trình soạn thảo:
- Bạn vừa gõ nhầm một đoạn văn bản làm hỏng bố cục trang giấy.
- Thay vì cố gắng xóa từng chữ và sửa lại (rất lâu và dễ sai thêm), bạn nhấn `Ctrl + Z` để quay lại ngay thời điểm trang giấy còn đẹp.

## 4. Where it fits
Vị trí trong quy trình xử lý sự cố:
`Incident Detected -> Alert -> Decision to Rollback -> GitHub Actions / Git Commands -> Verification -> Post-mortem`

## 5. When to use
- Khi bản deploy mới làm sập ứng dụng (Crash).
- Khi phát hiện lỗi logic nghiêm trọng ảnh hưởng đến thanh toán hoặc dữ liệu người dùng.
- Khi hiệu năng hệ thống giảm đột ngột sau khi update (Latency spike).

## 6. When NOT to use
- Khi lỗi nhỏ, không ảnh hưởng đến người dùng và có thể sửa nhanh bằng một "Hotfix" (triển khai bản vá mới).
- Khi bản deploy mới đã thực hiện thay đổi cấu trúc Database (Migration) không thể đảo ngược (Backward incompatible).

## 7. Trade-offs
| Pros | Cons |
|------|------|
| Tốc độ hồi phục cực nhanh. | Có thể làm mất các tính năng mới vừa deploy thành công. |
| Quy trình rõ ràng, ít rủi ro hơn sửa lỗi trực tiếp. | Có rủi ro về tính nhất quán dữ liệu nếu DB không rollback theo. |
| Có thể tự động hóa hoàn toàn. | Đòi hỏi phải lưu trữ (Artifact) các version cũ. |

## 8. Alternatives
| Option | So sánh |
|--------|---------|
| Hotfix | Viết code sửa lỗi rồi deploy bản mới. Chậm hơn nhưng giữ được các thay đổi khác. |
| Feature Flags | Tắt tính năng lỗi từ bảng điều khiển mà không cần rollback code/deploy. |

## 9. How
Các phương thức rollback phổ biến trên GitHub:

### Cách 1: Revert Commit (Sạch sẽ nhất cho lịch sử Git)
```bash
# Tìm mã hash của commit gây lỗi
git log
# Tạo một commit mới đảo ngược lại commit lỗi
git revert <commit_hash>
# Push lên nhánh main để trigger CI/CD deploy lại
git push origin main
```

### Cách 2: Re-run Production Deployment (Nhanh nhất)
1. Vào tab **Actions** trên GitHub.
2. Chọn workflow deployment thành công gần nhất.
3. Nhấn **Re-run all jobs**. 
*Lưu ý: Cách này yêu cầu script deploy phải dựa trên Tag hoặc Commit Hash cụ thể.*

### Cách 3: Sử dụng GitHub Environments
Vào **Settings -> Environments -> production**, kiểm tra danh sách các bản deployment cũ và kích hoạt lại bản ổn định.

## 10. Production concerns
### Database Migration
Đây là thách thức lớn nhất. Nếu bản code mới đã chạy `ALTER TABLE` thêm cột, việc rollback code cũ (vốn không biết có cột đó) có thể gây lỗi. Luôn thiết kế DB migration theo kiểu "Expand then Contract" để hỗ trợ rollback code an toàn.

### Artifact Persistence
Để rollback nhanh, bạn phải đảm bảo Docker Images hoặc file JAR của các version cũ vẫn còn được lưu trữ trên Registry (như ECR hoặc Docker Hub).

## 11. Common mistakes
- Mistake: Dùng `git push --force` để quay về bản cũ. Điều này sẽ làm hỏng lịch sử Git của cả team.
  Fix: Luôn dùng `git revert` hoặc re-deploy artifact cũ.

- Mistake: Rollback code nhưng quên rollback các biến môi trường (Secrets) nếu chúng đã bị thay đổi.

## 12. Sample project
Thiết lập một GitHub Action:
- Mỗi khi deploy thành công, tự động gắn một Git Tag kiểu `deploy-stable-<timestamp>`.
- Tạo một workflow "Manual Rollback" nhận đầu vào là tên Tag và thực hiện deploy lại Tag đó.

## 13. Interview
### Core Q&A
1. Q: Sự khác biệt giữa `git reset` và `git revert` trong quy trình rollback là gì?
   A: `git reset` xóa bỏ lịch sử commit (nguy hiểm cho team), còn `git revert` tạo một commit mới để đảo ngược thay đổi (an toàn và minh bạch).

2. Q: Làm sao để rollback Database khi thực hiện rollback code?
   A: Đây là vấn đề phức tạp. Tốt nhất nên viết migration có hỗ trợ `down()` hoặc thiết kế code sao cho tương thích với cả bản DB cũ và mới.

### Scenario
"Bạn phát hiện Production bị lỗi sau khi merge PR. Bạn sẽ chọn revert PR đó hay thực hiện hotfix?"
-> Trả lời: Nếu lỗi làm sập hệ thống (High impact), tôi sẽ chọn revert PR ngay lập tức để khôi phục dịch vụ. Nếu lỗi nhỏ và dễ sửa, tôi sẽ chọn hotfix để tránh làm gián đoạn các tính năng khác vừa được merge.

## 14. References
- GitHub Actions: [Re-running workflows](https://docs.github.com/en/actions/managing-workflow-runs/re-running-workflows-and-jobs)
- Git: [git-revert documentation](https://git-scm.com/docs/git-revert)

## 15. Real-world Code
Nghiên cứu các tool như ArgoCD hoặc Octopus Deploy để thấy cách họ quản lý "Rollback buttons" một cách chuyên nghiệp.

## 16. Community
- Reddit: r/DevOps.
- Stack Overflow: Tag [git-revert] [github-actions-deployment].
