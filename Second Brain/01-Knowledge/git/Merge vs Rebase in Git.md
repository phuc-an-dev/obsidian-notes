---
created: 2026-04-30
tags:
  - "#type/concept"
  - "#status/draft"
  - "#lang/devops"
  - "#topic/version-control"
related:
  - "[[Branching Strategy in Git]]"
  - "[[Squash Commit]]"
  - "[[Cherry-pick in Git]]"
---

## 1. What
Merge và Rebase là hai phương thức chính trong Git dùng để tích hợp các thay đổi từ nhánh này sang nhánh khác. Trong khi Merge tạo ra một commit gộp để kết nối lịch sử của hai nhánh, Rebase di chuyển toàn bộ các commit của một nhánh lên trên đỉnh của một nhánh khác, tạo ra một đường thẳng lịch sử.

## 2. Why
Trong quá trình làm việc nhóm, các nhánh sẽ thường xuyên bị lệch pha với nhánh chính (main/develop). Chúng ta cần các công cụ này để cập nhật code mới nhất từ nhánh chính vào nhánh đang phát triển (feature branch) hoặc để gộp code tính năng đã hoàn thành vào dòng code chính thức.

## 3. Mental Model
- **Merge (Hợp nhất)**: Hãy tưởng tượng hai dòng sông đổ vào một hồ chung. Hồ chính là "Merge Commit", nó ghi lại thời điểm hai dòng chảy gặp nhau. Lịch sử của cả hai dòng sông đều được giữ nguyên vẹn.
- **Rebase (Xây lại nền)**: Hãy tưởng tượng bạn đang xây một ngôi nhà (nhánh feature) trên một mảnh đất cũ. Khi mảnh đất đó được nâng cấp (nhánh main có code mới), bạn nhấc toàn bộ ngôi nhà của mình lên và đặt nó lên trên nền đất đã được nâng cấp mới nhất. Lịch sử trông như thể bạn vừa mới bắt đầu xây nhà trên nền đất mới đó.

## 4. Where it fits
Nằm trong giai đoạn tích hợp code (Integration phase) của quy trình phát triển:
Local Changes -> Update from Main (Merge/Rebase) -> Resolve Conflicts -> Push/Pull Request.

## 5. When to use
- **Dùng Merge**: Khi bạn muốn gộp một nhánh tính năng đã hoàn thiện vào nhánh chính (`main`/`develop`) và muốn giữ lại bằng chứng lịch sử rõ ràng về việc nhánh đó đã tồn tại.
- **Dùng Rebase**: Khi bạn muốn cập nhật code mới nhất từ nhánh chính vào nhánh feature cá nhân của mình để tránh các commit gộp rác (merge commits) làm rối lịch sử.

## 6. When NOT to use
- **Tuyệt đối không Rebase trên các nhánh công khai (Public Branches)**: Nếu bạn rebase một nhánh mà người khác cũng đang làm việc trên đó (như `main`), bạn sẽ làm đảo lộn lịch sử của họ và gây ra thảm họa gộp mã.
- **Không dùng Merge**: Khi bạn thực hiện các thay đổi nhỏ và muốn lịch sử commit của mình trông sạch sẽ, tuyến tính.

## 7. Trade-offs
| Tiêu chí | Merge | Rebase |
|----------|-------|--------|
| **Lịch sử** | Không thay đổi (Non-destructive), giữ nguyên vết tích. | Bị viết lại (Rewrites history), tạo ra đường thẳng đẹp. |
| **Độ sạch** | Có thể bị rối bởi nhiều "Merge commit". | Rất sạch sẽ, dễ theo dõi flow logic. |
| **Xử lý xung đột** | Xử lý tất cả trong một lần gộp. | Phải xử lý xung đột qua từng commit một. |
| **Mức độ an toàn** | Cao, dễ dàng quay lại (undo). | Thấp hơn, cần cẩn trọng vì thay đổi mã băm (hash) của commit. |

## 8. Alternatives
| Option | So sánh |
|--------|---------|
| Squash Merge | Kết hợp cả hai: gộp tất cả commit của nhánh feature thành một commit duy nhất rồi mới merge vào main. Giữ lịch sử main sạch nhưng mất chi tiết của nhánh feature. |
| Cherry-pick | Chỉ lấy một vài commit cụ thể thay vì lấy toàn bộ nhánh. |

## 9. How
### Cách thực hiện Merge
```bash
git checkout main
git merge feature-branch
```

### Cách thực hiện Rebase
```bash
git checkout feature-branch
git rebase main

# Nếu có conflict, sửa xong thì:
git add .
git rebase --continue
```

## 10. Production concerns
### Scaling
Trong các dự án lớn, việc sử dụng Rebase cho các nhánh cá nhân giúp các cấp quản lý (Maintainers) dễ dàng review code và hiểu được thứ tự logic của các thay đổi.

### Failure
Nếu Rebase bị lỗi nặng, bạn luôn có thể dùng `git rebase --abort` để quay lại trạng thái trước khi bắt đầu.

### Monitoring
Kiểm tra lịch sử bằng `git log --graph --oneline --all` để thấy rõ sự khác biệt giữa cấu trúc Merge (nhánh rẽ) và Rebase (đường thẳng).

## 11. Common mistakes
- Mistake: Rebase nhánh `main` lên nhánh `feature`.
  Fix: Luôn làm ngược lại, rebase nhánh `feature` lên trên `main`.

- Mistake: Quên không `git push --force` sau khi rebase nhánh cá nhân đã được push lên remote trước đó.
  Fix: Sau khi rebase, lịch sử local và remote sẽ khác nhau, bạn cần dùng `git push --force-with-lease` để cập nhật remote.

## 12. Sample project
Tạo 2 nhánh cùng sửa một file. Nhánh 1 dùng Merge để cập nhật, Nhánh 2 dùng Rebase. So sánh kết quả của `git log` để thấy sự khác biệt về cấu trúc cây lịch sử.

## 13. Interview
### Core Q&A
1. Q: Tại sao Rebase được coi là nguy hiểm?
   A: Vì Rebase viết lại lịch sử bằng cách tạo ra các commit mới có mã hash khác với ban đầu. Nếu các commit cũ đã được chia sẻ với người khác, việc thay đổi này sẽ khiến repository của họ không còn khớp với server.

2. Q: Khi nào bạn sẽ chọn Squash Merge thay vì Merge thông thường?
   A: Khi nhánh feature có quá nhiều commit nhỏ, vụn vặt (ví dụ: "fix typo", "update readme") mà tôi không muốn chúng xuất hiện trong lịch sử của nhánh chính.

### Scenario
Tình huống: Bạn đang rebase và gặp quá nhiều conflict ở mỗi commit. Bạn nên làm gì?
Trả lời: Nếu số lượng commit quá lớn và conflict quá phức tạp, tôi sẽ `git rebase --abort` và chọn phương án `git merge` hoặc `git merge --squash` để giải quyết conflict trong một lần duy nhất.

## 14. References
- Git SCM - Merging: https://git-scm.com/book/en/v2/Git-Branching-Basic-Branching-and-Merging
- Git SCM - Rebasing: https://git-scm.com/book/en/v2/Git-Branching-Rebasing
- Atlassian - Merging vs Rebasing: https://www.atlassian.com/git/tutorials/merging-vs-rebasing

## 15. Real-world Code
Nhiều dự án mã nguồn mở yêu cầu cộng tác viên (contributors) phải rebase nhánh của họ lên bản mới nhất của dự án gốc trước khi gửi Pull Request để đảm bảo việc merge diễn ra mượt mà (Fast-forward).

## 16. Community
- Reddit: r/git (nơi thường xuyên diễn ra tranh luận về Merge vs Rebase)
- Stack Overflow: Tag [git-rebase], [git-merge]
- Blog: "Golden Rule of Rebasing" bài viết kinh điển về việc không rebase nhánh public.
