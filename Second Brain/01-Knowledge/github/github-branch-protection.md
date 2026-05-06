---
created: 2026-05-05
tags:
  - "#type/concept"
  - "#status/draft"
  - "#lang/devops"
  - "#topic/github"
related:
  - "[[github-collaborators]]"
  - "[[github-teams]]"
  - "[[github-actions-workflow-protection]]"
---

## 1. What
GitHub Branch Protection là tập hợp các quy tắc (rules) được áp dụng cho một nhánh cụ thể trong repository (thường là main hoặc master) nhằm kiểm soát cách các thay đổi được merge vào nhánh đó, ngăn chặn việc xóa hoặc sửa đổi lịch sử commit trái phép.

## 2. Why
Trước khi có Branch Protection, bất kỳ ai có quyền Write cũng có thể vô tình `git push --force` làm mất lịch sử code, hoặc merge code chưa qua kiểm thử trực tiếp vào môi trường production. Điều này gây ra rủi ro cực lớn về tính ổn định của sản phẩm và sự tin cậy của mã nguồn.

## 3. Mental Model
Hãy tưởng tượng nhánh `main` giống như một đại lộ chính dẫn thẳng tới trung tâm thành phố. Branch Protection giống như một trạm kiểm soát (Checkpost) ở đầu đại lộ. Bạn không thể lái xe thẳng vào; bạn phải dừng lại để cảnh sát kiểm tra giấy phép lái xe (Code Review), kiểm tra an toàn xe (CI/CD Status Checks) và đôi khi cần có sự xác nhận từ ban quản lý (Signed Commits) thì cửa mới mở.

## 4. Where it fits
Nằm trong phần cài đặt quản lý mã nguồn: `Repository Settings -> Code and automation -> Branches -> Branch protection rules`. Nó đóng vai trò là lớp bảo vệ cuối cùng trước khi code đi vào các nhánh quan trọng.

## 5. When to use
- Luôn dùng cho nhánh `main`, `master`, `production`.
- Dùng cho các nhánh `develop` hoặc `release` trong các dự án lớn có nhiều team cùng tham gia.
- Khi cần tuân thủ các quy chuẩn bảo mật (Compliance) yêu cầu code phải được ít nhất 2 người review.

## 6. When NOT to use
- Các repository cá nhân, dự án demo chỉ có một mình bạn làm.
- Các nhánh feature tạm thời (short-lived feature branches) vì sẽ làm chậm tốc độ phát triển không cần thiết.
- Khi team cực nhỏ (2 người) và sự tin tưởng tuyệt đối, tuy nhiên vẫn khuyến khích dùng ở mức tối thiểu.

## 7. Trade-offs
| Pros | Cons |
|------|------|
| Ngăn chặn triệt để lỗi do con người (force push, accidental delete). | Tăng thêm thời gian chờ đợi (waiting time) để được merge code. |
| Đảm bảo chất lượng code qua quy trình review bắt buộc. | Có thể gây tắc nghẽn (bottleneck) nếu reviewer bận. |
| Tích hợp chặt chẽ với CI/CD để tự động hóa việc kiểm thử. | Cần cấu hình và bảo trì các Status Checks thường xuyên. |

## 8. Alternatives
| Option | So sánh |
|--------|---------|
| Repository Rulesets | Phiên bản mới, linh hoạt hơn, có thể áp dụng cho nhiều repo cùng lúc (Organization-wide). |
| Git Hooks (pre-receive) | Cấu hình ở mức server git, tùy biến cao nhưng khó quản lý hơn UI GitHub. |
| CODEOWNERS | Chỉ định cụ thể ai phải review file nào, kết hợp tốt với Branch Protection. |

## 9. How
Cấu hình Branch Protection qua GitHub API để tự động hóa:
```bash
# JSON payload ví dụ để enable branch protection
curl -X PUT -H "Authorization: token $GITHUB_TOKEN" \
  https://api.github.com/repos/:owner/:repo/branches/main/protection \
  -d '{
    "required_status_checks": { "strict": true, "contexts": ["ci/circleci"] },
    "enforce_admins": true,
    "required_pull_request_reviews": { "required_approving_review_count": 1 },
    "restrictions": null
  }'
```

## 10. Production concerns
### Scaling
Với quy mô hàng ngàn repo, hãy sử dụng GitHub Repository Rulesets thay vì Branch Protection truyền thống để có thể áp dụng một quy tắc chung cho toàn bộ Organization chỉ với một lần cấu hình.

### Failure
Nếu hệ thống CI/CD bị lỗi và không gửi status "Success" về GitHub, developer sẽ không thể merge code dù code hoàn toàn đúng. Cần có cơ chế "Bypass" cho Admin trong trường hợp khẩn cấp (Emergency Hotfix).

### Monitoring
Theo dõi Audit Logs để xem ai đã thay đổi cấu hình bảo vệ nhánh hoặc ai thường xuyên bypass các quy tắc này.

## 11. Common mistakes
- Mistake: Không yêu cầu "Status Checks" (CI) phải pass trước khi merge.
  Fix: Luôn tích hợp GitHub Actions hoặc CircleCI và đánh dấu chúng là "Required" trong Branch Protection.

- Mistake: Cho phép Admin bypass các quy tắc bảo vệ.
  Fix: Tick vào ô "Enforce admins" để đảm bảo ngay cả quản trị viên cũng phải tuân thủ quy trình review.

## 12. Sample project
Thiết lập một quy trình làm việc chuẩn: 
1. Tạo branch mới từ `main`.
2. Push code lên và tạo Pull Request.
3. Chờ GitHub Action chạy test tự động.
4. Chờ ít nhất một đồng nghiệp Approve.
5. Chỉ khi đủ các điều kiện trên, nút "Merge" mới hiện màu xanh để bạn nhấn.

## 13. Interview
### Core Q&A
1. Q: Tại sao chúng ta nên dùng "Required Status Checks"?
   A: Để đảm bảo rằng chỉ có code đã vượt qua các bài kiểm tra tự động (unit test, linting, security scan) mới được phép merge vào code chính, giảm thiểu bug phát sinh trên production.

2. Q: Sự khác biệt giữa Branch Protection truyền thống và Repository Rulesets là gì?
   A: Rulesets mạnh mẽ hơn vì có thể áp dụng cho nhiều repo cùng lúc, hỗ trợ chế độ "Evaluate" (chạy thử nghiệm mà không chặn thật) và có nhiều tiêu chí lọc nhánh linh hoạt hơn (ví dụ: áp dụng cho tất cả nhánh bắt đầu bằng `release/*`).

### Scenario
Tình huống: "Một developer trong team vô tình force push lên nhánh main làm mất 3 ngày làm việc của team. Bạn sẽ làm gì để điều này không lặp lại?"
Giải pháp: Tôi sẽ kích hoạt Branch Protection Rule cho nhánh `main`. Đầu tiên là cấm "Force Push" và cấm "Delete branch". Sau đó, tôi yêu cầu mọi thay đổi phải thông qua Pull Request với ít nhất một lượt review và tất cả các Status Checks từ hệ thống CI phải thành công. Cuối cùng, tôi sẽ bật "Enforce Admins" để đảm bảo sự kỷ luật tuyệt đối cho toàn team.

## 14. References
- Official Docs: https://docs.github.com/en/repositories/configuring-branches-and-merges-in-your-repository/defining-the-mergeability-of-pull-requests/about-protected-branches
- GitHub Repo: https://github.com/features/actions
- Spec / RFC: GitHub API Documentation for Branch Protection
- Changelog: Introduction of Repository Rulesets (2023)

## 15. Real-world Code
Ví dụ Terraform để setup Branch Protection:
```hcl
resource "github_branch_protection" "main" {
  repository_id = github_repository.example.node_id
  pattern       = "main"

  required_status_checks {
    strict   = true
    contexts = ["build", "test"]
  }

  required_pull_request_reviews {
    dismiss_stale_reviews = true
    required_approving_review_count = 1
  }
}
```

## 16. Community
- Reddit: r/devops - thảo luận về git workflows.
- Stack Overflow: tags [github] [branch-protection].
- Blog: Martin Fowler's blog on Continuous Integration.
- Talk: GitHub Universe: Secure your supply chain.
