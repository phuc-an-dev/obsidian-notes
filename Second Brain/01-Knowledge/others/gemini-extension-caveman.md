---
created: 2026-05-05
tags:
  - "#type/library"
  - "#status/draft"
  - "#lang/others"
  - "#topic/performance"
related:
  - "[[saas]]"
---

## 1. What
Caveman là một extension dành cho Gemini CLI (phát triển bởi Julius Brussee) cung cấp chế độ giao tiếp siêu nén (ultra-compressed). Nó cắt giảm khoảng 75% lượng token đầu ra bằng cách sử dụng phong cách nói chuyện của "người tiền sử thông minh" nhưng vẫn đảm bảo 100% độ chính xác về kỹ thuật.

## 2. Why
Trước khi có Caveman:
- **Tốn Token**: Các mô hình AI thường có xu hướng giải thích dài dòng, sử dụng nhiều từ đệm (filler words).
- **Độ trễ (Latency)**: Output càng dài, thời gian chờ phản hồi càng lâu.
- **Chi phí**: Đối với người dùng API trả phí, việc dư thừa token gây lãng phí ngân sách.

## 3. Mental Model
Hãy tưởng tượng một **"Kỹ sư Senior cực kỳ bận rộn"**:
- Anh ta biết mọi thứ về hệ thống của bạn.
- Anh ta không quan tâm đến "Xin chào", "Cảm ơn", hay ngữ pháp đúng chuẩn.
- Anh ta chỉ nói những từ khóa quan trọng nhất để bạn hiểu phải làm gì tiếp theo. "Fix bug. Code here. Done."

## 4. Where it fits
Nó hoạt động như một lớp wrapper (extension) nằm trên Gemini CLI, thay đổi cách AI định dạng phản hồi trước khi in ra console.

## 5. When to use
- Khi bạn cần tốc độ phản hồi cực nhanh.
- Khi bạn muốn tiết kiệm token (và tiền) trong các phiên làm việc dài.
- Khi bạn đã hiểu rõ bối cảnh và chỉ cần các lệnh thực thi hoặc xác nhận nhanh.

## 6. When NOT to use
- Khi cần giải thích các khái niệm phức tạp cho người mới bắt đầu.
- Khi cần viết tài liệu chính thức (documentation) hoặc báo cáo cho khách hàng.
- Khi sự tinh tế trong ngôn ngữ tự nhiên là cần thiết để tránh hiểu lầm.

## 7. Trade-offs
| Pros | Cons |
|------|------|
| Tiết kiệm ~75% token output. | Khó đọc đối với người không quen phong cách "caveman". |
| Phản hồi nhanh hơn đáng kể. | Thiếu các sắc thái biểu cảm và lịch sự. |
| Tập trung tối đa vào thông tin kỹ thuật. | Có thể bỏ qua một số giải thích ngữ cảnh hữu ích. |

## 8. Alternatives
| Option | So sánh |
|--------|---------|
| Default Mode | Đầy đủ, lịch sự nhưng tốn kém và chậm hơn. |
| Custom System Prompt | Có thể cấu hình tương tự nhưng mất công duy trì và không có các shortcut command. |

## 9. How
### Cài đặt
```bash
gemini extensions install https://github.com/JuliusBrussee/caveman --consent
```

### Sử dụng
- Kích hoạt: Gõ `/caveman` hoặc nói "caveman mode".
- Các cấp độ (Intensity):
    - `lite`: Nén nhẹ.
    - `full` (mặc định): Nén mạnh.
    - `ultra`: Chỉ dùng từ khóa tối thiểu.
    - `wenyan`: Phong cách Hán văn cổ (Classical Chinese patterns).

## 10. Production concerns
### Scaling
- Rất hữu ích khi chạy các script tự động gọi AI nhiều lần, giúp giảm thiểu đáng kể chi phí vận hành.

### Failure
- Caveman có cơ chế **"Smart Safety"**: Nó sẽ tự động quay về tiếng Anh chuẩn khi phát hiện các cảnh báo bảo mật nghiêm trọng hoặc các hành động có tính phá hủy (destructive actions) để đảm bảo người dùng không hiểu lầm.

### Monitoring
- Có thể theo dõi lượng token tiết kiệm được thông qua dashboard của nhà cung cấp API.

## 11. Common mistakes
- **Mistake**: Quên tắt chế độ Caveman khi cần giải thích chi tiết cho đồng nghiệp.
  **Fix**: Sử dụng lệnh `/caveman off` hoặc restart session.

- **Mistake**: Hiểu lầm các hướng dẫn cực ngắn trong chế độ `ultra`.
  **Fix**: Nếu không chắc chắn, hãy yêu cầu AI giải thích lại ở chế độ `lite` hoặc `default`.

## 12. Sample project
Sử dụng Caveman trong một buổi refactoring mã nguồn kéo dài 2 tiếng với hàng trăm lượt hỏi đáp, giúp giảm chi phí API từ $5 xuống còn $1.2.

## 13. Interview
### Core Q&A
1. Q: Tại sao Caveman lại tiết kiệm được nhiều token như vậy?
   A: Nó loại bỏ hoàn toàn các cấu trúc câu phức tạp, từ nối, từ đệm và các phần chào hỏi/kết thúc không cần thiết, chỉ giữ lại các động từ và danh từ kỹ thuật cốt lõi.

2. Q: Caveman có làm giảm chất lượng code mà AI sinh ra không?
   A: Không. Caveman chỉ thay đổi phần giải thích bằng ngôn ngữ tự nhiên (natural language), phần mã nguồn (code blocks) vẫn được giữ nguyên chất lượng và độ chi tiết.

### Scenario
**Tình huống**: Bạn đang dùng 4G với tốc độ yếu và cần AI fix gấp một đoạn code.
**Giải quyết**: Bật `/caveman ultra`. Lượng dữ liệu tải về ít hơn sẽ giúp bạn nhận được phản hồi nhanh hơn trong điều kiện mạng kém.

## 14. References
- GitHub Repo: [JuliusBrussee/caveman](https://github.com/JuliusBrussee/caveman)
- Gemini CLI Extensions: [Official Documentation](https://github.com/google/gemini-cli)

## 15. Real-world Code
N/A (Prompt-based extension).

## 16. Community
- GitHub Issues của repo `juliusbrussee/caveman`.
- Cộng đồng người dùng Gemini CLI trên Reddit/Discord.
