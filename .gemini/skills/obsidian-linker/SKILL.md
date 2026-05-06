---
name: obsidian-linker
description: Automates updating the 'related' section in Obsidian notes using metadata mapping (titles and tags) for efficiency. Use when creating or updating notes to ensure bidirectional connectivity, fixing dead links, and maintaining technical hierarchy without exhaustive file reading.
---

# Obsidian Linker

## Overview

Skill này tự động hóa việc quản lý mục `related` trong frontmatter của các ghi chú Obsidian. Nó giúp xây dựng một mạng lưới kiến thức (Knowledge Graph) dày đặc và có tính hệ thống bằng cách sử dụng các cơ chế "đoán" thông minh dựa trên Metadata (Tên file và Tags) thay vì đọc toàn bộ nội dung file, giúp tiết kiệm context window.

## Metadata Mapping Logic

Để tối ưu hiệu suất, hãy áp dụng các quy tắc sau để tìm ghi chú liên quan mà không cần đọc sâu nội dung:

### 1. Tag Matrix (So khớp Topic & Lang)
- **Cùng Topic**: Ưu tiên liên kết các note có cùng `#topic/xyz`.
- **Topic Dependencies**: 
  - `#topic/ci-cd` -> `#topic/devops`, `#topic/github`.
  - `#topic/security` -> `#topic/http`, `#topic/networking`.
  - `#topic/performance` -> `#topic/optimization`, `#topic/monitoring`.
- **Language/Framework Mapping**:
  - `#lang/java` -> `#topic/spring`.
  - `#lang/javascript` -> `#lang/nodejs`, `#lang/react`.

### 2. Naming Pattern Matching (Hệ phả tên gọi)
- **Framework Inclusion**: Nếu tên file chứa cùng một Framework (vd: "Spring Boot"), chúng nên được liên kết với nhau.
- **Parent-Child Relation**: Note có tên ngắn (Concept gốc, vd: `Nginx.md`) là cha của note có tên dài hơn chứa nó (vd: `Nginx in Ubuntu.md`).
- **Sibling Relation**: Các note có cùng tiền tố trước chữ `in` (vd: `RestTemplate in Spring Boot` và `WebClient in Spring Boot`).

## Workflow Decision Tree

### Bước 1: Thu thập Metadata
Chỉ sử dụng các lệnh như `ls`, `grep` hoặc `glob` để lấy danh sách tên file và tags từ frontmatter. Tránh dùng `read_file` trừ khi thực sự cần kiểm tra phần `1. What`.

### Bước 2: Phân tích & Gợi ý
Dựa trên Metadata Mapping Logic, hãy đề xuất 3-5 ghi chú liên quan nhất. 
- *Luôn ưu tiên các ghi chú có quan hệ phân cấp (Cha/Con) hoặc cùng Framework.*

### Bước 3: Cập nhật Bidirectional (Hai chiều)
Khi thêm `[[Note B]]` vào mục `related` của `Note A`, hãy thực hiện:
1. Đọc mục `related` của `Note A`, thêm `[[Note B]]` nếu chưa có.
2. Đọc mục `related` của `Note B`, thêm `[[Note A]]` (tính đối xứng).

### Bước 4: Sửa lỗi Link & Định dạng
- **Dead links**: Nếu một link trong `related` không tồn tại file tương ứng, hãy đề xuất xóa hoặc cập nhật.
- **Formatting**: Luôn dùng định dạng list YAML:
  ```yaml
  related:
    - "[[Tên Note]]"
  ```

## Quy tắc & Ràng buộc

1. **Số lượng**: Giới hạn tối đa 5 liên kết cho mỗi note.
2. **Loại trừ**: Tuyệt đối không quét hoặc liên kết tới các thư mục: `00-Inbox`, `.obsidian`, `.git`.
3. **Approval**: Luôn hiển thị danh sách gợi ý và hỏi người dùng trước khi ghi vào file (trừ khi được yêu cầu "tự động hoàn toàn").
4. **Emoji**: Không sử dụng emoji trong file Markdown theo quy chuẩn dự án.

## Lệnh hỗ trợ
- "Update related cho note này": Quét và cập nhật cho 1 file.
- "Bulk update related cho thư mục [path]": Quét toàn bộ thư mục để tối ưu liên kết nội bộ.
- "Check dead links": Tìm và sửa các liên kết không tồn tại.
