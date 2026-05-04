---
created: 2026-04-30
tags:
  - "#type/concept"
  - "#status/draft"
  - "#lang/devops"
  - "#topic/version-control"
related:
  - "[[Merge vs Rebase in Git]]"
  - "[[Git Stash]]"
---

## 1. What
`git cherry-pick` là một lệnh mạnh mẽ trong Git cho phép bạn chọn một hoặc nhiều commit cụ thể từ một nhánh khác và áp dụng chúng vào nhánh hiện tại của mình. Thay vì gộp toàn bộ một nhánh (merge) hoặc chuyển gốc toàn bộ nhánh (rebase), bạn chỉ lấy ra những "mẩu" code tinh túy nhất mà bạn cần.

## 2. Why
Đôi khi một nhánh tính năng (feature branch) có chứa nhiều thay đổi chưa sẵn sàng để đưa vào nhánh chính, nhưng lại có một bản sửa lỗi (bug fix) quan trọng hoặc một cải tiến nhỏ mà bạn cần ngay lập tức. Cherry-pick cho phép bạn "cứu" lấy những thay đổi đó mà không phải kéo theo toàn bộ các thay đổi chưa hoàn thiện khác.

## 3. Mental Model
Hãy tưởng tượng bạn đang ở một tiệc buffet (nhánh khác). Thay vì bê nguyên cả cái bàn tiệc về nhà (merge), bạn chỉ dùng kẹp để gắp đúng miếng sushi ngon nhất (commit) và đặt nó vào đĩa của mình (nhánh hiện tại). Bạn có được thứ mình muốn mà đĩa vẫn gọn gàng.

## 4. Where it fits
Nằm trong giai đoạn tích hợp code có chọn lọc (Selective Integration):
Branch A (Commit 1 -> Commit 2 [Bug Fix] -> Commit 3) -> Cherry-pick Commit 2 -> Branch B (Nhận được bản Bug Fix).

## 5. When to use
- **Hotfix**: Sửa lỗi trên nhánh develop và muốn áp dụng ngay vào nhánh main mà không muốn merge toàn bộ code đang develop dở.
- **Undo/Redo**: Lỡ xóa nhầm một commit sau khi rebase/reset, bạn có thể tìm lại mã hash trong `git reflog` và cherry-pick nó lại.
- **Collaborating**: Lấy một thay đổi cụ thể từ nhánh của đồng nghiệp mà không muốn nhận toàn bộ logic của họ.

## 6. When NOT to use
- **Duplication**: Không dùng cherry-pick thay cho merge thông thường vì nó tạo ra các commit trùng lặp (khác mã hash nhưng cùng nội dung), làm lịch sử git bị rối và gây khó khăn khi merge thật sự sau này.
- **Large batches**: Nếu bạn cần cherry-pick hàng chục commit, hãy xem xét dùng merge hoặc rebase sẽ hiệu quả và an toàn hơn.

## 7. Trade-offs
| Pros | Cons |
|------|------|
| Rất linh hoạt, chọn lọc cao | Tạo ra các commit bản sao (Duplicate commits) |
| Giải quyết nhanh các tình huống khẩn cấp | Có thể gây xung đột logic nếu commit phụ thuộc vào code trước đó |
| Giữ nhánh chính sạch sẽ | Làm mất đi mối quan hệ nguồn gốc giữa các nhánh |

## 8. Alternatives
| Option | So sánh |
|--------|---------|
| Merge | Gộp toàn bộ lịch sử, an toàn và rõ ràng hơn cho các tính năng lớn. |
| Rebase | Di chuyển toàn bộ nhánh, giữ lịch sử tuyến tính nhưng viết lại quá khứ. |
| Patch | Tạo file `.patch` và áp dụng thủ công (cách cổ điển hơn). |

## 9. How
```bash
# Lấy một commit cụ thể
git cherry-pick <commit-hash>

# Lấy nhiều commit cùng lúc
git cherry-pick <hash1> <hash2> <hash3>

# Lấy một khoảng commit (từ hash1 đến hash2, không bao gồm hash1)
git cherry-pick <hash1>..<hash2>

# Nếu có conflict
# 1. Sửa code bị conflict
# 2. git add .
# 3. git cherry-pick --continue

# Nếu muốn hủy bỏ
git cherry-pick --abort
```

## 10. Production concerns
### Scaling
Trong các hệ thống lớn, cherry-pick thường được dùng để "backport" các bản sửa lỗi từ phiên bản mới về các phiên bản cũ hơn (LTS versions) vẫn đang được hỗ trợ.

### Failure
Xung đột (Conflict) khi cherry-pick thường xảy ra nếu commit bạn chọn dựa trên một đoạn code không tồn tại ở nhánh hiện tại. Luôn kiểm tra kỹ code sau khi cherry-pick thành công.

### Monitoring
Dùng `git log --cherry-mark` để tìm các commit "tương đương" giữa các nhánh (những commit có nội dung giống nhau nhưng khác mã hash do cherry-pick).

## 11. Common mistakes
- Mistake: Cherry-pick một commit mà không biết nó phụ thuộc vào các commit trước đó.
  Fix: Luôn kiểm tra dependencies của commit đó hoặc cherry-pick cả chuỗi commit liên quan.

- Mistake: Quên không commit sau khi giải quyết conflict khi cherry-pick.
  Fix: Phải dùng `git cherry-pick --continue` để hoàn tất quá trình.

## 12. Sample project
Tạo nhánh `feature-A` với 3 commits. Sang nhánh `main`, thực hiện cherry-pick commit thứ 2 của `feature-A`. Kiểm tra `git log` để thấy commit mới ở `main` có nội dung giống hệt nhưng mã hash đã thay đổi.

## 13. Interview
### Core Q&A
1. Q: Điều gì xảy ra với mã hash của commit sau khi cherry-pick?
   A: Mã hash sẽ thay đổi. Mặc dù nội dung code, tác giả và message có thể giữ nguyên, nhưng vì nó được đặt vào một vị trí mới trong cây lịch sử (có parent khác), Git sẽ tính toán lại mã hash mới.

2. Q: Làm thế nào để cherry-pick một commit mà không tạo commit ngay lập tức (để sửa thêm)?
   A: Sử dụng flag `-n` hoặc `--no-commit`. Lệnh sẽ là `git cherry-pick -n <hash>`. Các thay đổi sẽ nằm ở vùng staging để bạn chỉnh sửa trước khi commit thủ công.

### Scenario
Tình huống: Bạn lỡ commit nhầm một tính năng quan trọng vào nhánh `wrong-branch`. Bạn muốn chuyển nó sang `right-branch` và xóa ở nhánh cũ. Bạn làm thế nào?
Trả lời: 
1. Sang nhánh `right-branch`: `git checkout right-branch`.
2. Cherry-pick commit đó: `git cherry-pick <hash>`.
3. Quay lại `wrong-branch`: `git checkout wrong-branch`.
4. Xóa commit nhầm: `git reset --hard HEAD~1` (nếu là commit cuối cùng).

## 14. References
- Git SCM - Cherry-pick: https://git-scm.com/docs/git-cherry-pick
- Atlassian Tutorial: https://www.atlassian.com/git/tutorials/cherry-pick

## 15. Real-world Code
Các dự án như Linux Kernel sử dụng cherry-pick cực kỳ nhiều để đưa các bản vá (patches) từ nhánh phụ vào nhánh chính thức của Linus Torvalds.

## 16. Community
- Stack Overflow: Tag [git-cherry-pick]
- Reddit: r/git (thảo luận về việc lạm dụng cherry-pick làm nát lịch sử repo).
