---
created: 2026-05-04
tags:
  - "#type/concept"
  - "#status/draft"
  - "#lang/devops"
  - "#topic/performance"
related:
  - "[[Common Git Commands]]"
  - "[[Branching Strategy in Git]]"
---

## 1. What
**Git LFS (Large File Storage)** là một tiện ích mở rộng mã nguồn mở cho Git, giúp quản lý các tệp tin có kích thước lớn (như video, hình ảnh độ phân giải cao, tệp thực thi, bộ dữ liệu) bằng cách thay thế chúng bằng các tệp con trỏ văn bản (text pointers) bên trong kho lưu trữ Git thực tế.

## 2. Why
Git được thiết kế để quản lý các tệp văn bản và mã nguồn. Khi bạn commit một tệp nhị phân lớn (binary blob), Git sẽ lưu trữ toàn bộ lịch sử của tệp đó. Qua thời gian, việc này làm dung lượng repo tăng vọt, khiến các thao tác `git clone` hoặc `git pull` trở nên cực kỳ chậm chạp vì phải tải về mọi phiên bản cũ của các tệp lớn đó. Git LFS giải quyết vấn đề này bằng cách chỉ tải về phiên bản của tệp lớn mà bạn thực sự cần cho commit hiện tại.

## 3. Mental Model
Hãy tưởng tượng Git LFS như một **"Dịch vụ gửi đồ (Cloakroom)"**. Khi bạn đi dự tiệc (Repo Git), thay vì vác theo một cái vali to nặng (Large File), bạn gửi nó ở quầy giữ đồ. Nhân viên đưa cho bạn một **"Tấm thẻ giữ đồ (Pointer file)"** nhỏ nhẹ. Bạn chỉ cần cầm tấm thẻ này vào bữa tiệc. Khi nào cần lấy vali ra dùng, bạn đưa tấm thẻ cho nhân viên, và họ sẽ mang đúng cái vali đó ra cho bạn.

## 4. Where it fits
Nó hoạt động như một lớp trung gian trong luồng làm việc của Git:
`Working Directory (Large File) -> Git LFS Client -> LFS Storage (Remote)`
`Working Directory (Pointer File) -> Git Index -> Git Repository (Remote)`

## 5. When to use
- Khi project chứa nhiều tài nguyên đồ họa (PSD, AI, PNG/JPG lớn).
- Khi lưu trữ các tệp âm thanh hoặc video.
- Khi làm việc với các model AI hoặc bộ dữ liệu (datasets) lớn.
- Khi cần quản lý các tệp thư viện nhị phân (DLL, SO, JAR) mà không muốn làm nặng repo.

## 6. When NOT to use
- Khi tệp tin chủ yếu là văn bản (mã nguồn, tệp cấu hình).
- Khi project không có các tệp lớn (thường là trên vài MB).
- Khi server hosting Git của bạn không hỗ trợ LFS (mặc dù GitHub, GitLab, Bitbucket đều hỗ trợ tốt).

## 7. Trade-offs
| Pros | Cons |
|------|------|
| Giữ cho dung lượng repo nhỏ gọn, clone/pull cực nhanh. | Yêu cầu cài đặt thêm extension trên máy cá nhân. |
| Chỉ tải xuống phiên bản cần thiết của tệp lớn. | Có thể phát sinh chi phí lưu trữ/băng thông trên cloud (như GitHub LFS). |
| Quá trình làm việc gần như không đổi so với Git thường. | Phức tạp hơn khi cần xóa hoàn toàn lịch sử tệp lớn. |

## 8. Alternatives
| Option | So sánh |
|--------|---------|
| `git-annex` | Mạnh mẽ hơn nhưng cấu hình phức tạp và khó dùng hơn LFS. |
| `DVC (Data Version Control)` | Chuyên dụng cho Machine Learning, hỗ trợ nhiều loại storage (S3, GCS). |
| Lưu trữ Cloud (S3/Drive) | Đơn giản nhưng mất đi sự đồng bộ chặt chẽ với các commit code. |

## 9. How
```bash
# 1. Cài đặt Git LFS (chỉ làm 1 lần trên máy)
git lfs install

# 2. Chọn loại tệp muốn quản lý bằng LFS (ví dụ: tất cả tệp .psd)
git lfs track "*.psd"

# 3. Đảm bảo tệp cấu hình .gitattributes được thêm vào repo
git add .gitattributes

# 4. Commit và Push như bình thường
git add model.psd
git commit -m "Add large design file"
git push origin main

# Kiểm tra trạng thái các tệp LFS
git lfs status
```

## 10. Production concerns
### Scaling
Khi số lượng tệp LFS lên đến hàng chục GB, hãy kiểm tra giới hạn băng thông (bandwidth) của provider. GitHub có giới hạn miễn phí khá thấp cho LFS.

### Failure
Nếu một người trong team chưa cài `git lfs install`, họ sẽ chỉ thấy các tệp pointer (chứa mã băm SHA256) thay vì nội dung file thực tế. Luôn nhắc nhở team chạy lệnh install.

### Monitoring
Sử dụng `git lfs ls-files` để theo dõi danh sách các tệp đang được quản lý bởi LFS và dung lượng của chúng.

## 11. Common mistakes
- **Mistake**: Quên commit tệp `.gitattributes`.
  **Fix**: Luôn kiểm tra `git status` và đảm bảo `.gitattributes` đã được track ngay sau khi chạy lệnh `git lfs track`.

- **Mistake**: Track tệp LFS sau khi đã commit nó vào lịch sử Git thường.
  **Fix**: Phải dùng công cụ như `git lfs migrate` để chuyển đổi lịch sử cũ sang LFS, nếu không repo vẫn sẽ nặng.

## 12. Sample project
Tạo một repo cho dự án Game dùng Unity. Sử dụng Git LFS để quản lý các thư mục `Assets/Textures` và `Assets/Models`.
**Ràng buộc**: Repo chính (mã nguồn) phải được giữ dưới 100MB trong khi tổng dung lượng assets là 2GB.

## 13. Interview
### Core Q&A
1. **Q**: Pointer file trong Git LFS chứa thông tin gì?
   **A**: Nó chứa thông tin metadata bao gồm: phiên bản LFS, mã băm (oid sha256) của tệp gốc và kích thước tệp tính bằng byte.
2. **Q**: Điều gì xảy ra khi bạn chạy `git pull` trong một repo có dùng LFS?
   **A**: Git sẽ tải về các tệp pointer. Sau đó, Git LFS client sẽ tự động dựa vào mã băm trong pointer để tải về tệp thật từ LFS storage và thay thế vào working directory.
3. **Q**: Làm thế nào để chuyển đổi các tệp lớn đã lỡ commit vào Git thường sang LFS?
   **A**: Sử dụng lệnh `git lfs migrate import --include="*.zip"` (ví dụ cho tệp zip). Lưu ý lệnh này sẽ thay đổi lịch sử commit (rewrite history).

### Scenario
**Tình huống**: Bạn clone một repo và thấy tệp video `.mp4` chỉ nặng vài trăm byte và không thể mở được. Nguyên nhân là gì?
**Trả lời**: Do máy tôi chưa cài đặt hoặc chưa kích hoạt Git LFS (`git lfs install`). Thứ tôi đang thấy chỉ là tệp pointer thay vì tệp thật. Tôi cần chạy `git lfs pull` để tải về nội dung thực tế.

## 14. References
- Official Docs: [https://git-lfs.github.com/](https://git-lfs.github.com/)
- GitHub Help: [Configuring Git LFS](https://docs.github.com/en/repositories/working-with-files/managing-large-files/configuring-git-large-file-storage)

## 15. Real-world Code
- Các dự án Game Engine (Unreal, Unity), các dự án Deep Learning (lưu weight của model).

## 16. Community
- Reddit: r/git
- Stack Overflow: Tag [git-lfs]
- Blog: Atlassian Git LFS Tutorial.
