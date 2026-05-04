---
created: 2026-04-30
tags:
  - "#type/concept"
  - "#status/draft"
  - "#lang/devops"
  - "#topic/version-control"
related:
  - "[[.gitignore]]"
  - "[[Git Stash]]"
  - "[[Git vs GitHub]]"
---

## 1. What
Git là một hệ thống quản lý phiên bản phân tán (Distributed Version Control System - DVCS) giúp theo dõi mọi thay đổi trong mã nguồn trong quá trình phát triển phần mềm. Các câu lệnh Git phổ biến là những công cụ cơ bản để tương tác với kho lưu trữ (repository), quản lý nhánh (branch) và cộng tác với nhóm.

## 2. Why
Trước khi có Git, việc quản lý các phiên bản code thường gặp khó khăn như mất dữ liệu, khó khăn khi gộp (merge) code từ nhiều người hoặc không thể làm việc ngoại tuyến (offline). Git ra đời để giải quyết các vấn đề này, cung cấp khả năng lưu lại các trạng thái của dự án tại bất kỳ thời điểm nào và cho phép nhiều người cùng làm việc trên một codebase mà không ghi đè lên nhau.

## 3. Mental Model
Hãy tưởng tượng Git như một hệ thống "Lưu điểm" (Save points) trong một trò chơi điện tử RPG. 
- `git add` giống như việc bạn chọn những vật phẩm/thành tựu bạn muốn lưu vào lần save tới.
- `git commit` là hành động nhấn nút "Save Game" để tạo ra một điểm hồi phục vĩnh viễn trong lịch sử.
- `git branch` là việc tạo ra một "Dòng thời gian song song" (Parallel timeline) để bạn thử nghiệm các hướng đi mới mà không làm hỏng dòng thời gian chính.

## 4. Where it fits
Quy trình làm việc cơ bản của Git tuân theo sơ đồ sau:
Working Directory (File cục bộ) -> Staging Area (Chờ commit) -> Local Repo (Đã lưu cục bộ) -> Remote Repo (Đã đẩy lên server)

## 5. When to use
- Khi bắt đầu một dự án mới (`git init`).
- Khi muốn lưu lại một tính năng đã hoàn thiện (`git commit`).
- Khi cần thử nghiệm tính năng mới mà không ảnh hưởng code ổn định (`git branch`).
- Khi làm việc nhóm trên các nền tảng như GitHub, GitLab, Bitbucket.

## 6. When NOT to use
- Không dùng Git để quản lý các file binary dung lượng cực lớn (video, database dumps) vì sẽ làm chậm repository (nên dùng Git LFS).
- Không dùng Git để lưu trữ các thông tin nhạy cảm như mật khẩu, API keys, file `.env` (phải dùng `.gitignore`).

## 7. Trade-offs
| Pros | Cons |
|------|------|
| Làm việc offline hoàn toàn | Đường cong học tập (learning curve) khá dốc cho người mới |
| Xử lý nhánh và gộp code cực nhanh | Quản lý file lớn không hiệu quả |
| Tính toàn vẹn dữ liệu cao nhờ hashing | Lịch sử commit có thể bị rối nếu không tuân thủ quy trình |

## 8. Alternatives
| Option | So sánh |
|--------|---------|
| SVN (Subversion) | Quản lý tập trung, dễ học hơn nhưng kém linh hoạt và chậm hơn Git trong việc branching/merging. |
| Mercurial | Tương tự Git nhưng hướng đến sự đơn giản và nhất quán hơn, tuy nhiên cộng đồng nhỏ hơn. |

## 9. How
```bash
# Khởi tạo repository mới
git init

# Sao chép một repo từ remote
git clone <url>

# Kiểm tra trạng thái các file
git status

# Thêm file vào vùng chờ (staging area)
git add <file_name>
git add . # Thêm tất cả thay đổi

# Lưu lại các thay đổi vào lịch sử
git commit -m "Your commit message"

# Quản lý nhánh
git branch # Liệt kê các nhánh
git checkout -b <branch_name> # Tạo và chuyển sang nhánh mới
git switch <branch_name> # Chuyển nhánh (cách mới)

# Cập nhật và đẩy code
git pull origin <branch_name> # Lấy code mới nhất về
git push origin <branch_name> # Đẩy code lên remote

# Xem lịch sử
git log --oneline
```

## 10. Production concerns
### Scaling
Khi dự án lớn dần, cần áp dụng các chiến lược như Gitflow hoặc Trunk-based Development để tránh xung đột.

### Failure
Nếu lỡ commit sai, có thể dùng `git revert` để tạo commit đảo ngược thay vì `git reset` (nếu code đã được push lên remote) để tránh làm hỏng lịch sử của người khác.

### Monitoring
Sử dụng các công cụ như Git Hooks để tự động chạy linter hoặc test trước khi commit/push.

## 11. Common mistakes
- Mistake: Commit trực tiếp các file nhạy cảm (secrets, .env) lên repository.
  Fix: Luôn tạo file `.gitignore` ngay từ đầu và sử dụng công cụ như `git-filter-repo` nếu lỡ commit nhạy cảm.

- Mistake: Sử dụng `git push --force` trên các nhánh chung (như main/develop).
  Fix: Chỉ dùng force push trên nhánh cá nhân sau khi đã rebase, hoặc tốt hơn là dùng `git push --force-with-lease`.

## 12. Sample project
Tạo một repository cá nhân, thực hiện quy trình: Tạo nhánh `feature/login` -> Thêm file `auth.txt` -> Commit -> Merge vào `main` sử dụng `--no-ff` để giữ lại dấu vết của nhánh feature.

## 13. Interview
### Core Q&A
1. Q: Sự khác biệt giữa `git fetch` và `git pull` là gì?
   A: `git fetch` chỉ tải các thay đổi từ remote về local repo nhưng không tự động gộp vào code đang làm việc. `git pull` là sự kết hợp của `git fetch` và `git merge`, nó tải về và gộp luôn vào nhánh hiện tại.

2. Q: `git merge` và `git rebase` khác nhau như thế nào?
   A: `git merge` tạo ra một commit gộp mới và giữ nguyên lịch sử của cả hai nhánh. `git rebase` viết lại lịch sử bằng cách đặt các commit của nhánh hiện tại lên trên đỉnh của nhánh đích, tạo ra một đường thẳng lịch sử sạch sẽ hơn.

### Scenario
Tình huống: Bạn đang thực hiện dở tính năng thì có hotfix khẩn cấp ở nhánh main. Bạn sẽ làm gì để chuyển sang fix lỗi mà không mất code đang viết dở?
Trả lời: Sử dụng `git stash` để tạm lưu các thay đổi chưa commit, sau đó `git checkout main` để fix lỗi. Sau khi xong, quay lại nhánh cũ và dùng `git stash pop` để lấy lại code đang viết.

## 14. References
- Official Docs: https://git-scm.com/doc
- GitHub Repo: https://github.com/git/git
- Spec / RFC: N/A
- Changelog: https://raw.githubusercontent.com/git/git/master/Documentation/RelNotes/2.44.0.txt

## 15. Real-world Code
Hầu hết các dự án open-source lớn như Linux Kernel, Spring Framework, React đều sử dụng Git và có thể tham khảo cách họ đặt tên commit/nhánh.

## 16. Community
- Reddit: r/git
- Stack Overflow: Tag [git]
- Blog: GitHub Blog, GitTower Blog
- Talk: "Git from the Bits Up" - Scott Chacon
