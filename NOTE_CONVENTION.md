# Quy chuẩn và Template Ghi chú Obsidian (dev-brain)

Tài liệu này định nghĩa cấu trúc 16 phần và các quy tắc viết note để đảm bảo kiến thức được hệ thống hóa, dễ tra cứu và hỗ trợ tốt cho việc ôn luyện phỏng vấn.

## 1. Nguyên tắc chung
- **Ngôn ngữ**: Tiếng Việt có dấu cho phần giải thích (không dùng tiếng Việt không dấu); Tiếng Anh cho thuật ngữ kỹ thuật và code.
- **Tính thực tế**: Ưu tiên ví dụ từ các dự án thực tế (Spring Boot, React, MySQL, AWS).
- **Tính sâu sắc**: Không chỉ ghi chép "nó là gì", mà phải tập trung vào "tại sao dùng", "đánh đổi là gì" và "khi nào không dùng".
- **Không dùng emoji**: Tuyệt đối không dùng emoji trong bất kỳ phần nào của note. Dùng text thuần túy.

## 2. Cấu trúc Metadata (Frontmatter)

Mọi note phải bắt đầu bằng:

```yaml
---
created: yyyy-MM-dd
tags:
  - "#type/concept"       # concept | library | pattern | tutorial
  - "#status/draft"       # draft | review | done
  - "#lang/java"          # java | spring | javascript | nodejs | react | devops | database | system-design
  - "#topic/async"        # async | http | i18n | state-management | performance | error-handling
related:
  - "[[Note liên quan]]"
---
```

### Hướng dẫn chọn tag

| Nhóm | Giá trị | Khi nào dùng |
|:-----|:--------|:------------|
| `#type` | `concept` | Khái niệm nền tảng (Event Loop, Optional) |
| | `library` | Thư viện/tool cụ thể (dayjs, axios, tanstack-react-query) |
| | `pattern` | Pattern kiến trúc/thiết kế (query-key-factory, framework-agnostic) |
| | `tutorial` | Hướng dẫn từng bước (Researching an Existing Module) |
| `#status` | `draft` | Mới tạo, chưa hoàn thiện |
| | `review` | Đã viết xong, cần đọc lại |
| | `done` | Hoàn thiện, sẵn sàng ôn luyện |
| `#lang` | `java` | Java Core |
| | `spring` | Spring Boot / Spring Framework |
| | `javascript` | JavaScript thuần |
| | `nodejs` | Node.js runtime |
| | `react` | React và ecosystem |
| | `devops` | DevOps, AWS, CI/CD |
| | `database` | Database, SQL, ORM |
| | `system-design` | System Design |
| `#topic` | `async` | Lập trình bất đồng bộ |
| | `http` | HTTP clients/servers |
| | `i18n` | Internationalization |
| | `state-management` | Quản lý state |
| | `performance` | Hiệu năng |
| | `error-handling` | Xử lý lỗi |

---

## 3. Chi tiết 16 Sections

| STT | Section | Yêu cầu nội dung |
|:---:|:--------|:-----------------|
| 1 | **What** | 2-3 câu định nghĩa súc tích. |
| 2 | **Why** | Vấn đề tồn tại trước khi có nó. Tại sao nó ra đời. |
| 3 | **Mental Model** | **Quan trọng nhất.** Dùng phép ẩn dụ (analogy) để dễ nhớ. |
| 4 | **Where it fits** | Vị trí trong hệ thống. Dùng sơ đồ text `A -> B -> C`. |
| 5 | **When to use** | Các điều kiện/ngữ cảnh cụ thể nên áp dụng. |
| 6 | **When NOT to use** | Hệ quả nếu dùng sai. Trường hợp không phù hợp. |
| 7 | **Trade-offs** | Bảng so sánh Pros/Cons. |
| 8 | **Alternatives** | So sánh với các giải pháp khác. |
| 9 | **How** | Code tối thiểu chạy được (Minimal working example). |
| 10 | **Production concerns** | Scaling, Failure modes (nếu down thì sao?), Monitoring. |
| 11 | **Common mistakes** | Cặp Mistake / Fix (tối thiểu 2 cặp). Dùng text marker, không dùng emoji. |
| 12 | **Sample project** | Bài tập thực hành với Constraint khó để hiểu sâu. |
| 13 | **Interview** | Core Q&A và Scenario (tình huống thực tế). Mỗi phần phải có từ 3-8 câu. |
| 14 | **References** | Official docs, GitHub repo, Spec/RFC, Changelog. |
| 15 | **Real-world Code** | GitHub repos production-grade để học pattern thực tế. |
| 16 | **Community** | Reddit, Stack Overflow, Blog, Conference talks đáng đọc. |

---

## 4. Blank Template

````markdown
---
created: yyyy-MM-dd
tags:
  - "#type/concept"
  - "#status/draft"
  - "#lang/"
  - "#topic/"
related:
  - "[[]]"
---

## 1. What


## 2. Why


## 3. Mental Model


## 4. Where it fits


## 5. When to use


## 6. When NOT to use


## 7. Trade-offs
| Pros | Cons |
|------|------|
| | |

## 8. Alternatives
| Option | So sánh |
|--------|---------|
| | |

## 9. How
```language

```

## 10. Production concerns
### Scaling

### Failure

### Monitoring

## 11. Common mistakes
- Mistake:
  Fix:

- Mistake:
  Fix:

## 12. Sample project


## 13. Interview
### Core Q&A
1. Q:
   A:

### Scenario


## 14. References
- Official Docs:
- GitHub Repo:
- Spec / RFC:
- Changelog:

## 15. Real-world Code


## 16. Community
- Reddit:
- Stack Overflow:
- Blog:
- Talk:
````

---

## 5. Quy tắc tổ chức thư mục

**Quy tắc bắt buộc**: Mọi note kiến thức, tài nguyên và dự án PHẢI nằm trong thư mục `Second Brain/`. Tuyệt đối không để note ở thư mục gốc của project.

Tất cả các note kiến thức phải được đặt vào đúng thư mục con trong `Second Brain/01-Knowledge/`. Nếu thư mục chưa tồn tại, phải tạo mới.

| Tech Stack | Thư mục đích |
|:-----------|:-------------|
| Java Core | `Second Brain/01-Knowledge/java-core/` |
| Spring Boot | `Second Brain/01-Knowledge/java-spring/` |
| JavaScript | `Second Brain/01-Knowledge/javascript/` |
| Node.js | `Second Brain/01-Knowledge/nodejs/` |
| React | `Second Brain/01-Knowledge/react/` |
| DevOps / AWS | `Second Brain/01-Knowledge/devops/` |
| System Design | `Second Brain/01-Knowledge/system-design/` |
| Database | `Second Brain/01-Knowledge/database/` |
| Khác | `Second Brain/01-Knowledge/others/` |
