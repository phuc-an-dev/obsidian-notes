---
created: 2026-04-30
tags:
  - "#type/concept"
  - "#status/draft"
  - "#lang/devops"
  - "#topic/version-control"
related:
  - "[[Merge vs Rebase in Git]]"
  - "[[Code Review and Merge Process]]"
---

## 1. What
Squash commit là kỹ thuật gộp nhiều commit nhỏ, vụn vặt thành một commit duy nhất có ý nghĩa hơn. Quá trình này thường được thực hiện trước khi gộp (merge) một nhánh tính năng (feature branch) vào nhánh chính (main/develop) để giữ cho lịch sử dự án gọn gàng và dễ theo dõi.

## 2. Why
Trong quá trình phát triển, lập trình viên thường tạo ra rất nhiều commit trung gian như "fix typo", "update readme", "wip", hoặc các commit thử nghiệm. Nếu đưa tất cả các commit này vào lịch sử chính, nó sẽ trở nên cực kỳ rối rắm, gây khó khăn cho việc tra cứu lỗi (debug) hoặc hiểu về quá trình tiến hóa của tính năng đó trong tương lai.

## 3. Mental Model
Hãy tưởng tượng bạn đang viết một cuốn sách. Mỗi ngày bạn viết một vài đoạn, xóa đi sửa lại nhiều lần. Những bản nháp nham nhở đó là các "vụn" commit. Khi xuất bản cuốn sách, bạn gộp tất cả các lần viết lách đó lại thành một chương hoàn chỉnh và sạch sẽ. Độc giả (nhánh chính) chỉ cần đọc chương đó thay vì xem toàn bộ quá trình bạn tẩy xóa.

## 4. Where it fits
Nằm ở bước cuối cùng trước khi hoàn tất một nhiệm vụ:
Feature Development -> Multiple Small Commits -> Squash Commits -> Pull Request / Merge to Main.

## 5. When to use
- Trước khi gửi Pull Request để Reviewer dễ dàng kiểm soát thay đổi.
- Khi muốn "làm sạch" các commit rác sau khi đã hoàn thành một tính năng.
- Khi sử dụng quy trình Trunk-based Development để giữ nhánh chính luôn tuyến tính và sạch.

## 6. When NOT to use
- Không squash các commit đã được push lên một nhánh chung mà người khác đang cùng làm việc (vì nó thay đổi mã hash).
- Đừng gộp các thay đổi không liên quan logic vào một commit duy nhất (làm mất tính nguyên tử - atomic - của commit).

## 7. Trade-offs
| Pros | Cons |
|------|------|
| Lịch sử commit cực kỳ sạch sẽ và dễ hiểu | Mất đi chi tiết lịch sử của từng bước nhỏ |
| Giúp lệnh `git bisect` hoạt động hiệu quả hơn | Khó khăn nếu muốn quay lại (revert) chỉ một phần nhỏ của tính năng |
| Tập trung vào "tại sao" và "cái gì" thay vì "khi nào" | Đòi hỏi kỹ năng sử dụng interactive rebase |

## 8. Alternatives
| Option | So sánh |
|--------|---------|
| Standard Merge | Giữ nguyên mọi commit vụn vặt, lịch sử thực tế nhưng rối. |
| Rebase | Giữ lịch sử tuyến tính nhưng vẫn giữ nguyên số lượng commit (nếu không dùng flag squash). |
| Git Amend | Chỉ dùng để gộp thay đổi mới nhất vào commit ngay trước đó. |

## 9. How
### Cách 1: Sử dụng Interactive Rebase (Phổ biến nhất)
```bash
# Gộp 3 commit gần nhất
git rebase -i HEAD~3

# Một trình soạn thảo văn bản sẽ hiện ra. 
# Thay chữ "pick" bằng "squash" (hoặc "s") cho các commit muốn gộp vào commit đầu tiên.
# pick 1a2b3c4 Commit 1
# squash 5d6e7f8 Commit 2
# squash 9g1h2i3 Commit 3

# Sau đó lưu lại, Git sẽ yêu cầu bạn soạn thảo lại message cho commit gộp cuối cùng.
```

### Cách 2: Sử dụng Merge Squash
```bash
git checkout main
git merge --squash feature-branch
git commit -m "Tính năng hoàn chỉnh sau khi squash"
```

## 10. Production concerns
### Scaling
Trong các hệ thống lớn, việc squash commit giúp đội ngũ vận hành dễ dàng xác định commit nào gây ra lỗi để thực hiện rollback nhanh chóng.

### Failure
Nếu xảy ra xung đột (conflict) trong quá trình rebase để squash, bạn phải giải quyết từng bước một. Nếu quá phức tạp, có thể dùng `git rebase --abort`.

### Monitoring
Dùng `git log --oneline` để kiểm chứng kết quả sau khi squash so với trước đó.

## 11. Common mistakes
- Mistake: Squash commit của người khác trên nhánh chung.
  Fix: Chỉ squash code của chính mình trên nhánh feature cá nhân.

- Mistake: Xóa mất message quan trọng khi gộp commit.
  Fix: Hãy tóm tắt lại toàn bộ các thay đổi quan trọng vào message của commit gộp.

## 12. Sample project
Tạo 3 commit: "Add UI", "Fix UI", "Update UI style". Thực hiện `git rebase -i` để gộp thành "Implement complete UI for Login feature".

## 13. Interview
### Core Q&A
1. Q: Tại sao Squash commit lại giúp ích cho việc tìm lỗi (debugging)?
   A: Vì khi dùng `git bisect` để tìm commit gây lỗi, nếu lịch sử có quá nhiều commit "rác" không chạy được hoặc chỉ thay đổi nhỏ, quá trình tìm kiếm sẽ rất mất thời gian và có thể cho kết quả sai. Một commit squash đảm bảo mỗi điểm dừng trong lịch sử đều là một trạng thái chạy được.

2. Q: Sự khác biệt giữa `pick`, `squash` và `fixup` trong interactive rebase là gì?
   A: `pick` là giữ nguyên commit. `squash` là gộp commit đó vào commit phía trước và cho phép chỉnh sửa message. `fixup` cũng gộp vào commit trước nhưng bỏ qua message của nó (chỉ giữ lại message của commit đầu tiên).

### Scenario
Tình huống: Reviewer yêu cầu bạn gộp 10 commits trong PR thành 1 commit duy nhất trước khi họ merge. Bạn làm thế nào?
Trả lời: Tôi sẽ sử dụng `git rebase -i HEAD~10`, giữ commit đầu tiên là `pick` và 9 commit còn lại là `fixup` để gộp nhanh. Sau đó tôi sẽ `git push --force-with-lease` lên remote để cập nhật PR.

## 14. References
- Git Rebase Interactive: https://git-scm.com/docs/git-rebase#_interactive_mode
- GitHub Squash and Merge: https://docs.github.com/en/pull-requests/collaborating-with-pull-requests/incorporating-changes-from-a-pull-request/about-pull-request-merges#squash-and-merge-your-pull-request-commits

## 15. Real-world Code
Nhiều công ty công nghệ lớn (như Google) yêu cầu quy trình gộp commit nghiêm ngặt để đảm bảo mỗi commit trên nhánh chính đều tương ứng với một tính năng hoàn thiện.

## 16. Community
- Blog: "Commit Often, Perfect Later, Publish Once" - Châm ngôn về squash commit.
- Reddit: r/git (nơi tranh luận về việc có nên giữ chi tiết lịch sử hay không).
