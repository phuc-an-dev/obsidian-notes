---
created: 2026-04-22
tags:
  - "#type/library"
  - "#status/draft"
  - "#lang/javascript"
  - "#topic/automation"
related: "[[javascript]]"
---

## 1. What
Google Apps Script (GAS) là một nền tảng phát triển ứng dụng nhanh (Rapid Application Development) dựa trên đám mây của Google. Nó sử dụng ngôn ngữ lập trình JavaScript (với bộ engine V8 hiện đại) để tự động hóa các tác vụ trên Google Workspace (Sheets, Docs, Drive, Gmail, v.v.) và kết nối với các dịch vụ bên ngoài.


## 2. Why
Trước khi có GAS, việc tự động hóa các sản phẩm của Google yêu cầu phải sử dụng các API phức tạp, quản lý server riêng và xử lý xác thực OAuth phiền toái. GAS ra đời để cung cấp một môi trường Serverless hoàn toàn miễn phí, nơi code có thể truy cập trực tiếp vào dữ liệu người dùng trong hệ sinh thái Google mà không cần thiết lập hạ tầng.


## 3. Mental Model
Hãy coi GAS như một "Quản gia" (Butler) thông minh trong ngôi nhà Google Workspace của bạn. Bạn không cần phải tự mình copy dữ liệu từ Sheets sang Docs hay gửi email thông báo hàng ngày; bạn chỉ cần viết một bản hướng dẫn (Script) và Quản gia này sẽ thực hiện nó chính xác vào đúng thời điểm (Triggers) hoặc khi có sự kiện xảy ra.


## 4. Where it fits
Google Workspace -> Apps Script Engine (V8) -> Google Services APIs (GmailApp, SpreadsheetApp, DriveApp) -> External APIs (UrlFetchApp).


## 5. When to use
- Tự động hóa các báo cáo định kỳ từ Google Sheets.
- Tạo các hàm tùy chỉnh (Custom Functions) trong Google Sheets mà hàm mặc định không làm được.
- Xây dựng các Add-ons cho Google Docs/Forms để mở rộng tính năng.
- Tạo các Web App nhỏ gọn để nhập liệu hoặc làm dashboard nội bộ.
- Kết nối Google Sheets với các API bên ngoài (ví dụ: lấy giá coin, tỷ giá hối đoái).


## 6. When NOT to use
- Các ứng dụng yêu cầu hiệu năng cực cao hoặc xử lý dữ liệu khổng lồ (vượt quá giới hạn quota của Google).
- Các tác vụ chạy quá lâu (giới hạn 6 phút cho tài khoản cá nhân, 30 phút cho Workspace).
- Các ứng dụng cần bảo mật mã nguồn tuyệt đối (vì người có quyền chỉnh sửa file thường có thể xem được script).


## 7. Trade-offs
| Pros | Cons |
|------|------|
| Zero Infrastructure (Serverless). | Bị giới hạn nghiêm ngặt bởi Quotas (Execution time, email limits). |
| Tích hợp cực sâu và dễ dàng với Google Services. | Khó khăn trong việc quản lý phiên bản (Version Control) nếu không dùng công cụ bên ngoài như clasp. |
| Hoàn toàn miễn phí (trong hạn mức). | Debugging đôi khi khó khăn hơn so với môi trường local. |


## 8. Alternatives
| Option | So sánh |
|--------|---------|
| Microsoft VBA | Dành cho Excel/Word offline. Không mạnh về cloud/web integration. |
| Python (with Google API) | Mạnh hơn, không bị giới hạn thời gian chạy nhưng cần tự quản lý server và OAuth. |
| Zapier / Make (Integromat) | No-code/Low-code, dễ dùng hơn nhưng tốn phí và ít linh hoạt hơn code thuần. |


## 9. How
Ví dụ: Tự động gửi email khi một giá trị trong Sheet vượt ngưỡng.

```javascript
function checkThresholdAndSendEmail() {
  const sheet = SpreadsheetApp.getActiveSpreadsheet().getActiveSheet();
  const value = sheet.getRange("A1").getValue();
  const threshold = 100;
  
  if (value > threshold) {
    GmailApp.sendEmail(
      "your-email@example.com", 
      "Cảnh báo vượt ngưỡng", 
      `Giá trị hiện tại là ${value}, đã vượt ngưỡng ${threshold}!`
    );
  }
}
```

Tối ưu hóa hiệu năng bằng batch operations:
```javascript
// SAI: Gọi getValue trong vòng lặp (Rất chậm)
for (let i = 1; i <= 1000; i++) {
  let val = sheet.getRange(i, 1).getValue();
}

// ĐÚNG: Lấy tất cả dữ liệu một lần (Nhanh)
const data = sheet.getRange(1, 1, 1000, 1).getValues();
data.forEach(row => {
  let val = row[0];
});
```


## 10. Production concerns
### Scaling
GAS không dành cho việc scale lên hàng triệu user. Nó phù hợp cho nhu cầu nội bộ hoặc doanh nghiệp vừa và nhỏ.

### Failure
Sử dụng `try...catch` và `LockService` để xử lý tranh chấp dữ liệu khi nhiều người dùng hoặc nhiều script chạy cùng lúc.

### Monitoring
Theo dõi tại [script.google.com/home/executions](https://script.google.com/home/executions) để xem lịch sử chạy và các lỗi phát sinh.


## 11. Common mistakes
- Mistake: Gọi `SpreadsheetApp.flush()` hoặc các hàm `getValue/setValue` bên trong vòng lặp.
  Fix: Luôn đọc/ghi dữ liệu theo mảng lớn (Batching) bằng `getValues()` và `setValues()`.

- Mistake: Lưu trữ API Key trực tiếp trong code (Hardcoding).
  Fix: Sử dụng `PropertiesService.getScriptProperties()` để lưu trữ các thông tin nhạy cảm.


## 12. Sample project
Tạo một Web App đơn giản bằng GAS: Khi người dùng truy cập URL, script sẽ đọc dữ liệu từ một Sheet và trả về dưới dạng JSON API để một ứng dụng React/Vue có thể tiêu thụ.


## 13. Interview
### Core Q&A
1. Q: Sự khác biệt giữa Container-bound script và Standalone script là gì?
   A: Container-bound gắn liền với một file (Sheet/Doc), có thể truy cập UI (Menu, Dialog). Standalone là file độc lập trên Drive, thường dùng làm Web App hoặc API.

2. Q: Làm thế nào để vượt qua giới hạn 6 phút thực thi?
   A: Chia nhỏ tác vụ, lưu trạng thái hiện tại vào `PropertiesService` và thiết lập Time-driven trigger để script tự chạy tiếp phần còn lại sau mỗi vài phút.

### Scenario
"Script của bạn chạy rất chậm khi xử lý 5000 dòng dữ liệu, bạn sẽ tối ưu như thế nào?"
-> Chuyển từ việc gọi API từng dòng sang dùng `getValues()` để lấy toàn bộ mảng dữ liệu vào bộ nhớ, xử lý bằng JS thuần, sau đó dùng `setValues()` để ghi kết quả một lần duy nhất.


## 14. References
- Official Docs: [https://developers.google.com/apps-script](https://developers.google.com/apps-script)
- Clasp (Command Line Apps Script Projects): [https://github.com/google/clasp](https://github.com/google/clasp)
- Quotas and Limits: [https://developers.google.com/apps-script/guides/services/quotas](https://developers.google.com/apps-script/guides/services/quotas)


## 15. Real-world Code
Nhiều doanh nghiệp sử dụng GAS để tự động hóa quy trình phê duyệt nghỉ phép, báo cáo doanh thu tự động từ SQL sang Sheets qua JDBC.


## 16. Community
- Stack Overflow: Tag [google-apps-script]
- Google Apps Script Community trên Google Groups.
- Các blog nổi tiếng như Ben Collins (Sheets/GAS expert).
