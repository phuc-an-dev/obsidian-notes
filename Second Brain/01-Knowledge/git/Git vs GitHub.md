---
created: 2026-04-30
tags:
  - "#type/concept"
  - "#status/draft"
  - "#lang/devops"
  - "#topic/version-control"
related:
  - "[[Common Git Commands]]"
  - "[[GitHub Actions]]"
  - "[[Pull Request in Git]]"
---

## 1. What
Git là một phần mềm quản lý phiên bản (Version Control System) cài đặt cục bộ trên máy tính. GitHub là một nền tảng dịch vụ lưu trữ trên đám mây (Cloud-based hosting service) cho các kho lưu trữ Git.

## 2. Why
Git ra đời để quản lý code và lịch sử thay đổi cục bộ. Tuy nhiên, để làm việc nhóm hiệu quả, cần một nơi trung tâm để chia sẻ code, thảo luận và quản lý dự án. GitHub ra đời để giải quyết nhu cầu kết nối các lập trình viên sử dụng Git trên toàn thế giới.

## 3. Mental Model
Hãy tưởng tượng:
- **Git** giống như Microsoft Word trên máy tính của bạn. Bạn dùng nó để viết, chỉnh sửa và lưu các bản nháp (v1, v2, Final).
- **GitHub** giống như Google Drive hoặc OneDrive. Bạn tải file Word lên đó để người khác cùng xem, nhận xét và cùng chỉnh sửa.

## 4. Where it fits
Local Machine (Git) <-> Internet (HTTP/SSH) <-> Remote Server (GitHub)

## 5. When to use
- **Dùng Git**: Luôn luôn, ngay khi bạn bắt đầu gõ dòng code đầu tiên để bảo vệ lịch sử làm việc của mình.
- **Dùng GitHub**: Khi bạn muốn sao lưu code lên cloud, chia sẻ với đồng nghiệp hoặc đóng góp cho các dự án mã nguồn mở.

## 6. When NOT to use
- Không dùng GitHub cho các dự án chứa dữ liệu cực kỳ nhạy cảm của doanh nghiệp mà không có các biện pháp bảo mật/private repo nghiêm ngặt (có thể dùng self-hosted GitLab thay thế).
- Không nhầm lẫn GitHub là công cụ duy nhất để lưu trữ Git (còn có GitLab, Bitbucket, Azure DevOps).

## 7. Trade-offs
| Đặc điểm | Git | GitHub |
|----------|-----|--------|
| **Bản chất** | Công cụ (Tool) | Dịch vụ (Service) |
| **Cài đặt** | Cài cục bộ trên máy | Truy cập qua trình duyệt |
| **Tính năng chính** | Quản lý phiên bản, nhánh, gộp | Lưu trữ, Pull Request, Issues, Actions |
| **Internet** | Không cần | Bắt buộc để đồng bộ |

## 8. Alternatives
| Git Alternatives | GitHub Alternatives |
|------------------|---------------------|
| SVN, Mercurial | GitLab, Bitbucket, Gitea, SourceForge |

## 9. How
```bash
# Git: Làm việc cục bộ
git init
git commit -m "Local change"

# GitHub: Kết nối với remote
git remote add origin https://github.com/user/repo.git
git push -u origin main
```

## 10. Production concerns
### Scaling
GitHub cung cấp GitHub Enterprise cho các tổ chức lớn với các tính năng bảo mật và quản lý nâng cao.

### Failure
Nếu GitHub sập, bạn vẫn có đầy đủ lịch sử code trên máy cục bộ nhờ kiến trúc phân tán của Git. Bạn có thể đẩy code sang một server khác bất cứ lúc nào.

### Monitoring
GitHub cung cấp các công cụ như GitHub Actions (CI/CD), Dependabot (quét lỗ hổng bảo mật) để giám sát dự án.

## 11. Common mistakes
- Mistake: Nghĩ rằng Git và GitHub là một.
  Fix: Hiểu rằng Git có thể hoạt động mà không cần GitHub.

- Mistake: Nghĩ rằng mọi code trên GitHub đều là mã nguồn mở.
  Fix: GitHub có hỗ trợ Private Repositories cho các dự án cá nhân hoặc doanh nghiệp.

## 12. Sample project
Tạo một repository trên GitHub, thực hiện clone về máy, commit code và push ngược lại để thấy được sự tương tác giữa local (Git) và remote (GitHub).

## 13. Interview
### Core Q&A
1. Q: Tôi có thể sử dụng Git mà không cần GitHub không?
   A: Có, hoàn toàn được. Bạn có thể quản lý phiên bản hoàn toàn cục bộ trên máy mình hoặc sử dụng các nền tảng khác như GitLab, Bitbucket.

2. Q: Tính năng quan trọng nhất của GitHub mà Git không có là gì?
   A: Đó là Pull Request (hoặc Merge Request). Đây là cơ chế cho phép thảo luận, review code và kiểm duyệt trước khi gộp code vào nhánh chính, điều mà Git thuần túy không cung cấp sẵn.

### Scenario
Tình huống: Một ứng viên nói "Tôi đã push code lên Git". Bạn đánh giá câu nói này thế nào?
Trả lời: Câu nói này chưa chính xác về thuật ngữ. Đúng ra phải là "Tôi đã commit code vào Git (local)" hoặc "Tôi đã push code lên GitHub/Remote". Việc phân biệt rõ ràng công cụ và nơi lưu trữ cho thấy sự hiểu biết căn bản của lập trình viên.

## 14. References
- Git Official: https://git-scm.com/
- GitHub Guides: https://guides.github.com/
- Difference Overview: https://www.geeksforgeeks.org/git-vs-github/

## 15. Real-world Code
Hầu hết các dự án trong `Second Brain` của bạn đang được lưu trữ theo cấu trúc Git và có thể đồng bộ lên GitHub.

## 16. Community
- GitHub Universe (Conference)
- GitHub Education
- Reddit: r/github
