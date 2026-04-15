# Quy chuẩn và Template Ghi chú Obsidian (dev-brain)

Tài liệu này định nghĩa cấu trúc 13 phần và các quy tắc viết note để đảm bảo kiến thức được hệ thống hóa, dễ tra cứu và hỗ trợ tốt cho việc ôn luyện phỏng vấn.

## 1. Nguyên tắc chung
- **Ngôn ngữ**: Tiếng Việt cho phần giải thích; Tiếng Anh cho thuật ngữ kỹ thuật và code.
- **Tính thực tế**: Ưu tiên ví dụ từ các dự án thực tế (Spring Boot, React, MySQL, AWS).
- **Tính sâu sắc**: Không chỉ ghi chép "nó là gì", mà phải tập trung vào "tại sao dùng", "đánh đổi là gì" và "khi nào không dùng".

## 2. Cấu trúc Metadata (Frontmatter)
Mọi note phải bắt đầu bằng:
```yaml
---
created: yyyy-MM-dd
tags:
  - "#type/concept"      # concept, bug-fix, interview, tutorial
  - "#status/draft"      # draft, review, done
  - "#lang/java"         # java, react, devops, database, system-design
related: "[[Note liên quan]]"
---
```

## 3. Chi tiết 13 Sections

| STT | Section | Yêu cầu nội dung |
|:---:|:---|:---|
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
| 11 | **Common mistakes** | Cặp ❌ **Mistake** / ✅ **Fix** (tối thiểu 2 cặp). |
| 12 | **Sample project** | Bài tập thực hành với **Constraint** khó để hiểu sâu. |
| 13 | **Interview** | Core Q&A, Follow-up và Scenario (tình huống thực tế). |

---

## 4. Blank Template (Copy & Paste)

```markdown
---
created: {{date}}
tags:
  - "#type/concept"
  - "#status/draft"
  - "#lang/java"
related: ""
---
## 1. What
...

## 2. Why (problem it solves)
...

## 3. Mental Model
> "Ví dụ: ..."

## 4. Where it fits (architecture)
`Client → ... → [Component] → ...`

## 5. When to use
- ...

## 6. When NOT to use
- ...

## 7. Trade-offs
| Pros | Cons |
|------|------|
| ...  | ...  |

## 8. Alternatives (with comparison)
| Option | Khi nào chọn |
|--------|-------------|
| [[...]] | ...         |

## 9. How (minimal example)
// Code ở đây

## 10. Production concerns
### Scaling
- ...
### Failure
- ...
### Monitoring
- ...

## 11. Common mistakes / anti-patterns
- ❌ **Mistake**: ...
  ✅ **Fix**: ...

## 12. Sample project (with constraint)
**Tên project**: ...
**Constraint**: ...
**Output**: ...

## 13. Interview
### Core Q&A
1. Q: ...
   A: ...
### Scenario
> "Tình huống: ..."
A: ...
```
