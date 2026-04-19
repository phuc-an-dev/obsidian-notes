# Obsidian Second Brain (phuc-an-dev)

Chào mừng bạn đến với kho lưu trữ kiến thức cá nhân (Second Brain) của mình. Đây là nơi mình hệ thống hóa các kiến thức về lập trình, đặc biệt là Java Spring Boot, React, và System Design.

## 📂 Cấu trúc thư mục

Hệ thống được tổ chức theo phương pháp PARA:
- **00-Inbox**: Nơi chứa các ghi chú nháp, ý tưởng vừa nảy sinh chưa được phân loại.
- **01-Knowledge**: Kho lưu trữ kiến thức cốt lõi (Core Concepts).
    - `java-spring/`: Các ghi chú chuyên sâu về Spring Framework, Spring Boot.
- **02-Projects**: Theo dõi tiến độ và ghi chú cho các dự án thực tế đang triển khai.
- **03-Resources**: Các nguồn tài liệu tham khảo, cheatsheets, sách và bài báo hay.

## 📝 Quy chuẩn ghi chú (Note Convention)

Mọi ghi chú trong hệ thống này đều tuân thủ theo quy chuẩn 16 phần định nghĩa tại [NOTE_CONVENTION.md](./NOTE_CONVENTION.md) để đảm bảo tính thực tế, sâu sắc và hỗ trợ tốt cho việc ôn luyện phỏng vấn.

### Metadata mẫu:
```yaml
---
created: yyyy-MM-dd
tags:
  - "#type/concept"
  - "#status/done"
  - "#lang/java"
related: "[[Note liên quan]]"
---
```

## 🚀 Các nội dung mới cập nhật

Gần đây mình đã hoàn thiện bộ note về giao tiếp HTTP và cơ chế cốt lõi của JavaScript:
- `EventListener`: Cơ chế loose coupling thông qua sự kiện.
- `Spring HTTP Core`: HttpHeaders, HttpStatus, ResponseEntity,...
- `HTTP Clients`: Tổng quan về các bộ gọi API.
- `RestTemplate`: Client đồng bộ truyền thống.
- `WebClient`: Client Reactive/Non-blocking hiện đại.
- `RestClient`: Client đồng bộ mới từ Spring 6.1.
- `Feign Client`: Declarative HTTP Client cho Microservices.
- `Event Publisher`: Cơ chế phát sự kiện trong Spring.
- `DeepL API`: Tích hợp dịch thuật AI.
- `Event Loop`: Cơ chế xử lý bất đồng bộ trong JavaScript/Node.js.
- `Non-blocking I/O`: Mô hình xử lý Input/Output hiệu năng cao.

---
*Ghi chú này được duy trì và cập nhật bởi Gemini CLI.*
