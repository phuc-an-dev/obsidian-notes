---
created: 2026-05-05
tags:
  - "#type/concept"
  - "#status/draft"
  - "#lang/others"
  - "#topic/error-handling"
related:
  - "[[github-secrets]]"
---

## 1. What
Git History là lịch sử toàn bộ các thay đổi được lưu trữ trong thư mục `.git`. Rủi ro lộ lọt thông tin xảy ra khi các dữ liệu nhạy cảm (secrets, keys, passwords) vô tình bị commit vào repo. Ngay cả khi bạn đã xóa dữ liệu đó ở commit hiện tại, nó vẫn tồn tại vĩnh viễn trong các commit cũ của lịch sử Git.

## 2. Why
Git được thiết kế để không bao giờ quên. Việc chỉ xóa một file nhạy cảm và commit mới (`git rm`) không giúp loại bỏ file đó khỏi cơ sở dữ liệu của Git. Kẻ tấn công có thể dễ dàng duyệt lại các commit cũ để tìm kiếm các keys bị bỏ quên, dẫn đến thảm họa bảo mật.

## 3. Mental Model
Hãy tưởng tượng Git History như một cuốn sổ ghi chép bằng bút bi không thể xóa. Nếu bạn lỡ viết mật khẩu vào trang 5, rồi trang 6 bạn lấy bút xóa gạch đi, thì người ta vẫn có thể lật lại trang 5 để xem mật khẩu đó là gì. Để xóa thực sự, bạn phải xé trang đó đi và viết lại toàn bộ cuốn sổ từ đầu (Rewrite history).

## 4. Where it fits
Local Repo (.git folder) -> Commits -> Push to Remote (GitHub/GitLab) -> Potential Leak.
Các công cụ xử lý can thiệp vào tầng Object Database của Git.

## 5. When to use
- Khi phát hiện đã lỡ commit file `.env` hoặc API key vào repository.
- Trước khi chuyển một repository từ Private sang Public.
- Khi cần dọn dẹp các file rác dung lượng lớn để giảm kích thước repo.
- Khi muốn ẩn danh hóa lịch sử commit (thay đổi email/tên tác giả cũ).

## 6. When NOT to use
- Không dùng nếu repository đó đang có nhiều người cùng làm việc và bạn không thể yêu cầu họ xóa bản local và clone lại (do rewrite history sẽ gây xung đột cực lớn).
- Không dùng nếu bạn không có quyền Admin để force push lên nhánh chính.

## 7. Trade-offs
| Pros | Cons |
|------|------|
| Loại bỏ hoàn toàn dấu vết dữ liệu nhạy cảm | Làm thay đổi mã Hash (SHA) của tất cả commit liên quan |
| Giảm dung lượng repository | Phá vỡ các Pull Requests và nhánh đang dang dở |
| Bảo vệ an toàn tuyệt đối cho dự án | Đòi hỏi tất cả thành viên team phải thực hiện thao tác khôi phục phức tạp |

## 8. Alternatives
| Option | So sánh |
|--------|---------|
| BFG Repo-Cleaner | Nhanh hơn gấp 10-100 lần so với git filter-branch, dễ sử dụng hơn |
| git filter-repo | Công cụ hiện đại được Git khuyên dùng thay thế cho filter-branch |
| git filter-branch | Công cụ cũ, chậm và dễ gây lỗi dữ liệu (không khuyến khích dùng nữa) |

## 9. How
```bash
# Cách 1: Sử dụng BFG Repo-Cleaner để xóa một file cụ thể
# Tải bfg.jar về máy trước
java -jar bfg.jar --delete-files .env

# Cách 2: Sử dụng git filter-repo (cần cài đặt trước)
git filter-repo --path .env --invert-paths

# Sau khi xử lý xong, phải dọn rác và force push
git reflog expire --expire=now --all && git gc --prune=now --aggressive
git push origin --force --all
```

## 10. Production concerns
### Scaling
Với các repo có hàng chục ngàn commit và dung lượng hàng GB, `git filter-branch` sẽ treo máy. Phải dùng `BFG` hoặc `git filter-repo` để xử lý trong vài giây/phút.

### Failure
Nếu force push thất bại do nhánh bị bảo vệ (protected branch), phải tạm thời tắt bảo vệ trong settings của GitHub/GitLab trước khi thực hiện.

### Monitoring
Sử dụng các công cụ như `trufflehog` hoặc `gitleaks` chạy định kỳ trên server để phát hiện sớm các secret mới bị leak vào lịch sử.

## 11. Common mistakes
- Mistake: Chỉ dùng `git rm --cached` và nghĩ rằng đã an toàn.
  Fix: File vẫn nằm trong lịch sử, phải dùng công cụ rewrite history.

- Mistake: Force push lên nhánh main mà không thông báo cho team.
  Fix: Luôn thông báo "Freeze code" cho toàn team trước khi dọn dẹp lịch sử để họ backup code local.

## 12. Sample project
Tạo một repo giả, commit một file `passwords.txt`. Sau đó dùng BFG để xóa file đó khỏi toàn bộ lịch sử. Sử dụng `git log --all --full-history -- **/passwords.txt` để kiểm tra lại xem file đã thực sự biến mất chưa.

## 13. Interview
### Core Q&A
1. Q: Tại sao lệnh `git rm` không đủ để xóa thông tin nhạy cảm?
   A: Vì Git lưu trữ mọi trạng thái của dự án theo thời gian. `git rm` chỉ tạo ra một commit mới không có file đó, nhưng file đó vẫn nằm trong các snapshot của các commit trước đó.
2. Q: "Rewrite history" trong Git có nguy hiểm gì?
   A: Nó làm thay đổi ID (SHA) của các commit. Nếu người khác đã pull code cũ về, khi họ pull lại sẽ bị xung đột nghiêm trọng vì lịch sử local của họ không còn khớp với server nữa.

### Scenario
Bạn vừa vô tình push một AWS Access Key lên một repo public. Bạn đã xóa file và push commit mới ngay lập tức. Liệu Key đó có còn bị lộ không và bạn phải làm gì?
Trả lời: Key vẫn bị lộ hoàn toàn. 1. Việc đầu tiên và quan trọng nhất là lên AWS Console để Deactivate/Delete Key đó ngay lập tức (Xử lý gốc). 2. Sau đó mới dùng BFG hoặc git filter-repo để dọn dẹp lịch sử repo. 3. Đổi mật khẩu/key của các dịch vụ liên quan nếu có nghi ngờ.

## 14. References
- Official Docs: https://git-scm.com/book/en/v2/Git-Tools-Rewriting-History
- GitHub Repo: https://github.com/rtyley/bfg-repo-cleaner
- Spec / RFC: N/A
- Changelog: Git 2.22 officially recommends git-filter-repo over filter-branch.

## 15. Real-world Code
https://github.com/newren/git-filter-repo (Mã nguồn công cụ lọc history mạnh mẽ nhất hiện nay)

## 16. Community
- Reddit: r/git
- Stack Overflow: Tag #git #git-rewrite-history
- Blog: "Cleaning up a messed-up Git repository" by Atlassian.
- Talk: "Git Internals" by Scott Chacon.
