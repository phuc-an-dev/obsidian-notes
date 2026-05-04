---
created: 2026-04-30
tags:
  - "#type/concept"
  - "#status/draft"
  - "#lang/devops"
  - "#topic/version-control"
related:
  - "[[Common Git Commands]]"
  - "[[Team Collaboration Workflow in Git]]"
  - "[[Cherry-pick in Git]]"
---

## 1. What
`git stash` là một lệnh trong Git cho phép bạn tạm thời lưu trữ (shelve) các thay đổi đang làm dở (đã add hoặc chưa add vào staging) để quay về trạng thái thư mục làm việc sạch sẽ (clean working directory). Sau đó, bạn có thể quay lại và lấy ra các thay đổi này để tiếp tục làm việc bất cứ khi nào bạn muốn.

## 2. Why
Trong quá trình phát triển, bạn thường gặp tình huống đang code dở một tính năng nhưng lại có một yêu cầu khẩn cấp (fix bug, review code nhánh khác). Git không cho phép bạn chuyển nhánh nếu có các thay đổi chưa commit mà gây xung đột. `git stash` giúp bạn "cất tạm" code dở mà không cần phải tạo các commit rác (wip commits).

## 3. Mental Model
Hãy tưởng tượng bạn đang nấu ăn (viết code). Đột nhiên có khách đến chơi và bạn cần dọn dẹp bàn bếp ngay lập tức. Thay vì vứt đồ ăn đi hoặc nấu vội cho xong, bạn cho tất cả nguyên liệu đang thái dở vào một cái hộp (stash) và cất vào tủ lạnh. Khi khách về, bạn lấy cái hộp đó ra và tiếp tục nấu nướng đúng tại công đoạn đang dở dang.

## 4. Where it fits
Nằm trong giai đoạn quản lý trạng thái tạm thời (Temporary State Management):
Working Directory (Dirty) -> Git Stash -> Working Directory (Clean) -> Switch Branch -> Work -> Switch Back -> Git Stash Pop -> Working Directory (Dirty).

## 5. When to use
- Khi cần chuyển nhánh gấp mà code hiện tại chưa đủ hoàn thiện để commit.
- Khi muốn lưu tạm các cấu hình thử nghiệm cục bộ mà không muốn đẩy lên server.
- Khi muốn "làm sạch" thư mục để pull code mới về mà không muốn merge ngay.

## 6. When NOT to use
- Không dùng stash thay cho commit lâu dài. Stash chỉ nên dùng cho các thay đổi ngắn hạn.
- Đừng lạm dụng stash quá nhiều mà không dọn dẹp (clear), bạn sẽ dễ bị quên nội dung trong các bản stash cũ.

## 7. Trade-offs
| Pros | Cons |
|------|------|
| Chuyển nhánh cực nhanh và an toàn | Dễ bị quên hoặc lạc mất code nếu có quá nhiều stash |
| Không làm bẩn lịch sử commit bằng các bản nháp | Conflict có thể xảy ra khi lấy code từ stash ra (pop/apply) |
| Có thể áp dụng stash vào bất kỳ nhánh nào | Không tự động lưu các file mới (untracked) trừ khi dùng flag |

## 8. Alternatives
| Option | So sánh |
|--------|---------|
| WIP Commit | Commit tạm với message "wip". Cách này an toàn hơn nhưng làm bẩn lịch sử (cần squash sau đó). |
| Git Worktree | Cho phép làm việc trên nhiều nhánh cùng lúc trong các thư mục khác nhau. Phù hợp cho các task song song dài hạn. |

## 9. How
```bash
# Lưu code đang dở (mặc định không lưu file untracked)
git stash

# Lưu kèm theo lời nhắn để dễ nhớ
git stash save "Đang làm dở tính năng login"
# Hoặc cách mới hơn:
git stash push -m "Đang làm dở tính năng login"

# Lưu cả file untracked (file mới tạo chưa add)
git stash -u

# Xem danh sách các bản stash
git stash list

# Lấy bản stash mới nhất ra và XÓA khỏi danh sách
git stash pop

# Lấy bản stash ra nhưng VẪN GIỮ trong danh sách
git stash apply stash@{0}

# Xóa một bản stash cụ thể
git stash drop stash@{0}

# Xóa toàn bộ stash
git stash clear
```

## 10. Production concerns
### Scaling
Trong các dự án lớn, việc sử dụng `git stash save` với message rõ ràng là bắt buộc để tránh nhầm lẫn khi bạn phải xử lý nhiều context-switching liên tục.

### Failure
Nếu khi `git stash pop` mà gặp conflict, Git sẽ giữ lại bản stash đó trong danh sách cho đến khi bạn giải quyết xong và xóa thủ công (drop). Điều này đảm bảo bạn không bị mất code nếu gộp lỗi.

### Monitoring
Kiểm tra nội dung của một bản stash trước khi lấy ra bằng: `git stash show -p stash@{0}`.

## 11. Common mistakes
- Mistake: Stash xong nhưng khi pop ra không thấy file mới đâu.
  Fix: Do file mới (untracked) không được stash mặc định. Phải dùng `git stash -u`.

- Mistake: Để danh sách stash quá dài và không biết cái nào là cái nào.
  Fix: Luôn dùng `-m` để đặt tên cho stash và thường xuyên `git stash clear`.

## 12. Sample project
Tạo thay đổi ở file A. Chạy `git stash`. Tạo thay đổi khác ở file A và commit. Chạy `git stash pop` để thấy Git cố gắng gộp các thay đổi từ stash vào bản hiện tại và xử lý conflict nếu có.

## 13. Interview
### Core Q&A
1. Q: Sự khác biệt giữa `git stash pop` và `git stash apply` là gì?
   A: `pop` lấy bản stash ra và xóa nó khỏi danh sách stash. `apply` lấy bản stash ra nhưng vẫn giữ nó lại trong danh sách. Nên dùng `apply` nếu bạn muốn áp dụng cùng một thay đổi cho nhiều nhánh khác nhau.

2. Q: Làm thế nào để stash chỉ các file đã được add vào staging (index)?
   A: Sử dụng lệnh `git stash --keep-index`. Các thay đổi đã staged sẽ vẫn được giữ lại trong working directory, trong khi mọi thứ khác được đưa vào stash.

### Scenario
Tình huống: Bạn lỡ `git stash clear` và mất hết code quan trọng chưa commit. Có cách nào cứu không?
Trả lời: Có thể cứu được bằng cách dùng `git fsck --unreachable` để tìm các mã hash của các "dangling commit" (các commit không có nhánh nào trỏ tới), sau đó dùng `git show <hash>` để tìm lại code và `git cherry-pick` hoặc `git merge` nó lại.

## 14. References
- Git Stash Documentation: https://git-scm.com/docs/git-stash
- Atlassian Git Stash Tutorial: https://www.atlassian.com/git/tutorials/saving-changes/git-stash

## 15. Real-world Code
Thường xuyên xuất hiện trong quy trình của các Senior Dev khi cần hotfix khẩn cấp mà không muốn tạo nhánh mới hay commit dở.

## 16. Community
- Reddit: r/git (nhiều bài viết về mẹo dùng stash hiệu quả).
- Stack Overflow: Tag [git-stash].
